---
title: "Videogame Linear Algebra"
---

{% assign base = "/assets/videogame-linear-algebra" | relative_url %}

{{ page.title }}
================

As a kid, I always wondered how 3D videogames figured out where to draw things on the screen. Even after taking courses in trigonometry, vector algebra and linear algebra, my initial attempts to use them for projecting things onto the screen were severely over-complicated and slow, relying too much on trig and not enough on linear algebra. It turns out most real-time renderers approach the math in one particular way that relies heavily on linear algebra. This article is intended as sort of a crash course to give you some exposure to the relevant areas of math; it barely touches on each topic, but I'm hoping that using 3D videogames as a concrete motivating example will make it _much_ easier to get your feet wet and develop a solid intuition for how and why matrices work. Also, I'm writing all code samples in Rust; hopefully it'll still be readable even if you're not familiar with the language.

The Big Picture
---------------

First, a very high-level overview/recap of how games typically draw things: A game world can be thought of as a set of models to draw, and a model is a set of triangles. Each triangle has three vertices (corners), and vertices have a position and whatever other attributes may be useful, like color or texture coordinates. To draw a triangle, the game first figures out where the vertices should be drawn on the screen, then fills in the triangle, smoothly blending any attributes (like color) between vertices and passing the blended attributes to a program that determines what to actually write to each fragment (essentially pixel) of the triangle. And, if you store each pixel's distance from the screen, you can draw the triangles in any order without needing to sort them from back to front. Rinse, repeat (a lot), and you have a depiction of a 3D world.

We're interested in the part where, given a 3D position of a vertex, we figure out where that would be on the screen. Additionally, because it's common for games to draw millions of triangles, whatever we come up with needs to be _fast_. However, launching into the full-blown solution would be difficult, so we need to build towards it:

