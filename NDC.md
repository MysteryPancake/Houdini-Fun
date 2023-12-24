# Houdini Normalized Device Coordinates
NDC is a screen space coordinate system, great for perspective illusions and raycasting tricks.

X and Y represent the 2D screen coordinates, while Z represents the distance to the camera.

<img src="./images/ndccoordinates.png" width="700">

The X and Y coordinates are normalized between 0 and 1, with 0.5 in the middle.

<img src="./images/ndcscreen.png" width="700">

The Z coordinates are 0 at the camera, negative in front and positive behind the camera. Why negative? No idea!

<img src="./images/ndczaxis.png" width="700">

## Converting NDC
You can convert a world space coordinate to NDC using `toNDC()`:

```js
vector ndcPos = toNDC("/obj/cam1", v@P);
```

Then convert it back to world space using `fromNDC()`:

```js
vector worldPos = fromNDC("/obj/cam1", ndcPos);
```

Here's some fun NDC tricks you can play with. [Download the HIP file!](./hips/ndcfun.hipnc?raw=true)

## Get the camera position

The origin of NDC space is the camera, so just convert `{0, 0, 0}` to world space.

<img src="./images/ndccampos.png" width="250" align="left">

```js
string cam = "/obj/cam1";
vector camPos = fromNDC(cam, {0, 0, 0});
addpoint(0, camPos);
```

`{0.5, 0.5, 0}` is technically more correct, but gives the same result.

<br clear="left"/>

## Draw a ray from the camera

The Z axis aligns with the camera direction, so move along it to draw a ray.

<img src="./images/ndccamline.gif" width="250" align="left">

```js
string cam = chs("cam");
float offset = chf("raylength");

// Sample two positions along the Z axis in NDC space to draw a ray
vector camPos = fromNDC(cam, {0.5, 0.5, 0});
vector camPos2 = fromNDC(cam, set(0.5, 0.5, -offset));

int a = addpoint(0, camPos);
int b = addpoint(0, camPos2);

addprim(0, "polyline", a, b);
```

<br clear="left"/>

## Flatten to the XY plane

Using NDC coordinates directly in world space distorts the geometry to how it looks on screen, like a printed photo.

<img src="./images/ndcflat.png" width="250" align="left">

```js
string cam = chs("cam");

// Flatten by setting Z to a constant value
v@P = toNDC(cam, v@P);
v@P.z = 0;
```

<br clear="left"/>

<img src="./images/ndcsilouette.png" width="250" align="left">

You can make a silhouette from this pretty easily. TODO
