# Houdini Extract Transform Remake

Ever wondered how Extract Transform works? Turns out it's based on [the Umeyama algorithm](https://nghiaho.com/?page_id=671).

The algorithm works like this:

1. Find the best translation by subtracting centroids
2. Find the best rotation using [Singular Value Decomposition (SVD)](https://www.sidefx.com/docs/houdini/vex/functions/svddecomp.html)
3. Find the best scale using [the Umeyama algorithm](https://nghiaho.com/?page_id=671).

| [Download the HIP file!](./hips/extract_transform_svd.hiplc?raw=true) |
| --- |

## Align Translation

Aligning the translation is easy. The best translation happens when you align the center of mass (average) of each point cloud.

You don't even need VEX for this, just use Extract Centroid set to "Center of Mass", then offset the position by that amount.

```js
// Detail Wrangle: Solves translation only
// Input 1: Source
// Input 2: Target

// 1. Calculate centroids
float n = npoints(1);
vector source_centroid = 0, target_centroid = 0;
for (int i = 0; i < n; ++i) {
    source_centroid += point(1, "P", i);
    target_centroid += point(2, "P", i);
}

// 2. Turn translation into 4x4 matrix
matrix transform = ident();
translate(transform, (target_centroid - source_centroid) / n);

// 3. Add point for Transform Pieces to use
setpointattrib(0, "transform", addpoint(0, {0, 0, 0}), transform);
```

## Align Translation + Rotation

Aligning the rotation is harder. You need to build a covariance matrix, then solve it with SVD.

Luckily we don't need to leave VEX! Houdini has a SVD solver called `svddecomp()`.

```js
// Detail Wrangle: Solves translation and rotation only
// Input 1: Source
// Input 2: Target

// 1. Calculate centroids
float n = npoints(1);
vector source_centroid = 0, target_centroid = 0;
for (int i = 0; i < n; ++i) {
    source_centroid += point(1, "P", i);
    target_centroid += point(2, "P", i);
}
target_centroid /= n;
source_centroid /= n;

// 2. Build covariance matrix
matrix3 covariance = ident();
for (int i = 0; i < n; ++i) {
    vector source_diff = point(1, "P", i) - source_centroid;
    vector target_diff = point(2, "P", i) - target_centroid;
    covariance += outerproduct(target_diff, source_diff);
}

// 3. Solve rotation with SVD
matrix3 U;
vector S;
matrix3 V;
svddecomp(covariance, U, S, V);
matrix3 R = V * transpose(U);

// 4. Flip if determinant is negative (this causes negative scales to screw up)
if (determinant(R) < 0) {
    R = V * diag({1, 1, -1}) * transpose(U);
}

// 5. Combine translation and rotation into 4x4 matrix
matrix transform = set(R);
translate(transform, target_centroid - source_centroid);

// 6. Add point for Transform Pieces to use
setpointattrib(0, "transform", addpoint(0, {0, 0, 0}), transform);
```

## Align Translation + Rotation + Scale

Aligning the scale is even harder. One popular way is the [Umeyama algorithm](https://stackoverflow.com/a/32244818).

Sadly it only solves uniform scale, and breaks on negative scales. This happens when you flip the sign of the 2nd column of the rotation matrix.

Extract Transform set to "Uniform Scale" also breaks with negative scales, so it probably uses this method!

```js
// Detail Wrangle: Solves translation, rotation, uniform scale (like Eigen::umeyama)
// Input 1: Source
// Input 2: Target

// 1. Calculate centroids
float n = npoints(1);
vector source_centroid = 0, target_centroid = 0;
for (int i = 0; i < n; ++i) {
    source_centroid += point(1, "P", i);
    target_centroid += point(2, "P", i);
}
target_centroid /= n;
source_centroid /= n;

// 2. Build covariance matrix
float deviation = 0;
matrix3 covariance = ident();
for (int i = 0; i < n; ++i) {
    vector source_diff = point(1, "P", i) - source_centroid;
    vector target_diff = point(2, "P", i) - target_centroid;
    deviation += length2(source_diff);
    covariance += outerproduct(target_diff, source_diff);
}

// 3. Solve rotation with SVD
matrix3 U;
vector S;
matrix3 V;
svddecomp(covariance, U, S, V);
matrix3 R = V * transpose(U);

// 4. Flip if determinant is negative (this causes negative scales to screw up)
float det = determinant(R);
vector e = set(1, 1, det < 0 ? -1 : 1);
if (det < 0) {
    R = V * diag(e) * transpose(U);
}

// 5. Solve scale using standard deviation
R *= dot(S, e) / deviation;

// 6. Combine translation rotation and scale into 4x4 matrix
matrix transform = set(R);
translate(transform, target_centroid - (R * source_centroid));

// 7. Add point for Transform Pieces to use
setpointattrib(0, "transform", addpoint(0, {0, 0, 0}), transform);
```
