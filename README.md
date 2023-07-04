# CGWiki DLC
Various tips and tricks I learnt at UTS Animal Logic Academy and beyond. Hope someone finds this helpful!

## Smoke / Fluids: Fix moving colliders
Fluids often screw up whenever colliders move, for example water in a moving cup or smoke in an elevator. Either the collider deletes the volume as it moves, or velocity doesn't transfer from the collider.

A great fix comes from Raphael Gadot: stabilise the environment around the collider. This means the sim is done in local space with the collider fixed in place, then inverted back to world space. Raphael stresses this isn't a proper technique, but it works well in my tests.

For gravity:
1. Add `@orient` and `@up` vectors in world space (before Transform Pieces).
```js
v@up = {0, 1, 0};
p@orient = {0, 0, 0, 1};
```
2. Add a Gravity Force node to your sim (after Transform Pieces). Use the transformed `@up` vector as the gravity force.

<img src="images/movingcontainers1.PNG"/>

```js
Force X = -9.81 * point(-1, 0, "up", 0)
Force Y = -9.81 * point(-1, 0, "up", 1)
Force Z = -9.81 * point(-1, 0, "up", 2)
```
Make sure the Gravity Force is set to "Calculate Always" since the gravity always changes.

For acceleration:
1. Add a Trail node set to "Calculate Velocity", then enable "Calculate Acceleration". It's faster to do this after packing so it only trails one point. Make sure to trail in the correct reference frame.

2. Add another Gravity Force node, using negative `@accel` as the force vector.

<img src="images/movingcontainers2.PNG"/>

```js
Force X = -point(-1, 0, "accel", 0)
Force Y = -point(-1, 0, "accel", 1)
Force Z = -point(-1, 0, "accel", 2)
```
Make sure the Gravity Force is set to "Calculate Always", since the acceleration always changes.

For stabilisation:
1. Pick a face on the collider you want to stabilise. Blast everything except that face.
2. Use an Extract Transform node to find where it moves over time.
3. Pack everything else. Make sure to enable "No Point Velocities" in case it screws with our trail.
4. Plug the packed geometry into a Transform Pieces node, then plug the Extract Transform result into the other input.
5. It should be stabilized. Unpack and do your sim with the gravity forces described above.
6. Pack the simulation result.
7. Use a Transform Pieces node with the same Extract Transform input. This time, set it to "Invert Transform" to go back to world space.
8. Unpack the world space result.

This works best for enclosed containers. When fluid exits the container, you have to do a separate sim in the new reference frame. This is possible by killing points outside the container, then feeding the killed points into the other sim.

Another tip is using "Central Difference" when trailing. This gives the fluid more time to move away from the collider, and helps calculate motion blur.

## Karma: Fix motion blur
Motion blur in Karma rarely works properly out of the box, even with manual velocity vectors.

A great fix comes from [CGWiki](https://www.tokeru.com/cgwiki/index.php?title=UsdGuide18): simply add a Cache node set to "Rolling Window". Usually I use 1 frame before and 1 frame after.

This works faster than Karma's motion blur LOP, which caches the entire timeline at once.

# Fluids: Fix density loss 

Don't take this section seriously. These are just techniques which seem to work for me. 

Density loss often happens when Surface Tension is enabled. Droplets tend to disappear when bunched too close together, so try disabling it before anything else. 

Grid Scale and Particle Radius Scale also affect the density. It seems to oversmooth sometimes, leading to particles being smoothed out of existence. 

According to [SideFX](https://www.sidefx.com/docs/houdini/nodes/dop/flipsolver.html), if the Particle Radius Scale divided by the Grid Scale is at least sqrt(3)/2, it will never be underresolved. 

In case this affects density, to find the minimum: 
```js
Particle Radius Scale = Grid Scale × (sqrt(3)/2)
Grid Scale = Particle Radius Scale / (sqrt(3)/2)
```
