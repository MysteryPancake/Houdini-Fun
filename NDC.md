# Houdini Normalized Device Coordinates
NDC is a screen space coordinate system, great for perspective illusions and raycasting tricks.

X and Y represent the 2D screen coordinates, while Z represents the distance to the camera.

<img src="./images/ndccoordinates.png" width="800">

The X and Y coordinates are normalized between 0 and 1, with 0.5 in the middle.

<img src="./images/ndcscreen.png" width="800">

The Z coordinates are 0 at the camera, negative in front and positive behind the camera. Why negative? No idea, it confuses me too.

<img src="./images/ndczaxis.png" width="800">
