---
title: Mesh bending along a spline in Unity (Part 1)
date: 2025-03-09+0700
categories: [Unity]
tags: [job system]
image: /assets/img/2025-03-09-spline-mesh-optimization-part-1/poster.jpg
redirect_from:
  - /blog/2025/03/09/spline-mesh-optimization-part-1.html
---

This article was written in 2022, but only now have I had the opportunity to publish it in English. As I read through it now, I feel like rewriting most of it, but I will leave it as is. Moreover, after this article, the code was used in another project, which led to further optimizations. Therefore, it makes more sense to simply add a second part to this article.

#### Defining the Problem

In one of our prototypes, we used a visually appealing [library](https://github.com/methusalah/SplineMesh) to create 'rubber' hands for a character. Everything went smoothly during the prototyping stage, but later in development, we ran into performance issues on mobile devices. Because of this, we had to switch to a lower-poly mesh, which affected the visual quality. So, I decided to take a look under the hood and make some quick optimizations.

#### Getting Ready

To save time, we won’t profile the entire game. Instead, it's better to create a separate project where we can isolate and analyze specific parts of the code. To speed up performance measurements, we’ll use the [Unity Performance Testing package](https://docs.unity3d.com/Packages/com.unity.test-framework.performance@2.8/manual/index.html).

Next, we take a prefab from the library’s sample set that best matches our use case.  

![image-2.gif](/assets/img/2025-03-09-spline-mesh-optimization-part-1/image-2.gif)

The original prefab has three 'roots,' each with 790 vertices. We'll add a second version of the prefab, where each root has 45,835 vertices. Then, we'll run two tests: one with 50 low-poly objects and another with a single high-poly object.

We create a scene with a simple spawner. Then, we write a test:

```c#
public class PerformanceTest {
  [UnityTest, Performance]
  public IEnumerator Test() {
    SceneManager.LoadScene("Performance Test Scene", LoadSceneMode.Additive);
    yield return Measure.Frames()
      .WarmupCount(30)
      .MeasurementCount(30)
      .Run();
  }
}
```

Loading a scene in this test isn’t exactly the best approach. It would be better to create objects directly within the test itself, allowing us to run parametric tests with different object counts, and so on. However, in a simple scenario, this would require placing our prefabs in the Resources folder, which isn’t ideal either—because then our test prefab would end up in the final build of projects using our library. More advanced solutions like Addressables introduce extra dependencies. So, we’ll try to keep things simple for the end users of our library.

Additionally, we'll use our test scene for both testing and profiling. Now, let's take the initial measurements (from here on, frame time is given in milliseconds):

| Test Name                 | Min      | Max      | Median   | Average          |
| ------------------------- | -------- | -------- | -------- | ---------------- |
| IL2CPP - 1 Big mesh       | 65.9858  | 67.6671  | 66.5221  | 66.51235         |
| IL2CPP - 50 simple meshes | 52.1913  | 60.2441  | 58.4168  | 58.0337566666667 |
| Mono - 1 Big mesh         | 113.4949 | 119.7267 | 117.1779 | 116.80328        |
| Mono - 50 simple meshes   | 107.682  | 117.7221 | 110.545  | 111.915016666667 |

Now, let's take a look at the profiler and start identifying issues.

#### Reducing Allocations

![image-1649742642579.png](/assets/img/2025-03-09-spline-mesh-optimization-part-1/image-1649742642579.png)

The first thing that stands out is the large number of allocations. Here, we need to take a closer look at how the library works. First, it calculates a fixed number of samples along the spline. Then, the mesh vertices are updated based on these samples. Since the mesh has a fixed number of vertices, all arrays should also have fixed sizes, meaning there shouldn’t be a need for constant memory allocation.

Let’s start with this. It probably won’t have a huge impact on FPS, but it’s always better to avoid putting extra strain on the GC and wasting time on unnecessary object creation.

When recalculating vertex positions, normals, etc., the author stores the results in a list as objects and then uses LINQ to extract the required data arrays. Additionally, each time a new mesh is recalculated, the author repeatedly accesses properties of the original mesh. It looks something like this:

```c#
bentVertices.Add(sample.GetBent(vert));
...
MeshUtility.Update(result,
	source.Mesh,
	source.Triangles,
	bentVertices.Select(b => b.position),
	bentVertices.Select(b => b.normal));
...
mesh.vertices = vertices == null ? source.vertices : vertices.ToArray();
mesh.normals = normals == null ? source.normals : normals.ToArray();
```

This can be avoided. We only need to cache the original mesh data when it changes—there’s no need to access it every time. Instead of storing calculated values in a single list, we can use separate arrays for each field. Since we already know the required size, there’s no need to create new arrays every frame. This way, we eliminate the use of LINQ and the overhead of calling `ToArray()`.

[Commit link](https://github.com/utkaka/SplineMesh/commit/e1a12c418871b874d5b84c10cd1685d2516ab8c8)

![Profile.png](/assets/img/2025-03-09-spline-mesh-optimization-part-1/profile.png)

Getting better, but still not perfect. Let’s take a look at what’s happening inside `CurveSample.GetBent()`:

```c#
public MeshVertex GetBent(MeshVertex vert) {
  var res = new MeshVertex(vert.position, vert.normal, vert.uv);

  // application of scale
  res.position = Vector3.Scale(res.position, new Vector3(0, scale.y, scale.x));

  // application of roll
  res.position = Quaternion.AngleAxis(roll, Vector3.right) * res.position;
  res.normal = Quaternion.AngleAxis(roll, Vector3.right) * res.normal;

  // reset X value
  res.position.x = 0;

  // application of the rotation + location
  Quaternion q = Rotation * Quaternion.Euler(0, -90, 0);
  res.position = q * res.position + location;
  res.normal = q * res.normal;
  return res;
}
```

It seems like it’s just calculations, so where are all these allocations coming from? The issue lies in the return type of the object.

```c#
public class MeshVertex {
  public Vector3 position;
  public Vector3 normal;
  public Vector2 uv;

  public MeshVertex(Vector3 position, Vector3 normal, Vector2 uv) {
    this.position = position;
    this.normal = normal;
    this.uv = uv;
  }

  public MeshVertex(Vector3 position, Vector3 normal): this(position, normal, Vector2.zero){}
}
```

For a simple object made up of value-type fields and created in large quantities, the author chose a class. Let’s try changing it to a struct instead.

![image-1649745865152.png](/assets/img/2025-03-09-spline-mesh-optimization-part-1/image-1649745865152.png)

The topic of "class vs struct" is complex and broad. However, it's always best to understand the reasoning behind your choice of one over the other. In this case, switching to a struct helped eliminate allocations. But this conversion will also come in handy later.

The next memory-related concern is object comparison in `CurveSample`. Let’s take a look at the code:

```c#
public struct CurveSample {
	...
	public override bool Equals(object obj) {
      if (obj == null || GetType() != obj.GetType()) {
        return false;
      }
      CurveSample other = (CurveSample)obj;
      return location == other.location &&
        tangent == other.tangent &&
        up == other.up &&
        scale == other.scale &&
        roll == other.roll &&
        distanceInCurve == other.distanceInCurve &&
        timeInCurve == other.timeInCurve;
    }
	...
}
```

This time, the author did use a struct, but only overrode the base comparison method, which inevitably leads to boxing and unboxing. To avoid this, structs need to define a comparison method with an explicit type, like this:

```c#
public bool Equals(CurveSample other) {
  return location == other.location &&
    tangent == other.tangent &&
    up == other.up &&
    scale == other.scale &&
    Math.Abs(roll - other.roll) < float.Epsilon &&
    Math.Abs(distanceInCurve - other.distanceInCurve) < float.Epsilon &&
    Math.Abs(timeInCurve - other.timeInCurve) < float.Epsilon;
}

public override bool Equals(object obj) {
  return obj is CurveSample other && Equals(other);
}
```

![image-1649747297303.png](/assets/img/2025-03-09-spline-mesh-optimization-part-1/image-1649747297303.png)

And the last patient in this section is `UnityEvent`. I’m not quite sure why the author chose them for dispatching internal library events. They can be useful when we want to work with events in Unity’s inspector. However, if we’re subscribing to events and dispatching them from code, I don’t see the point of using anything other than traditional C# delegates.

```c#
//before
public UnityEvent Changed = new UnityEvent();
//after
public event Action Changed;
```

It works exactly the same, but in the profiler:

![image-1649747676302.png](/assets/img/2025-03-09-spline-mesh-optimization-part-1/image-1649747676302.png)

Now, let's take some performance measurements:

| Test Name                               | Min      | Max      | Median   | Average          |
| --------------------------------------- | -------- | -------- | -------- | ---------------- |
| testResults - IL2CPP - 1 Big mesh       | 49.8567  | 58.6284  | 50.1922  | 53.0550533333333 |
| testResults - IL2CPP - 50 simple meshes | 41.9036  | 51.0035  | 49.9033  | 48.0567666666667 |
| testResults - Mono - 1 Big mesh         | 102.1293 | 110.7378 | 108.0379 | 107.18864        |
| testResults - Mono - 50 simple meshes   | 104.0972 | 112.3465 | 108.4403 | 108.59851        |

We didn’t expect much, but we did get something.

#### Optimizing Calculations

Now, let's experiment with the Unity Job System using the Burst Compiler. We'll start with `CurveSample.GetBent()` since, according to the latest profiler data, it’s the most time-consuming call.

At this point, it’s worth diving a bit deeper into what’s happening. After any changes to our curve, the `MeshBender` component loops through all the mesh vertices, calculates their position along the curve, finds the closest sample to that point, and then applies the sample’s distortion to the vertex. There’s also a small caching mechanism using a `Dictionary` to store computation results for nearby vertices.

```c#
for (var i = 0; i < _sourceVertices.Count; i++) {
  var vert = _sourceVertices[i];
  var distanceRate = source.Length == 0 ? 0 : Math.Abs(vert.position.x - source.MinX) / source.Length;
  if (!sampleCache.TryGetValue(distanceRate, out var sample)) {
    if (!useSpline) {
    	sample = curve.GetSampleAtDistance(curve.Length * distanceRate);
    } else {
    	var intervalLength =
    	intervalEnd == 0 ? spline.Length - intervalStart : intervalEnd - intervalStart;
    	var distOnSpline = intervalStart + intervalLength * distanceRate;
    	if (distOnSpline > spline.Length) {
    		distOnSpline = spline.Length;
  		}
  		sample = spline.GetSampleAtDistance(distOnSpline);
  	}

  	sampleCache[distanceRate] = sample;
  }

  var bent = sample.GetBent(vert);
  _vertices[i] = bent.position;
  _normals[i] = bent.normal;
}
```

Now, instead of calling `sample.GetBent(vert)` in the loop, we’ll prepare the data for our Job, where the calculations will actually take place:

```c#
[BurstCompile]
public struct CurveSampleBentJob : IJobParallelFor {
	[ReadOnly]
	public NativeArray<CurveSample> Curves;
    [ReadOnly]
    public NativeArray<MeshVertex> VerticesIn;
    [WriteOnly]
    public NativeArray<float3> VerticesOut;
    [WriteOnly]
    public NativeArray<float3> NormalsOut;

	public void Execute(int i) {
    	var curve = Curves[i];
        var vertexIn = VerticesIn[i];
		var bent = new MeshVertex(vertexIn.position, vertexIn.normal, vertexIn.uv);
        // application of scale
        bent.position = new float3(0.0f, bent.position.y * curve.scale.y, bent.position.z * curve.scale.x);
        // application of roll
        bent.position = math.mul(quaternion.AxisAngle(new float3(1.0f, 0.0f ,0.0f), math.radians(curve.roll)), bent.position);
        bent.normal = math.mul(quaternion.AxisAngle(new float3(1.0f, 0.0f ,0.0f), math.radians(curve.roll)), bent.normal);
        bent.position.x = 0;
        // application of the rotation + location
        var q =  math.mul(curve.Rotation, quaternion.Euler(0.0f, math.radians(-90.0f), 0.0f));
        bent.position = math.mul(q, bent.position) + curve.location;
        bent.normal = math.mul(q, bent.normal);
        VerticesOut[i] = bent.position;
        NormalsOut[i] = bent.normal;    
	}
}
```

A bit of explanation. The `[BurstCompile]` attribute is used to ensure our Job gets compiled with Burst. The `IJobParallelFor` interface indicates that the job should run in parallel over an array of data, with the element index for computation being passed to the `Execute(int i)` method. Additionally, all calculations have been rewritten using the `Unity.Mathematics` package, which allows the Burst compiler to leverage SIMD instructions to optimize the calculations when possible.

When transitioning to `Unity.Mathematics`, it’s important to be aware of some differences between it and standard math. For example, the standard `Quaternion.AngleAxis` accepts the angle in degrees, whereas `Unity.Mathematics.quaternion.AxisAngle` expects it in radians. I didn’t notice this mistake at first and spent some time wondering why the results weren’t matching up.

I’d also like to point out that it’s useful here that we made `MeshVertex` a struct, otherwise, we wouldn’t have been able to pass it to the Job.

Now, let’s rewrite the code in `MeshBender` responsible for starting the calculations.

```c#
var jobVerticesIn = new NativeArray<MeshVertex>(_sourceVertices.Length, Allocator.TempJob, NativeArrayOptions.UninitializedMemory);
var jobVerticesOut = new NativeArray<Vector3>(_sourceVertices.Length, Allocator.TempJob, NativeArrayOptions.UninitializedMemory);
var jobNormalsOut = new NativeArray<Vector3>(_sourceVertices.Length, Allocator.TempJob, NativeArrayOptions.UninitializedMemory);
var jobCurveSamples = new NativeArray<CurveSample>(_sourceVertices.Length, Allocator.TempJob, NativeArrayOptions.UninitializedMemory);

for (var i = 0; i < _sourceVertices.Length; i++) {
  var vert = _sourceVertices[i];
  var distanceRate = source.Length == 0 ? 0 : Math.Abs(vert.position.x - source.MinX) / source.Length;
  if (!sampleCache.TryGetValue(distanceRate, out var sample)) {
    if (!useSpline) {
      sample = curve.GetSampleAtDistance(curve.Length * distanceRate);
    } else {
      var intervalLength =
        intervalEnd == 0 ? spline.Length - intervalStart : intervalEnd - intervalStart;
      var distOnSpline = intervalStart + intervalLength * distanceRate;
      if (distOnSpline > spline.Length) {
        distOnSpline = spline.Length;
      }

      sample = spline.GetSampleAtDistance(distOnSpline);
    }
    sampleCache[distanceRate] = sample;
  }

  _curveSamples[i] = sample;
}

jobVerticesIn.CopyFrom(_sourceVertices);
jobCurveSamples.CopyFrom(_curveSamples);

var job = new CurveSampleBentJob {
  Curves = jobCurveSamples,
  VerticesIn = jobVerticesIn,
  VerticesOut = jobVerticesOut,
  NormalsOut = jobNormalsOut
  };
job.ScheduleParallel(_sourceVertices.Length, 4, default).Complete();

jobVerticesOut.CopyTo(_vertices);
jobNormalsOut.CopyTo(_normals);

jobCurveSamples.Dispose();
jobVerticesIn.Dispose();
jobVerticesOut.Dispose();
jobNormalsOut.Dispose();
```

The complexity has definitely increased. Now, we need to prepare the input data for our calculations as a `NativeArray`, and then convert it back to managed arrays afterward.

| Test Name                               | Min     | Max     | Median  | Average          |
| --------------------------------------- | ------- | ------- | ------- | ---------------- |
| testResults - IL2CPP - 1 Big mesh       | 40.5467 | 42.6622 | 41.692  | 41.6773633333333 |
| testResults - IL2CPP - 50 simple meshes | 40.5681 | 42.8388 | 41.6424 | 41.6936766666667 |
| testResults - Mono - 1 Big mesh         | 49.0124 | 51.0935 | 49.9636 | 49.9608833333333 |
| testResults - Mono - 50 simple meshes   | 66.8223 | 75.5494 | 74.3093 | 72.56501         |

Let’s take another look at the profiler:

![image-1649752639038.png](/assets/img/2025-03-09-spline-mesh-optimization-part-1/image-1649752639038.png)

Now, let's focus on two things.

First: `MeshBender.ComputeIfNeeded()` is now running much faster, which is exactly what we were aiming for.

Second: `CubicBezierCurve.ComputeSample()` has become the 'leader' in execution time, and it seems there’s something off with it. A quick breakdown: On the scene, we have 50 objects, each with 3 'roots'. Two of the roots have 4 `CubicBezierCurve` instances, and the third one has 5. That means we should have 50 * (2 * 4 + 5) = 650 curves in total. But there are 1300 calls. This means each curve is calculating samples twice. The reason is that each curve is defined by two points. Whenever either point changes, the curve catches the change event and performs recalculations. If both points of the curve change within the same frame, the curve recalculates the samples unnecessarily. This highlights the need for careful handling of events to avoid what we could call 'event hell'.

My solution can’t be called elegant; I didn’t rewrite the entire library logic because that would have taken a bit more time. Instead, I mark the curves that need recalculation as 'dirty' and trigger the calculations through a coroutine. There's not much to brag about, but you can check out the changes in [this commit](https://github.com/utkaka/SplineMesh/commit/e04e2b507f4fec9027607660f2b28169ff6bdad9).

However, we've managed to reduce the number of calls to the expected amount.

![image-1649753684121.png](/assets/img/2025-03-09-spline-mesh-optimization-part-1/image-1649753684121.png)

Don’t pay too much attention to the increased timings in the profiler for now, because by this point, I’ve already rewritten all the calculations using `Unity.Mathematics`, which is primarily designed to be used with the Burst Compiler. However, in the test measurements in the build, this didn’t have any noticeable impact.

| Test Name                               | Min     | Max     | Median  | Average          |
| --------------------------------------- | ------- | ------- | ------- | ---------------- |
| testResults - IL2CPP - 1 Big mesh       | 40.6897 | 42.6201 | 41.6866 | 41.66077         |
| testResults - IL2CPP - 50 simple meshes | 40.7038 | 42.7063 | 41.668  | 41.64089         |
| testResults - Mono - 1 Big mesh         | 49.4439 | 50.3378 | 49.9846 | 49.9465366666667 |
| testResults - Mono - 50 simple meshes   | 57.4371 | 59.4481 | 58.3213 | 58.2975433333333 |

The next suspicious area is `SourceMesh`. This object is used by the author to store information about the original mesh. This time, it’s the other way around. Instead of using a class, the author made it a struct, but with a reference-type field.

```c#
public struct SourceMesh {
  private Vector3 translation;
  private Quaternion rotation;
  private Vector3 scale;

  internal Mesh Mesh { get; }
  ...
}
```

This might have been done to allow such manipulations while avoiding allocations:

```c#
mb.Source = SourceMesh.Build(tm.mesh)
  .Translate(tm.translation)
  .Rotate(Quaternion.Euler(tm.rotation))
  .Scale(tm.scale);
```

This issue can be easily solved by creating a specialized constructor.

```c#
public SourceMesh(Mesh mesh, Vector3 translation, Quaternion rotation, Vector3 scale) {
  _translation = translation;
  _rotation = rotation;
  _scale = scale;
  BuildData(mesh);
}
```

Storing a reference to the original mesh and extracting its parameters each time seems redundant to me. All the arrays for vertices, normals, and other data can be extracted from the mesh during the creation phase, and then we can work directly with them.

Next, I made probably not the most crucial optimization, but I don’t consider it unnecessary. Just to remind you, `MeshBender` loops through all the mesh vertices and for each one, it grabs the closest curve sample. There was also a caching mechanism for samples of nearby vertices.

```c#
for (var i = 0; i < _sourceVertices.Length; i++) {
  var vert = _sourceVertices[i];
  var distanceRate = source.Length == 0 ? 0 : Math.Abs(vert.position.x - source.MinX) / source.Length;
  if (!sampleCache.TryGetValue(distanceRate, out var sample)) {
    ...
  }
}
```

Here, you can see that the mesh is stretched along the X-axis. Accordingly, the point's position along this axis is normalized, and samples are cached based on this value. But these are the coordinates of the original mesh, so there's no need to recalculate these groups every time the curve changes. We can do this at the creation stage of the `SourceMesh` instead:

```c#
private Dictionary<float, List<int>> _sampleGroups;
...

private void BuildData(Mesh mesh) {
  ...
  for (var i = 0; i <Vertices.Length; i++) {
    var distanceRate = Length == 0 ? 0 : Math.Abs(Vertices[i].position.x - MinX) / Length;
    if (!_sampleGroups.TryGetValue(distanceRate, out var group)) {
      group = new List<int>();
      _sampleGroups[distanceRate] = group;
    }
    group.Add(i);
  }
}
```

In `MeshBender`, we will no longer use caching via a dictionary. Well, for now, we will, but through a different dictionary, populated in another place. Later, we’ll replace it with two arrays. Now, we’ll go through the groups, take a sample for each one, and only then assign the sample to the vertices of that group.

```c#
foreach (var distanceRate in source.SampleGroups.Keys) {
  CurveSample sample;
  if (!useSpline) {
    sample = curve.GetSampleAtDistance(curve.Length * distanceRate);
  } else {
    var intervalLength =
      intervalEnd == 0 ? spline.Length - intervalStart : intervalEnd - intervalStart;
    var distOnSpline = intervalStart + intervalLength * distanceRate;
    if (distOnSpline > spline.Length) {
      distOnSpline = spline.Length;
    }
    sample = spline.GetSampleAtDistance(distOnSpline);
  }

  var sampleGroup = source.SampleGroups[distanceRate];

  for (var i = 0; i < sampleGroup.Count; i++) {
    _curveSamples[sampleGroup[i]] = sample;
  }
}
```

Now, we can think about what other calculations we can move to a Job. A good candidate for this is `CubicBezierCurve.CreateSample()`. We’ll move all the calculations to a separate Job, where I’ve inlined all the calls to save on some extra calculations. So instead of:

```c#
private CurveSample CreateSample(float distance, float time) {
  return new CurveSample(
    GetLocation(time),
    GetTangent(time),
    GetUp(time),
    GetScale(time),
    GetRoll(time),
    distance,
    time);
}
```

We get something like this:

```c#
[BurstCompile]
public struct ComputeSamplesJob : IJobParallelFor {
  public SplineNode Node1;
  public SplineNode Node2;
  [WriteOnly]
  public NativeArray<CurveSample> Samples;
  public void Execute(int i) {
  var time = (float)i / CubicBezierCurve.STEP_COUNT;
  //Location
  var omt = 1f - time;
  var omt2 = omt * omt;
  var t2 = time * time;
  var inverseDirection = 2 * Node2.Position - Node2.Direction;
  var location = Node1.Position * (omt2 * omt) +
                 Node1.Direction * (3f * omt2 * time) +
                 inverseDirection * (3f * omt * t2) +
                 Node2.Position * (t2 * time);
  //Tangent
  var tangent = Node1.Position * -omt2 +
                Node1.Direction * (3 * omt2 - 2 * omt) +
                inverseDirection * (-3 * t2 + 2 * time) +
                Node2.Position * t2;
  tangent = math.normalize(tangent);
  //Up
  var up = math.lerp(Node1.Up, Node2.Up, time);
  //Scale
  var scale = math.lerp(Node1.Scale, Node2.Scale, time);
  //Roll
  var roll = math.lerp(Node1.Roll, Node2.Roll, time);

  Samples[i] = new CurveSample(
      location,
      tangent,
      up,
      scale,
      roll,
      0.0f,
      time);
	}
}
```

The calculation of the curve length can also be moved out:

```c#
[BurstCompile]
public struct ComputeCurveLengthJob : IJob {
  public NativeArray<CurveSample> Samples;
  public NativeArray<float> Length;
  public void Execute() {
    var length = 0.0f;
    for (var i = 0; i <= CubicBezierCurve.STEP_COUNT; i++) {
    var sample = Samples[i];
    if (i > 0) length += math.distance(Samples[i - 1].Location, sample.Location);
    sample.DistanceInCurve = length;
    Samples[i] = sample;
    }

    Length[0] = length;
  }
}
```

And this is how their sequential execution looks now:

```c#
public void ComputeSamples() {
  if (!_isDirty) return;
  samples ??= new CurveSample[STEP_COUNT + 1];

  var jobCurveSamples = new NativeArray<CurveSample>(STEP_COUNT + 1, Allocator.TempJob, NativeArrayOptions.UninitializedMemory);
  var job = new ComputeSamplesJob {
  Node1 = _node1,
  Node2 = _node2,
  Samples = jobCurveSamples,
  };
  var jobHandle = job.Schedule(STEP_COUNT + 1, 4, default);

  var jobLength = new NativeArray<float>(1, Allocator.TempJob, NativeArrayOptions.UninitializedMemory);

  var computeCurveLengthJob = new ComputeCurveLengthJob {
  Samples = jobCurveSamples,
  Length = jobLength
  };

  computeCurveLengthJob.Schedule(jobHandle).Complete();

  _length = jobLength[0];
  jobCurveSamples.CopyTo(samples);
  jobCurveSamples.Dispose();
  jobLength.Dispose();

  _isDirty = false;

  Changed?.Invoke();
}
```

I also moved the calculation of the Lerp between two curve samples, which is actually used in `MeshBender`. I won't include the code here. On one hand, I want to spare the readers from having to dig through the commits on GitHub. On the other hand, there’s already too much code in the article. So, the commit with these changes can be found [here](https://github.com/utkaka/SplineMesh/commit/1a7df017ae8a994c98db36344aca307dc66a39f7).

| Test Name                               | Min     | Max     | Median  | Average          |
| --------------------------------------- | ------- | ------- | ------- | ---------------- |
| testResults - IL2CPP - 1 Big mesh       | 32.3238 | 34.3637 | 33.3293 | 33.3280933333333 |
| testResults - IL2CPP - 50 simple meshes | 40.0466 | 58.7367 | 41.6677 | 43.0790133333333 |
| testResults - Mono - 1 Big mesh         | 32.3963 | 34.3302 | 33.3263 | 33.2917133333333 |
| testResults - Mono - 50 simple meshes   | 40.3834 | 51.5294 | 41.6574 | 42.18962         |

It's not the most important observation, but still worth noting that the Mono performance has practically leveled with IL2CPP.

Further optimization of the library would have required more significant changes, and accordingly, more time. But at this point, the current performance was already quite satisfactory. So, I decided to take one last look at the profiler, just in case something quick could be improved. And something was found.

![image-1649761425950.png](/assets/img/2025-03-09-spline-mesh-optimization-part-1/image-1649761425950.png)

The author of the library called `RecalculateTangents()` every time the mesh was updated, but in our materials, they’re generally not needed. So, we can make this an additional option for those who want it. Of course, we could come up with something using the Job System, but for now, let's just add a flag so that this call will only be made when necessary. Then, we’ll perform the final measurements.

| Test Name                               | Min     | Max     | Median  | Average          |
| --------------------------------------- | ------- | ------- | ------- | ---------------- |
| testResults - IL2CPP - 1 Big mesh       | 7.5389  | 9.3847  | 8.3297  | 8.32364          |
| testResults - IL2CPP - 50 simple meshes | 23.3105 | 33.3585 | 25.0162 | 26.35596         |
| testResults - Mono - 1 Big mesh         | 7.5006  | 16.7793 | 8.3345  | 9.44308333333333 |
| testResults - Mono - 50 simple meshes   | 23.8324 | 50.2136 | 24.9554 | 27.4275966666667 |

#### Conclusions and Final Thoughts

{%
  include embed/video.html
  src='/assets/img/2025-03-09-spline-mesh-optimization-part-1/demo.webm'
  types='mp4'
  poster='poster.jpg'
  title='Demo video'
  autoplay=true
  loop=true
  muted=true
%}

First, let's discuss the significant difference between a large mesh and many small ones. The use of the Job System itself is not a silver bullet and comes with its own overhead, such as the constant transferring of data between managed arrays and NativeArrays, as well as the creation of these NativeArrays. The larger the amount of data we put into a single NativeArray and process with one Job, the more noticeable the performance gain will be. Currently, each curve and each MeshBender on the scene performs calculations independently of the others, and the overhead is cumulative.

In my opinion, the most effective solution to this problem would be the use of ECS, especially since we’re already using other parts of DOTS, and they work best together. This would allow us to write separate systems for recalculating curves and processing mesh vertices, as well as control the order of their execution without needing to rely on `LateUpdate`. However, switching to ECS would require a complete rewrite of the project, so let’s leave that for the future or as a homework assignment for those interested.

Another useful improvement could be the use of the [Advanced Mesh API](https://docs.unity3d.com/ScriptReference/Mesh.MeshData.html), which is specifically designed for processing meshes in the Job System. Currently, we're transferring data back and forth too much, which introduces unnecessary overhead.

At the moment, my [fork](https://github.com/utkaka/SplineMesh) of the original library can't be called complete. During the refactoring process, I completely commented out the editor scripts and haven't fixed them yet. Also, in MeshBender, I only converted the FillStretch method to use the Job System; the other two methods are still functional, but likely with worse performance.