1. [We'll cover the very basics of trigonometry and vector algebra.](#foundational-trigonometry)
2. [We'll devise an efficient way to perform 2D rotations.](#rotating-points)
3. [We'll expand that approach to also handle scaling and translations.](#scaling-and-translation)
4. [We'll learn how to combine and invert spatial transformations.](#bringing-everything-together-in-2d)
5. [We'll add a basic 3D perspective transformation to our toolbelt.](#your-first-3d-perspective)
6. [We'll adjust the perspective's position, orientation and field of view.](#adjusting-3d-perspectives)

Foundational Trigonometry
-------------------------

(if you're familiar with trigonometry, feel free to [skip to the next section](#foundational-vector-algebra))

The most important thing to know about trigonometry is the meaning of [`sin(a)` and `cos(a)`](https://en.wikipedia.org/wiki/Sine_and_cosine) (pronounced sine of `a` and cosine of `a`, respectively). An interactive diagram is worth a thousand words, so...

<figure>
  <div id="figure-sincos" class="jxgbox" style="aspect-ratio: 3/2"></div>
</figure>
<script type="text/javascript">{
  let board = JXG.JSXGraph.initBoard('figure-sincos', {
    boundingbox: [-3, 1.5, 3, -2.5],
    axis: true,
    showNavigation: false,
  });
  board.suspendUpdate();

  let attrs = {
    fixed: true,
    size: 3,
    withLabel: false,
  };

  let unitCircle = board.create('circle', [
    [0, 0], 1
  ], {
    ...attrs,
    strokeColor: 'lightgrey',
  });

  let aColor = 'blue';
  let cosColor = 'darkred';
  let sinColor = 'darkgreen';
  let shadow = "0 0 0.5em #fff";
  let backgroundCss = "text-shadow: " + Array(8).fill(shadow).join(", ");

  let a = board.create('slider', [
    [-2, -2],
    [2, -2],
    [0, 1, 6.28]
  ], {
    name: 'a',
    highline: { strokeColor: aColor },
    label: { strokeColor: aColor },
  });
  let cosText = board.create('text', [-2, -1.5, () => 'cos(a) = ' + Math.cos(a.Value()).toFixed(2) ], {
    color: cosColor,
  });
  let sinText = board.create('text', [-2, -1.75, () => 'sin(a) = ' + Math.sin(a.Value()).toFixed(2) ], {
    color: sinColor,
  });

  let origin = board.create('point', [0, 0], {visible: false});
  let unitX = board.create('point', [1, 0], {visible: false});

  let target = board.create('point', [
    () => Math.cos(a.Value()),
    () => Math.sin(a.Value()),
  ], {...attrs, color: aColor, visible: false});
  let arc = board.create('arc', [origin, unitX, target], {
    ...attrs,
    strokeColor: aColor,
    name: 'a',
    withLabel: true,
    label: {
      color: aColor,
      offset: [0, 0],
    },
  });

  let cosAX = board.create('point', [() => Math.cos(a.Value()), 0], {visible: false});

  let cosA = board.create('line', [origin, cosAX], { color: cosColor, straightFirst: false, straightLast: false });
  let cosAMid = board.create('midpoint', [cosA.point1, cosA.point2], { visible: false });
  let cosAMidLabel = board.create('text', [0, 0, 'cos(a)'], {
    anchor: cosAMid,
    fixed: true,
    color: cosColor,
    anchorX: 'middle',
    anchorY: 'top',
    cssStyle: backgroundCss,
  });

  let sinA = board.create('line', [cosAX, target], { color: sinColor, straightFirst: false, straightLast: false });
  let sinAMid = board.create('midpoint', [sinA.point1, sinA.point2], { visible: false });
  let sinAMidLabel = board.create('text', [0.0625, 0, 'sin(a)'], {
    anchor: sinAMid,
    fixed: true,
    color: sinColor,
    anchorX: 'left',
    anchorY: 'middle',
    cssStyle: backgroundCss,
  });

  board.unsuspendUpdate();
}</script>

That is to say, if you have a circle centered at the [origin](https://en.wikipedia.org/wiki/Origin_(mathematics)) (the coordinate `⟨0, 0⟩`) with a [radius](https://en.wikipedia.org/wiki/Radius) of 1, and you walk along the [circumference](https://en.wikipedia.org/wiki/Circumference) from `⟨1, 0⟩` towards `⟨0, 1⟩` for a distance of `a`, then your position has the [cartesian coordinates](https://en.wikipedia.org/wiki/Cartesian_coordinate_system) `⟨cos(a), sin(a)⟩`. In this situation, `a` represents an angle measured in [radians](https://en.wikipedia.org/wiki/Radian); whereas there are 360 degrees in a full revolution, there are only 2π radians in a full revolution.

It's often useful to convert an angle to a [slope](https://en.wikipedia.org/wiki/Slope), which can be done with `sin(a) / cos(a)`. This is so common it has a special name: `tan(a)` (pronounced tangent of `a`).

If you want to go from cartesian coordinates to an angle (measured in radians, of course), you can do so with `atan2(y, x)` (`atan` is pronounced arc-tangent, with "arc" meaning inverse).

Foundational Vector Algebra
---------------------------

(if you're familiar with vector algebra, feel free to [skip to the next section](#rotating-points))

When we talk about [vectors](https://en.wikipedia.org/wiki/Vector_(mathematics_and_physics)) in the context of math, we don't exactly mean C++'s `vector` or Rust's `Vec`, though those two types almost certainly drew their name from the mathematical concept. There are generally a couple of ways to think about vectors:

* The geometry-focused definition: A length and direction
* The programmer/engineer definition: A tuple (list) of numbers, usually acting as coordinates, offsets or weights

We usually depict vectors with arrows, but there's an important wrinkle to be aware of: Whereas an arrow may appear to have a particular start and end, a vector is more like an offset or displacement; the only thing that matters is where the end is _relative to the start_. To make that concrete, in the diagram below, `A` and `B` are the same vector, but `C` and `D` aren't equal to any other vector shown (unless you've elected to move the dot so they all have zero length):

<figure>
  <div id="figure-vector-equivalence" class="jxgbox" style="aspect-ratio: 3/2"></div>
</figure>
<script type="text/javascript">{
  let board = JXG.JSXGraph.initBoard('figure-vector-equivalence', {
    boundingbox: [-15, 10, 15, -10],
    axis: true,
    showNavigation: false,
  });
  board.suspendUpdate();

  let origin = board.create('point', [0, 0], { fixed: true, visible: false });
  let bEnd = board.create('point', [1, 3], { withLabel: false });
  let b = board.create('arrow', [origin, bEnd], { name: "B", withLabel: true });

  let aStart = board.create('point', [-8, -2], { visible: false });
  let aEnd = board.create('parallelpoint', [origin, bEnd, aStart], { visible: false });
  let a = board.create('arrow', [aStart, aEnd], { name: "A", withLabel: true });

  let cStart = board.create('point', [2, -5], { visible: false });
  let cEnd = board.create('point', [
    () => cStart.X() + 0.3 * bEnd.X(),
    () => cStart.Y() + 0.3 * bEnd.Y(),
  ], { visible: false });
  let c = board.create('arrow', [cStart, cEnd], { name: "C", withLabel: true, color: 'darkgreen' });

  let dStart = board.create('point', [5, 1], { visible: false });
  let dEnd = board.create('point', [
    () => dStart.X() + bEnd.Y(),
    () => dStart.Y() - bEnd.X(),
  ], { visible: false });
  let d = board.create('arrow', [dStart, dEnd], { name: "D", withLabel: true, color: 'darkred' });

  board.unsuspendUpdate();
}</script>

As programmers, we usually represent a vector as the cartesian coordinates of where its endpoint would be if its start were situated at the origin; the mathy way of writing that is `⟨x, y, z⟩` (though a vector may have any number of components).

A pair of vectors may be added component-wise, i.e. `⟨a, b, c⟩ + ⟨d, e, f⟩ = ⟨a + d, b + e, c + f⟩`. This corresponds to adding vectors head-to-tail:

<figure>
  <div id="figure-vector-addition" class="jxgbox" style="aspect-ratio: 3/2"></div>
</figure>
<script type="text/javascript">{
  let board = JXG.JSXGraph.initBoard('figure-vector-addition', {
    boundingbox: [-15, 10, 15, -10],
    axis: true,
    showNavigation: false,
  });
  board.suspendUpdate();

  let origin = board.create('point', [0, 0], { fixed: true, visible: false });
  let aEnd = board.create('point', [3, 8], { withLabel: false });
  let a = board.create('arrow', [origin, aEnd], { color: 'darkred' });
  let bEnd = board.create('point', [9, -2], { withLabel: false });
  let b = board.create('arrow', [origin, bEnd], { color: 'darkblue' });
  let abEnd = board.create('parallelpoint', [origin, aEnd, bEnd], { visible: false });
  let ab = board.create('arrow', [origin, abEnd], { color: 'purple' });
  let parallelA = board.create('arrow', [bEnd, abEnd], { color: 'darkred' });
  let parallelB = board.create('arrow', [aEnd, abEnd], { color: 'darkblue' });

  let aMid = board.create('midpoint', [origin, aEnd], { visible: false });
  let aLabel = board.create('text', [-0.5, 0, 'A'], { anchor: aMid, fixed: true });
  let bMid = board.create('midpoint', [origin, bEnd], { visible: false });
  let bLabel = board.create('text', [0, -0.5, 'B'], { anchor: bMid, fixed: true });
  let abMid = board.create('midpoint', [origin, abEnd], { visible: false });
  let abLabel = board.create('text', [-0.5, 0.5, 'A + B'], { anchor: abMid, fixed: true });

  board.unsuspendUpdate();
}</script>

A vector may be multiplied by a [scalar](https://en.wikipedia.org/wiki/Scalar_(mathematics)) (a 1-dimensional number), i.e. `a * ⟨b, c, d⟩ = ⟨a * b, a * c, a * d⟩`. This scales (resizes) the vector, hence the term "scalar".

<figure>
  <div id="figure-vector-scaling" class="jxgbox" style="aspect-ratio: 3/2"></div>
</figure>
<script type="text/javascript">{
  let board = JXG.JSXGraph.initBoard('figure-vector-scaling', {
    boundingbox: [-15, 10, 15, -10],
    axis: true,
    showNavigation: false,
  });
  board.suspendUpdate();

  let a = board.create('slider', [
    [-10, -8],
    [10, -8],
    [-2, 1.5, 2]
  ], { name: 'a' });

  let origin = board.create('point', [0, 0], { fixed: true, visible: false });
  let vEnd = board.create('point', [6, 2], { name: "V", label: { autoPosition: true } });
  let v = board.create('arrow', [origin, vEnd]);

  let avEnd = board.create('point', [
    () => a.Value() * vEnd.X(),
    () => a.Value() * vEnd.Y(),
  ], { color: 'blue', name: "aV", label: { autoPosition: true } });
  let av = board.create('arrow', [origin, avEnd]);

  board.unsuspendUpdate();
}</script>

For anything we're interested in, a vector's length is determined through the Pythagorean theorem, i.e. `|⟨a, b, c⟩| = sqrt(a² + b² + c²)`. A [unit vector](https://en.wikipedia.org/wiki/Unit_vector) is any vector with a length of 1.

Rotating Points
---------------

Now that we have some tools on our belt, let's figure out how to solve a much simpler version of the 3D-world-to-screen mapping problem: 2D rotation.

Imagine you're building a 2D game where you can rotate the camera or world, perhaps something like [Cameltry](https://en.wikipedia.org/wiki/Cameltry). So you have a bunch of 2D points, representing camera-relative positions of various objects, and you want to rotate them all around the origin by one particular angle. In other words, you want to build something like this: (light grey dots represent original unrotated positions)

<script type="text/javascript">
  var smileyCoords = [
    // Left eye
    [-4.5, 2.5],
    [-4, 3],
    [-3, 3],
    [-2.5, 2.5],
    // Right eye
    [2.5, 2.5],
    [3, 3],
    [4, 3],
    [4.5, 2.5],
    // Smile
    [-5, -1.5],
    [-4, -2.4],
    [-3, -3.1],
    [-2, -3.6],
    [-1, -3.9],
    [0, -4],
    [1, -3.9],
    [2, -3.6],
    [3, -3.1],
    [4, -2.4],
    [5, -1.5],
  ];
</script>

<figure>
  <div id="figure-rotation" class="jxgbox" style="aspect-ratio: 3/2"></div>
</figure>
<script type="text/javascript">{
  let board = JXG.JSXGraph.initBoard('figure-rotation', {
    boundingbox: [-15, 9, 15, -11],
    axis: true,
    showNavigation: false,
  });
  board.suspendUpdate();

  let angle = board.create('slider', [
    [-10, -8],
    [10, -8],
    [-360, 30, 360]
  ], {
    name: 'angle',
    snapWidth: 1,
  });
  let rotation = board.create('transform', [() => angle.Value() * Math.PI / 180, [0, 0]], {type: 'rotate'});

  let attrs = {
    fixed: true,
    size: 3,
    withLabel: false,
  };
  let points = smileyCoords.map(p => board.create('point', p, {...attrs, color: 'lightgrey'}));
  let rotatedPoints = points.map(p => board.create('point', [p, rotation], {...attrs, color: 'blue'}));

  board.unsuspendUpdate();
}</script>

How might you do that? For me, the most obvious way would be a bit of trig: Convert the points to [polar coordinates](https://en.wikipedia.org/wiki/Polar_coordinate_system) (angle and distance from origin), adjust the angle, then convert back. The code for that might look something like this: (I'm using [`glam`](https://crates.io/crates/glam) for the examples because it's fairly simple and minimal)

```rust
use glam::Vec2;

#[derive(Clone, Copy)]
pub struct Radians(pub f32);

/// Rotates p around the origin by rotation_angle from the x axis towards the y axis
#[inline]
pub fn rotate(p: Vec2, rotation_angle: Radians) -> Vec2 {
    // Convert to polar coordinates
    let angle = f32::atan2(p.y, p.x);
    let distance = (p.x * p.x + p.y * p.y).sqrt();

    // Apply the actual rotation
    let new_angle = angle + rotation_angle.0;

    // Convert back to cartesian coordinates
    Vec2 {
        x: distance * new_angle.cos(),
        y: distance * new_angle.sin(),
    }
}
```

That _works_, but it's much slower than it needs to be. In computer graphics, we generally regard additions and multiplications as dirt-cheap, and stuff like `sqrt(x)`, `atan2(y, x)`, `sin(x)`, or `cos(x)` as expensive. Not only does the above code have to call all those nasty expensive functions, but their inputs all ultimately depend on `p` so the expensive stuff needs to be recalculated _for every single point that gets rotated_. Fortunately, there's a way to factor out the expensive computations...

Vector algebra courses sometimes like to make a big fuss over how cartesian coordinates are just a shorthand for a [linear combination](https://en.wikipedia.org/wiki/Linear_combination) of [basis vectors](https://en.wikipedia.org/wiki/Basis_(linear_algebra)), i.e. `⟨x, y⟩ = x * ⟨1, 0⟩ + y * ⟨0, 1⟩`. This turns out to be important because if you have `basis_x = ⟨1, 0⟩` and `basis_y = ⟨0, 1⟩` and a point `p = ⟨x, y⟩ = x * basis_x + y * basis_y`, then rotating `basis_x` and `basis_y` will also rotate `p`. This is another one of those circumstances where an interactive diagram is worth a thousand words, so...

<figure>
  <div id="figure-basis-rotation" class="jxgbox" style="aspect-ratio: 3/2"></div>
</figure>
<script type="text/javascript">{
  let board = JXG.JSXGraph.initBoard('figure-basis-rotation', {
    boundingbox: [-7.5, 5, 7.5, -5],
    axis: true,
    showNavigation: false,
  });
  board.suspendUpdate();

  let angle = board.create('slider', [
    [-5, -4],
    [5, -4],
    [-360, 30, 360]
  ], {
    name: 'angle',
    snapWidth: 1,
  });

  let origin = board.create('point', [0, 0], { fixed: true, visible: false });
  let basisXEnd = board.create('point', [
    () => Math.cos(angle.Value() * Math.PI / 180),
    () => Math.sin(angle.Value() * Math.PI / 180),
  ], { visible: false } );
  let basisYEnd = board.create('point', [
    () => -basisXEnd.Y(),
    () => basisXEnd.X(),
  ], { visible: false } );
  let scaledBasisXEnd = board.create('point', [
    () => 4 * basisXEnd.X(),
    () => 4 * basisXEnd.Y(),
  ], { visible: false } );
  let pEnd = board.create('point', [
    () => scaledBasisXEnd.X() + 2 * basisYEnd.X(),
    () => scaledBasisXEnd.Y() + 2 * basisYEnd.Y(),
  ], { name: "p = 4 * basix_x + 2 * basis_y", fixed: true, label: { parse: false } } );

  let scaledBasisX = board.create('arrow', [origin, scaledBasisXEnd], { color: '#f88' });
  let scaledBasisY = board.create('arrow', [scaledBasisXEnd, pEnd], { color: '#8f8' });

  let basisX = board.create('arrow', [origin, basisXEnd], { color: 'darkred' });
  let basisY = board.create('arrow', [origin, basisYEnd], { color: 'darkgreen' });

  let basisXMid = board.create('midpoint', [origin, basisXEnd], { visible: false });
  let basisXLabel = board.create('text', [0, 0, 'basis_x'], {
    anchor: basisXMid,
    fixed: true,
    parse: false,
  });
  let basisYMid = board.create('midpoint', [origin, basisYEnd], { visible: false });
  let basisYLabel = board.create('text', [0, 0, 'basis_y'], {
    anchor: basisYMid,
    fixed: true,
    parse: false,
    anchorX: 'right',
  });

  let scaledBasisXMid = board.create('midpoint', [origin, scaledBasisXEnd], { visible: false });
  let scaledBasisXLabel = board.create('text', [0, 0, '4 * basis_x'], {
    anchor: scaledBasisXMid,
    fixed: true,
    parse: false,
  });
  let scaledBasisYMid = board.create('midpoint', [scaledBasisXEnd, pEnd], { visible: false });
  let scaledBasisYLabel = board.create('text', [0, 0, '2 * basis_y'], {
    anchor: scaledBasisYMid,
    fixed: true,
    parse: false,
  });

  board.unsuspendUpdate();
}</script>

Rewriting the rotation code in terms of this insight yields:

```rust
use glam::Vec2;

#[derive(Clone, Copy)]
pub struct Radians(pub f32);

/// Rotates p around the origin by rotation_angle from the x axis towards the y axis
#[inline]
pub fn rotate(p: Vec2, rotation_angle: Radians) -> Vec2 {
    // Create rotated basis vectors
    let basis_x = Vec2 {
        x: rotation_angle.0.cos(),
        y: rotation_angle.0.sin(),
    };
    let basis_y = Vec2 {
        x: -basis_x.y,
        y: basis_x.x,
    };

    // Interpret p relative to basis vectors
    p.x * basis_x + p.y * basis_y
}
```

**Note: The crux of this article is how the above code works. Before moving on, you may want to study the above diagram and code until you're confident you understand them.**

The above has the obvious advantage that it doesn't need `atan2(y, x)` or `sqrt(x)`, but it also has a subtler, more important benefit: `sin(x)` and `cos(x)` don't depend on `p`, so we can split the expensive part from the cheap part, and compute the expensive part once, instead of per point. Rewriting the code to allow that:

```rust
use glam::Vec2;

#[derive(Clone, Copy)]
pub struct Radians(pub f32);

#[derive(Clone, Copy)]
pub struct Rotation {
    basis_x: Vec2,
    basis_y: Vec2,
}

impl Rotation {
    /// Creates a rotation around the origin by rotation_angle from the x axis towards the y axis
    pub fn new(rotation_angle: Radians) -> Self {
        let basis_x = Vec2 {
            x: rotation_angle.0.cos(),
            y: rotation_angle.0.sin(),
        };
        let basis_y = Vec2 {
            x: -basis_x.y,
            y: basis_x.x,
        };

        Self { basis_x, basis_y }
    }

    /// Applies this rotation to p
    #[inline]
    pub fn apply(&self, p: Vec2) -> Vec2 {
        p.x * self.basis_x + p.y * self.basis_y
    }
}
```

With this new code, `Rotation::new(angle)` is still expensive, but `rotation.apply(p)` is almost free: it's a mere 4 multiplications and 2 additions! At least on my computer, `rotation.apply(p)` is 50x faster than the trigonometry-based rotation when operating on large batches of points. Pretty good!

Scaling and Translation
-----------------------

Suppose we want to adjust a camera's zoom level. That could be handled by scaling how far points are from the origin:

<figure>
  <div id="figure-scaling" class="jxgbox" style="aspect-ratio: 3/2"></div>
</figure>
<script type="text/javascript">{
  let board = JXG.JSXGraph.initBoard('figure-scaling', {
    boundingbox: [-15, 9, 15, -11],
    axis: true,
    showNavigation: false,
  });
  board.suspendUpdate();

  let scaleX = board.create('slider', [
    [-10, -8],
    [10, -8],
    [-3, 1.5, 3]
  ], {
    name: 'x scale',
    snapWidth: 0.1,
  });
  let scaleY = board.create('slider', [
    [-10, -9],
    [10, -9],
    [-3, 1.5, 3]
  ], {
    name: 'y scale',
    snapWidth: 0.1,
  });

  let scale = board.create('transform', [() => scaleX.Value(), () => scaleY.Value()], {type: 'scale'});

  let attrs = {
    fixed: true,
    size: 3,
    withLabel: false,
  };
  let points = smileyCoords.map(p => board.create('point', p, {...attrs, color: 'lightgrey'}));
  let rotatedPoints = points.map(p => board.create('point', [p, scale], {...attrs, color: 'blue'}));

  board.unsuspendUpdate();
}</script>

In much the same way that rotating bases vectors will rotate any point written in terms of those basis vectors, scaling basis vectors will scale any point written in terms of them:

<figure>
  <div id="figure-basis-scaling" class="jxgbox" style="aspect-ratio: 3/2"></div>
</figure>
<script type="text/javascript">{
  let board = JXG.JSXGraph.initBoard('figure-basis-scaling', {
    boundingbox: [-7.5, 5, 7.5, -5],
    axis: true,
    showNavigation: false,
  });
  board.suspendUpdate();

  let scaleX = board.create('slider', [
    [-5, -3],
    [5, -3],
    [-3, 1.5, 3]
  ], {
    name: 'x scale',
    snapWidth: 0.1,
  });
  let scaleY = board.create('slider', [
    [-5, -4],
    [5, -4],
    [-3, 1.5, 3]
  ], {
    name: 'y scale',
    snapWidth: 0.1,
  });

  let origin = board.create('point', [0, 0], { fixed: true, visible: false });
  let basisXEnd = board.create('point', [
    () => scaleX.Value(),
    0,
  ], { visible: false } );
  let basisYEnd = board.create('point', [
    0,
    () => scaleY.Value(),
  ], { visible: false } );
  let scaledBasisXEnd = board.create('point', [
    () => 4 * basisXEnd.X(),
    () => 4 * basisXEnd.Y(),
  ], { visible: false } );
  let pEnd = board.create('point', [
    () => scaledBasisXEnd.X() + 2 * basisYEnd.X(),
    () => scaledBasisXEnd.Y() + 2 * basisYEnd.Y(),
  ], { name: "p = 4 * basix_x + 2 * basis_y", fixed: true, label: { parse: false } } );

  let scaledBasisX = board.create('arrow', [origin, scaledBasisXEnd], { color: '#f88' });
  let scaledBasisY = board.create('arrow', [scaledBasisXEnd, pEnd], { color: '#8f8' });

  let basisX = board.create('arrow', [origin, basisXEnd], { color: 'darkred' });
  let basisY = board.create('arrow', [origin, basisYEnd], { color: 'darkgreen' });

  let basisXMid = board.create('midpoint', [origin, basisXEnd], { visible: false });
  let basisXLabel = board.create('text', [0, 0, 'basis_x'], {
    anchor: basisXMid,
    fixed: true,
    parse: false,
  });
  let basisYMid = board.create('midpoint', [origin, basisYEnd], { visible: false });
  let basisYLabel = board.create('text', [0, 0, 'basis_y'], {
    anchor: basisYMid,
    fixed: true,
    parse: false,
    anchorX: 'right',
  });

  let scaledBasisXMid = board.create('midpoint', [origin, scaledBasisXEnd], { visible: false });
  let scaledBasisXLabel = board.create('text', [0, 0, '4 * basis_x'], {
    anchor: scaledBasisXMid,
    fixed: true,
    parse: false,
  });
  let scaledBasisYMid = board.create('midpoint', [scaledBasisXEnd, pEnd], { visible: false });
  let scaledBasisYLabel = board.create('text', [0, 0, '2 * basis_y'], {
    anchor: scaledBasisYMid,
    fixed: true,
    parse: false,
  });

  board.unsuspendUpdate();
}</script>

Because the `Rotation` data structure just rewrites a point in terms of new basis vectors, it's already capable of handling scaling if we supply a new constructor that scales the basis vectors instead of rotating them. While we're at it, we'll rename `Rotation` to `Transformation` to reflect that it's not just doing rotations any more:

```rust
use glam::Vec2;

#[derive(Clone, Copy)]
pub struct Radians(pub f32);

#[derive(Clone, Copy)]
pub struct Transformation {
    pub basis_x: Vec2,
    pub basis_y: Vec2,
}

impl Transformation {
    /// Applies this transformation to p
    #[inline]
    pub fn apply(&self, p: Vec2) -> Vec2 {
        p.x * self.basis_x + p.y * self.basis_y
    }
}

/// Creates a rotation around the origin by rotation_angle from the x axis towards the y axis
pub fn rotation(rotation_angle: Radians) -> Transformation {
    let basis_x = Vec2 {
        x: rotation_angle.0.cos(),
        y: rotation_angle.0.sin(),
    };
    let basis_y = Vec2 {
        x: -basis_x.y,
        y: basis_x.x,
    };

    Transformation { basis_x, basis_y }
}

/// Creates a scaling transformation
pub fn scaling(scale: f32) -> Transformation {
    let basis_x = Vec2 { x: scale, y: 0.0 };
    let basis_y = Vec2 { x: 0.0, y: scale };

    Transformation { basis_x, basis_y }
}
```

With some more tweaks, we can get `Transformation` to also handle translation (moving points around, to handle camera panning)...

<figure>
  <div id="figure-translation" class="jxgbox" style="aspect-ratio: 3/2"></div>
</figure>
<script type="text/javascript">{
  let board = JXG.JSXGraph.initBoard('figure-translation', {
    boundingbox: [-15, 9, 15, -11],
    axis: true,
    showNavigation: false,
  });
  board.suspendUpdate();

  let translationX = board.create('slider', [
    [-10, -8],
    [5, -8],
    [-15, -5, 15]
  ], {
    name: 'x translation',
    snapWidth: 0.5,
  });
  let translationY = board.create('slider', [
    [-10, -9],
    [5, -9],
    [-15, 2, 15]
  ], {
    name: 'y translation',
    snapWidth: 0.5,
  });

  let translation = board.create('transform', [() => translationX.Value(), () => translationY.Value()], {type: 'translate'});

  let attrs = {
    fixed: true,
    size: 3,
    withLabel: false,
  };
  let points = smileyCoords.map(p => board.create('point', p, {...attrs, color: 'lightgrey'}));
  let rotatedPoints = points.map(p => board.create('point', [p, translation], {...attrs, color: 'blue'}));

  board.unsuspendUpdate();
}</script>

But first I should probably come clean: **We've re-invented [matrices](https://en.wikipedia.org/wiki/Matrix_(mathematics)) and `Transformation` is just a 2x2 matrix.**

You see, functions of the form `f(⟨a, b, c, ...⟩) = a * basis_a + b * basis_b + c * basis_c + ...` are used _everywhere_, so there's a special name for them: [linear transformations](https://en.wikipedia.org/wiki/Linear_map). Mathematicians, lazy reprobates that they are, have decided to fund their syntactic sugar addiction by pawning off most of the notation of linear transformations, leaving behind only a grid of coefficients with one column per basis vector. For example, the mathematical notation for `Transformation` would be:

| `basis_x.x` | `basis_y.x` |
| `basis_x.y` | `basis_y.y` |

(well, ok, if this were _real_ mathematical notation, there wouldn't be any lines separating the 4 values... I told you mathematicians were lazy!) Anyway, when you represent a linear transformation as a 2D grid, it's called a "matrix".

Matrices are very common in computer graphics libraries and `glam` is no exception. We _could_ shrink our `Transformation` code by rewriting it to use matrices...

```rust
use glam::{Mat2, Vec2};

#[derive(Clone, Copy)]
pub struct Radians(pub f32);

#[derive(Clone, Copy)]
pub struct Transformation(Mat2);

impl Transformation {
    /// Applies this transformation to p
    #[inline]
    pub fn apply(&self, p: Vec2) -> Vec2 {
        self.0 * p
    }
}

/// Creates a rotation around the origin by rotation_angle from the x axis towards the y axis
pub fn rotation(rotation_angle: Radians) -> Transformation {
    Transformation(Mat2::from_angle(rotation_angle.0))
}

/// Creates a scaling transformation
pub fn scaling(scale: f32) -> Transformation {
    Transformation(Mat2::from_diagonal(Vec2::splat(scale)))
}
```

...but I'm not sure there's much point to writing such thin wrappers around `glam`. From now on, let's just use its matrices directly and dispense with `Transformation` altogether.

Before moving on from the above code, I want to draw attention to the fact that it applies the matrix to a vector by multiplying the two together. Unlike regular scalar multiplication, any multiplication involving matrices ***isn't*** [commutative](https://en.wikipedia.org/wiki/Commutative_property) (so order matters). With the way I've been describing matrices (one column per basis vector), you want to multiply with the matrix on the left and the vector on the right. An easy way to remember this is that a matrix represents a function, and much like how you'd write `transform(point)` when `transform` is a function, you should write `transform * point` when `transform` is a matrix.

Back to translation matrices... if you think about it, you may notice a roadblock to using a `Mat2` to implement translation: Suppose we have the vector `⟨0, 0⟩`... applying a 2x2 matrix to that vector yields `0 * basis_x + 0 * basis_y = ⟨0, 0⟩`. In other words... if a vector is all zeros, there's no matrix that can be applied to it to produce a non-zero vector.

Fortunately, that problem is easy enough to kludge around: Instead of representing a point `(x, y)` with the vector `⟨x, y⟩`, we'll use the vector `⟨x, y, 1⟩`. Then we can implement translation with a 3x3 matrix that looks like this:

| 1 | 0 | `translation.x` |
| 0 | 1 | `translation.y` |
| 0 | 0 | 1  |

This ends up representing the function `f(⟨x, y, 1⟩) = x * ⟨1, 0, 0⟩ + y * ⟨0, 1, 0⟩ + 1 * ⟨translation.x, translation.y, 1⟩ = ⟨x + translation.x, y + translation.y, 1⟩` which is what we want. Also, note that it keeps the final component intact with a value of 1, so that we can chain several translations together. `glam` provides [`Mat3::from_translation(vec2)`](https://docs.rs/glam/0.21.3/glam/f32/struct.Mat3.html#method.from_translation) to generate the above translation matrix.

Admittedly, tacking an extra component onto vectors feels like a weird ugly hack, but it'll help us in the long run: We'll eventually expand it to [homogeneous coordinates](https://en.wikipedia.org/wiki/Homogeneous_coordinates) which will be critical for 3D perspective transformations.

Bringing everything together in 2D
---------------------------------

We now have all the building blocks of 2D spatial transformations: translation, rotation and scaling! However, you might be starting to wonder: if you have a camera with full freedom of movement, will you have to pass around 3 different matrices to anything that wants to draw stuff? If you represent an entity's position/rotation/scaling with matrices, does that add another 3 matrices that must be applied to each point of that entity? That seems... less than ideal.

Fortunately, matrices can be [chained together via multiplication](https://en.wikipedia.org/wiki/Matrix_multiplication), which will produce a _single_ matrix that behaves identically to the composition of the matrices that went into it. To rephrase in terms of math, `matrix_a * (matrix_b * vector) = (matrix_a * matrix_b) * vector`. (fun note: you can think of a vector as a matrix with a single column, and applying a matrix to said vector works out to just be a special case of matrix multiplication)

So... this means if we have a 2D camera, we can easily construct a single transformation to map from screen coordinates to world coordinates:

```rust
use glam::{Mat3, Vec2};

struct Camera {
    // The world coordinates that correspond with the middle of the screen
    position: Vec2,
    // How much to rotate from the +x axis towards the +y axis
    rotation_radians: f32,
    // How much game-world distance is covered by the screen's diagonal
    diagonal_length: f32,
}

impl Camera {
    // Assumes the screen's coordinates range from -1..+1 in both axes
    // and the world's y axis points in the same direction as the screen's
    // Aspect ratio is screen_resolution.height / screen_resolution.width
    fn screen_to_world(&self, aspect_ratio: f32) -> Mat3 {
        // We want:
        //   scale_y / scale_x == aspect_ratio
        //   scale_x^2 + scale_y^2 == diagonal_length^2
        // Solving for scale_x and scale_y with algebra yields:
        let scale_x =
            self.diagonal_length * self.diagonal_length / (1.0 + aspect_ratio * aspect_ratio);
        let scale_y = scale_x * aspect_ratio;

        // Note: This can be further improved by using Mat3::from_scale_angle_translation
        let scale = Mat3::from_scale(Vec2 {
            x: scale_x,
            y: scale_y,
        });
        let rotation = Mat3::from_angle(self.rotation_radians);
        let translation = Mat3::from_translation(self.position);
        translation * rotation * scale
    }
}
```

...but if we want to figure out where on the screen to draw objects, we need to go from world coordinates to screen coordinates and `camera.screen_to_world()` is the wrong way around. Fortunately, you can [invert matrices that represent invertible functions](https://en.wikipedia.org/wiki/Invertible_matrix):

```
    let world_to_screen = camera.screen_to_world().inverse();
```

Inverting a general-purpose matrix is somewhat expensive, but if you know you're just using the matrix to store a position/rotation/scale, graphics libraries often provide ways to do the inversion more cheaply yet accurately. For example, `glam` provides an [`Affine2`](https://docs.rs/glam/0.21.3/glam/f32/struct.Affine2.html) type that is similar to a [`Mat3`](https://docs.rs/glam/0.21.3/glam/f32/struct.Mat3.html) generated from [`Mat3::from_scale_angle_translation`](https://docs.rs/glam/0.21.3/glam/f32/struct.Mat3.html#method.from_scale_angle_translation), except it's cheaper to invert.

There's one last thing worth mentioning before we go on to 3D... It's often useful to distinguish between points and directions. If you have a vector that represents a particular point in the game world, you want a world-to-screen transformation to apply translation. If you have a vector that represents a direction (such as a navigation marker on the HUD pointing towards an important location), you want the world-to-screen transformation to rotate it appropriately, but not translate it. Turns out you can do that by setting the last component to 0 instead of 1: The 0 will be multiplied against the last column of the matrix (the translation column), zeroing out any translation but leaving the rotation/scaling columns intact. Graphics libraries sometimes recognize this distinction. `glam` provides [`Mat3::transform_point2`](https://docs.rs/glam/0.21.3/glam/f32/struct.Mat3.html#method.transform_point2) (for points obviously) and [`Mat3::transform_vector2`](https://docs.rs/glam/0.21.3/glam/f32/struct.Mat3.html#method.transform_vector2) (for directions) to extend 2D vectors with an appropriate 3rd component before applying the matrix.

Your first 3D perspective
-------------------------

We now have a pretty good grasp on how to use matrices to represent 2D spatial transformations: we can use [`Vec3`](https://docs.rs/glam/0.21.3/glam/f32/struct.Vec3.html) to represent 2D points and directions, and we can use [`Mat3`](https://docs.rs/glam/0.21.3/glam/f32/struct.Mat3.html) to represent any number of 2D translation/rotation/scaling transforms. Extending this to 3D is trivial: just use [`Vec4`](https://docs.rs/glam/0.21.3/glam/f32/struct.Vec4.html) and [`Mat4`](https://docs.rs/glam/0.21.3/glam/f32/struct.Mat4.html). However, there's one last piece of the puzzle we need to draw 3D scenes: Perspective transformations.

You know how objects look smaller as they get farther away? That's what perspective transformations do; they map a [view frustum](https://en.wikipedia.org/wiki/Viewing_frustum) to normalized device coordinates (screen coordinates and depth).

<figure style="display: flex">
  <div id="figure-perspective-camera" class="jxgbox" style="flex: 1; aspect-ratio: 1/1"></div>
  <div id="figure-perspective-side" class="jxgbox" style="flex: 2; aspect-ratio: 2/1; margin-left: 1em"></div>
</figure>
<script type="text/javascript">{
  let board = JXG.JSXGraph.initBoard('figure-perspective-side', {
    boundingbox: [-1, 1, 3, -1],
    showNavigation: false,
  });
  board.suspendUpdate();

  let cameraUrl = "{{base}}/camera.svg";
  let camera = board.create('image', [cameraUrl, [-0.75, -1], [0.75, 1.33] ], { fixed: true });

  let frustumTop = board.create('line', [[0, 0], [3, 1]], { fixed: true, straightFirst: false, color: 'lightgrey' });
  let frustumBottom = board.create('line', [[0, 0], [3, -1]], { fixed: true, straightFirst: false, color: 'lightgrey' });

  let chickenUrl = "{{base}}/chicken.svg";
  let chicken = board.create('image', [chickenUrl, [2, -1], [0.25, 1] ]);
  let dragme = board.create('text', [
    () => chicken.X() + chicken.W() / 2,
    () => chicken.Y() + chicken.H(),
    "Drag me!"
  ], {
    fixed: true,
    anchorX: 'middle',
    anchorY: 'bottom',
  });

  board.unsuspendUpdate();

  let board2 = JXG.JSXGraph.initBoard('figure-perspective-camera', {
    boundingbox: [-1, 1, 1, -1],
    showNavigation: false,
  });
  board2.suspendUpdate();

  let midX = () => chicken.X() + chicken.W() / 2;
  let scale = () => 3 / midX();

  let viewChicken = board2.create('image', [chickenUrl, [
    () => scale() / -2,
    () => scale() * chicken.Y(),
  ], [
    scale,
    scale,
  ] ]);

  board.addChild(board2);
  board2.unsuspendUpdate();
}</script>

Let's start off creating a perspective transformation matrix for the easiest possible view frustum: We'll place the camera at the origin, looking towards the +Z axis, and we'll give it a 90° [field of view](https://en.wikipedia.org/wiki/Field_of_view_in_video_games) (usually abbreviated FOV) both vertically and horizontally. From the side, it'll look like this:

<figure>
  <div id="figure-frustum" class="jxgbox" style="aspect-ratio: 3/2"></div>
</figure>
<script type="text/javascript">{
  let board = JXG.JSXGraph.initBoard('figure-frustum', {
    boundingbox: [-2, 2, 4, -2],
    showNavigation: false,
    axis: true,
    zoomY: -1.0,
    defaultAxes: {
      x: {
        name: 'Z',
        withLabel: true,
        label: {
          position: 'urt',
          offset: [0, 10],
        },
      },
      y: {
        withLabel: true,
        name: 'Y',
        label: {
          position: 'lrt',
          offset: [10, 0],
        },
      },
    },
  });
  board.suspendUpdate();

  let cameraUrl = "{{base}}/camera.svg";
  let camera = board.create('image', [cameraUrl, [-0.75, 1], [0.75, 1.33] ], { fixed: true });

  let frustumTop = board.create('line', [[0, 0], [1, 1]], { fixed: true, straightFirst: false, color: 'lightgrey' });
  let frustumBottom = board.create('line', [[0, 0], [1, -1]], { fixed: true, straightFirst: false, color: 'lightgrey' });

  let screen = board.create('line', [[1, 1], [1, -1]], {
    name: "Screen",
    withLabel: true,
    fixed: true,
    color: 'darkred',
    straightFirst: false,
    straightLast: false,
  });

  let origin = board.create('point', [0, 0], { fixed: true, visible: false });
  let p = board.create('point', [2.5, -1.5], { name: "P" });
  let pRay = board.create('arrow', [origin, p], {
    color: 'teal',
  });

  let screenCoordY = () => NaN; // Seems like I need to supply a value?

  let i = board.create('intersection', [screen, pRay, 0], {
    name: () => "Screen coord Y = " + screenCoordY().toFixed(2),
    parse: false,
    color: 'blue',
  });

  screenCoordY = () => -i.Y();

  board.unsuspendUpdate();
}</script>

In most graphics systems, screen coordinates range from -1 to 1 in both the X and Y axes. If we imagine the screen being a 2x2 square that actually exists in the game world, lying on the `z = 1` plane, then that gives us an easy way to calculate the screen coordinates of a point `P`: We just draw a line from the camera to `P` and see where it intersects the screen. In the diagram above, the vertical red line represents the screen (seen from the side); try moving `P` around to see that the intersection really is calculating the Y screen coordinate of `P`.

So how do we calculate e.g. the intersection? The line to `P` is straight, so we can generate points on it with `y = P.y / P.z * z`. We conveniently placed the screen at `z = 1`, so we get `screen_coord.y = P.y / P.z * 1 = P.y / P.z`. The same logic holds for the X axis so the screen coordinates for a point `P` are `(P.x / P.z, P.y / P.z)`.

Here we run into a problem: We want a matrix that will represent `f(⟨x, y, z, 1⟩) = ⟨x/z, y/z, ?, ?⟩` but matrices can only represent linear transformations and linear transformations can't divide by their arguments. We simply can't build a matrix to do what we want. _However..._ remember that kludgy last component that's always 1? If we redefine the last component to be the length of 1 unit of distance, so that the vector `⟨x, y, z, w⟩` represents the point `(x/w, y/w, z/w)`, that'd allow us to smuggle a division into the matrix. In that case, we could use a matrix like this:

| 1 | 0 | 0 | 0 |
| 0 | 1 | 0 | 0 |
| 0 | 0 | 0 | 0 |
| 0 | 0 | 1 | 0 |

...which represents the function `f(⟨x, y, z, w⟩) = x * ⟨1, 0, 0, 0⟩ + y * ⟨0, 1, 0, 0⟩ + z * ⟨0, 0, 0, 1⟩ + w * ⟨0, 0, 0, 0⟩ = ⟨x, y, 0, z⟩`, and the returned vector represents the point `(x/z, y/z, 0)`, which is what we want!

This trick of interpreting `⟨x, y, z, w⟩` as `(x/w, y/w, z/w)` is known as [homogeneous coordinates](https://en.wikipedia.org/wiki/Homogeneous_coordinates) and although it feels icky for a point to have multiple representations (e.g. `⟨17, 21, 43, 1⟩` and `⟨34, 42, 86, 2⟩` are the same point), it can make a lot of math simpler. Happily, it plays nicely with the rules of matrices, so we can still e.g. chain matrices together with multiplication.

We're not done yet: When drawing a scene, you need some way to ensure nearby geometry is drawn over far away geometry. Although you _could_ try to sort each triangle by how far away it is and draw them from back to front, that'd require you to store and sort an entire scene's worth of triangles (so it'd use a ton of memory _and_ be slow to boot)... and it'd _still_ break whenever two triangles intersect. A much easier approach is use a [depth buffer](https://en.wikipedia.org/wiki/Z-buffering) to store how far from the camera each rendered pixel is, then only draw individual pixels of a triangle that are closer than any previous pixels. However, in order for this to work, we need to map the visible portion of the Z axis to the range 0 to 1. There are a number of different mappings we can choose, and the mapping doesn't have to be linear, it just need to be [monotonic](https://en.wikipedia.org/wiki/Monotonic_function). I'll go with a particularly nice one that maps infinity to 0 and the near-clipping plane (the closest points you can see) to 1:

| 1 | 0 | 0 | 0 |
| 0 | 1 | 0 | 0 |
| 0 | 0 | 0 | `near_z` |
| 0 | 0 | 1 | 0 |

This which represents the function `f(⟨x, y, z, w⟩) = x * ⟨1, 0, 0, 0⟩ + y * ⟨0, 1, 0, 0⟩ + z * ⟨0, 0, 0, 1⟩ + w * ⟨0, 0, near_z, 0⟩ = ⟨x, y, near_z * w, z⟩`, and `⟨x, y, near_z * w, z⟩` represents the point `(x/z, y/z, near_z*w/z)`. It's pretty easy to verify that `near_z / z` maps `z = near_z` to 1 and `z = infinity` to 0. This matrix does what we want: It represents a camera sitting at the origin, pointing towards +Z, with a 90° [FOV](https://en.wikipedia.org/wiki/Field_of_view_in_video_games), and it returns appropriate screen coords and depth for any point passed to it!

Adjusting 3D perspectives
-------------------------

We have a perspective matrix, but it's a bit limited; it's stuck looking at one particular place, and it only draws square images (unless you're ok with distortion). Our most pressing task is moving the camera around. Luckily matrix multiplication (i.e. chaining) solves that problem for us; instead of moving our perspective matrix around, we'll move the world:

```rust
use glam::{Affine3A, Mat4};

struct Camera {
    // Traditionally, when storing an object's translation/rotation, we store it
    // as a transformation that maps from object-relative coordinates to
    // world-relative coordinates. This has the advantage that e.g. the Affine3A's
    // translation is just the object's position in the world.
    position: Affine3A,
    // Distance to the near clipping plane
    near_z: f32,
}

impl Camera {
    // Returns a matrix that maps from world coordinates to clip coordinates.
    // Note: Clip coordinates are essentially screen coords and depth
    fn world_to_clip(&self) -> Mat4 {
        // Repositions the world so that the camera can just sit at the origin
        // looking towards +Z. This preps everything so that our basic
        // perspective matrix is doing the right thing!
        let world_to_camera = self.position.inverse();

        // This is our perspective matrix!
        let camera_to_clip = Mat4::from_cols(
            [1.0, 0.0, 0.0, 0.0].into(),
            [0.0, 1.0, 0.0, 0.0].into(),
            [0.0, 0.0, 0.0, 1.0].into(),
            [0.0, 0.0, self.near_z, 0.0].into(),
        );

        camera_to_clip * world_to_camera
    }
}
```

Next, we need to handle FOV adjustments so that we're not stuck drawing square images (rectangular images require the horizontal FOV to differ from the vertical FOV). Remember the imaginary screen sitting in front of the camera in the game world? If we adjust its width and height, that'll adjust the field of view:

<figure>
  <div id="figure-fov" class="jxgbox" style="aspect-ratio: 3/2"></div>
</figure>
<script type="text/javascript">{
  let board = JXG.JSXGraph.initBoard('figure-fov', {
    boundingbox: [-2, 2, 4, -2],
    showNavigation: false,
    axis: true,
    zoomY: -1.0,
    defaultAxes: {
      x: {
        name: 'Z',
        withLabel: true,
        label: {
          position: 'urt',
          offset: [0, 10],
        },
      },
      y: {
        withLabel: true,
        name: 'Y',
        label: {
          position: 'lrt',
          offset: [10, 0],
        },
      },
    },
  });
  board.suspendUpdate();

  let cameraUrl = "{{base}}/camera.svg";
  let camera = board.create('image', [cameraUrl, [-0.75, 1], [0.75, 1.33] ], { fixed: true });

  let screenPlane = board.create('line', [[1, 0], [1, 1]], { visible: false });
  let screenTop = board.create('glider', [1.0, 1.0, screenPlane], { name: "screen height" });
  let screenBottom = board.create('point', [1, () => -screenTop.Y()], { visible: false });

  let frustumTop = board.create('line', [[0, 0], screenTop], { fixed: true, straightFirst: false, color: 'lightgrey' });
  let frustumBottom = board.create('line', [[0, 0], screenBottom], { fixed: true, straightFirst: false, color: 'lightgrey' });

  let screen = board.create('line', [screenTop, screenBottom], {
    fixed: true,
    color: 'darkred',
    straightFirst: false,
    straightLast: false,
  });

  let origin = board.create('point', [0, 0], { fixed: true, visible: false });
  let p = board.create('point', [2.5, -1.5], { name: "P" });
  let pRay = board.create('arrow', [origin, p], {
    color: 'teal',
  });

  let screenCoordY = () => NaN; // Seems like I need to supply a value?

  let i = board.create('intersection', [screen, pRay, 0], {
    name: () => "Intersection Y = " + screenCoordY().toFixed(2),
    parse: false,
    color: 'blue',
  });

  screenCoordY = () => -i.Y();

  board.unsuspendUpdate();
}</script>

However, once we do that, the intersection of the line-to-`P` with the screen no longer has coordinates that range from -1 to 1; the intersection's Y coordinate now ranges from `-height/2` to `height/2`. We can fix that by dividing the Y output of our perspective matrix by `height/2` (which I'll call `half_height`), and we can even bake that into the perspective matrix itself. Doing the same for X and `half_width` yields:

| 1/`half_width` | 0 | 0 | 0 |
| 0 | 1/`half_height` | 0 | 0 |
| 0 | 0 | 0 | `near_z` |
| 0 | 0 | 1 | 0 |

So... how do we choose `half_width` and `half_height`? The most important thing is that `half_height` and `half_width` have the same proportions as the image you're drawing to, so `half_height = image.height / image.width * half_width`. Beyond that, `half_height = tan(vertical_fov/2)` and `half_width = tan(horizontal_fov/2)`. Probably the easiest thing to do is take a vertical FOV and aspect ratio, and use the vertial FOV to generate `half_height`, then use the aspect ratio to generate the appropriate `half_width`:

```rust
use glam::Mat4;

// vertical_fov is in radians
// aspect_ratio is width/height
fn perspective_infinite_reverse_rh(vertical_fov: f32, aspect_ratio: f32, near_z: f32) -> Mat4 {
    let half_height = (vertical_fov / 2.0).tan();
    let half_width = aspect_ratio * half_height;
    Mat4::from_cols(
        [1.0 / half_width, 0.0, 0.0, 0.0].into(),
        [0.0, 1.0 / half_height, 0.0, 0.0].into(),
        [0.0, 0.0, 0.0, 1.0].into(),
        [0.0, 0.0, near_z, 0.0].into(),
    )
}
```

This gets the job done (and `glam` provides [Mat4::perspective_infinite_reverse_rh](https://docs.rs/glam/0.21.3/glam/f32/struct.Mat4.html#method.perspective_infinite_reverse_rh) to build this matrix for you), but it'll result in an inappropriately large horizontal field of view if the screen (or window) is short and wide. Personally I'm a fan of specifying a diagonal field of view and aspect ratio. Starting with `half_width² + half_height² = tan(diagonal_fov/2)²` and `aspect_ratio = half_width / half_height`, a little algebra shows that `half_height = tan(diagonal_fov/2) / sqrt(aspect_ratio² + 1)`, so:

```rust
use glam::Mat4;

// diagonal_fov is in radians
// aspect_ratio is width/height
fn perspective_infinite_reverse_rh(diagonal_fov: f32, aspect_ratio: f32, near_z: f32) -> Mat4 {
    let half_height = (diagonal_fov / 2.0).tan() / (aspect_ratio * aspect_ratio + 1.0).sqrt();
    let half_width = aspect_ratio * half_height;
    Mat4::from_cols(
        [1.0 / half_width, 0.0, 0.0, 0.0].into(),
        [0.0, 1.0 / half_height, 0.0, 0.0].into(),
        [0.0, 0.0, 0.0, 1.0].into(),
        [0.0, 0.0, near_z, 0.0].into(),
    )
}
```

Hopefully this all makes linear algebra and matrices less mysterious.

Recap
-----

So, how might you put this all together to draw things? Just to give a vague sense of how this math can be used end-to-end:

```rust
use glam::{Affine3A, Mat4, Vec3};

struct World {
    camera: Camera,
    models: Vec<Model>,
}

impl World {
    fn draw(&self, aspect_ratio: f32) {
        let world_to_clip = self.camera.world_to_clip(aspect_ratio);

        for model in &self.models {
            for &model_to_world in &model.instances {
                // Precalculate a transformation directly from the model to the screen,
                // taking the instance's position into account.
                let model_to_clip = world_to_clip * model_to_world;

                // In a real game, we'd use a vertex shader to apply model_to_clip to everything
                let vertices_on_screen = model
                    .vertices
                    .iter()
                    .map(|&v| model_to_clip.project_point3(v));
                // draw(vertices_on_screen);
            }
        }
    }
}

struct Camera {
    position: Affine3A, // cameraspace-to-worldspace transformation
    near_z: f32,
    diagonal_fov: f32,
}

impl Camera {
    // aspect_ratio = width / height
    fn world_to_clip(&self, aspect_ratio: f32) -> Mat4 {
        let world_to_camera = self.position.inverse();

        let half_height =
            (self.diagonal_fov / 2.0).tan() / (aspect_ratio * aspect_ratio + 1.0).sqrt();
        let half_width = aspect_ratio * half_height;
        let camera_to_clip = Mat4::from_cols(
            [1.0 / half_width, 0.0, 0.0, 0.0].into(),
            [0.0, 1.0 / half_height, 0.0, 0.0].into(),
            [0.0, 0.0, 0.0, 1.0].into(),
            [0.0, 0.0, self.near_z, 0.0].into(),
        );

        camera_to_clip * world_to_camera
    }
}

struct Model {
    vertices: Vec<Vec3>,
    instances: Vec<Affine3A>, // modelspace-to-worldspace transformations
}
```

There are a lot of shortcuts and symmetries I haven't covered. For example, when a matrix's basis vectors are all mutually perpendicular and have a length of 1, you can efficiently invert the matrix by [transposing](https://en.wikipedia.org/wiki/Transpose) it (flipping it around the diagonal). And if you have two matrices `M` and `N`, then `M * N = transpose(N) * transpose(M)` so technically applying matrices by multiplying on the left is more of a convention than a necessity.

And linear algebra is useful for _many_ more things than just spatial transformations. The value of each cell of a matrix represents how strongly its column's input contributes to its row's output, so very large matrices are good at representing [how things flow through large networks](https://en.wikipedia.org/wiki/Adjacency_matrix); they can be used to represent how light bounces around a room, how stresses spread through a mechanical structure, how signals propagate through a neural network, or [what portion of time a user would spend on each website if they randomly clicked links](https://en.wikipedia.org/wiki/PageRank). Other uses for matrices center around [solving systems of equations](https://en.wikipedia.org/wiki/System_of_linear_equations). There really is a lot you can do with them!

This has only scratched the surface of linear algebra, but I hope it has provided enough scaffolding to give you a starting point for delving deeper.