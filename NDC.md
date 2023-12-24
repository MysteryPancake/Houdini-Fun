# Houdini Normalized Device Coordinates
NDC is a screen space coordinate system, great for perspective illusions and raycasting tricks.

X and Y represent the 2D screen coordinates, while Z represents the distance to the camera.

<img src="./images/ndccoordinates.png" width="800">

The X and Y coordinates are normalized between 0 and 1, with 0.5 in the middle.

<img src="./images/ndcscreen.png" width="800">

The Z coordinates are 0 at the camera, negative in front and positive behind the camera. Why negative? No idea, it confuses me too.

<img src="./images/ndczaxis.png" width="800">

## Converting NDC
You can convert a world space coordinate to NDC using `toNDC()`:

```js
vector ndcPos = toNDC("/obj/cam1", v@P);
```

Then convert it back to world space using `fromNDC()`:

```js
vector worldPos = fromNDC("/obj/cam1", ndcPos);
```

Here's some fun stuff you can try. [Download the HIP file!](./hips/ndcfun.hipnc?raw=true)

## Get the camera position

The origin of NDC space is the camera, so just convert `{0, 0, 0}` to world space.

<img src="./images/ndccampos.png" width="200" align="left">

```js
string cam = "/obj/cam1";
vector camPos = fromNDC(cam, {0, 0, 0});
addpoint(0, camPos);
```

Use `{0.5, 0.5, 0}` if you're pedantic, but it has the same result.

<br clear="left"/>
