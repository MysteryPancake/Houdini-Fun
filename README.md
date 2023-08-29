# CGWiki DLC
Various Houdini tips and tricks I use a bunch. Hope someone finds this helpful!

## Simple Spring Solver
Need to overshoot an animation or smooth it over time to reduce bumps? Introducing my simple spring solver!

I stole this from an article on [2D wave simulation](https://gamedevelopment.tutsplus.com/make-a-splash-with-dynamic-2d-water-effects--gamedev-236t) by Michael Hoffman. The idea is to set a target position then set the acceleration so it points towards the target. This causes a natural overshoot when the object flies past the target, since the velocity takes time to flip back. Next, you apply damping to stop it going too crazy.

First, add a target position to your geometry:
```c
v@targetP = v@P;
```
Next, add a solver. Inside the solver, add a point wrangle with this VEX;
```c
float freq = 0.3;
float damping = 0.2;

v@accel = (v@targetP - v@P) * freq;
v@v += v@accel;
v@v *= 1 - damping;
v@P += v@v;
```
To smooth motion over time, get the target position from the first input of the solver:
```c
float freq = 0.3;
float damping = 0.2;

v@accel = (v@opinput1_P - v@P) * freq;
v@v += v@accel;
v@v *= 1 - damping;
v@P += v@v;
```

## Smoke / Fluids: Fix moving colliders
Fluids often screw up whenever colliders move, like water in a moving cup or smoke in an elevator. Either the collider deletes the volume as it moves, or velocity doesn't transfer properly from the collider.

A great fix comes from Raph Gadot: Stabilize the collider, freeze it in place. Simulate in local space, apply forces in relative space, then invert back to world space.

This works best for enclosed containers or pinned geometry, since it's hard to mix local and world sims. Vellum Reference Frame is probably a better choice for cloth.

**Relative gravity**
1. Add an `@up` vector in world space (before Transform Pieces).
```js
v@up = {0, 1, 0};
```
2. Add a Gravity Force node to your sim (after Transform Pieces). Use the transformed `@up` vector as the gravity force.

```js
Force X = -9.81 * point(-1, 0, "up", 0)
Force Y = -9.81 * point(-1, 0, "up", 1)
Force Z = -9.81 * point(-1, 0, "up", 2)
```
Make sure the force is "Set Always"!

**Relative acceleration**
1. Add a Trail node set to "Calculate Velocity", then enable "Calculate Acceleration". It's faster to do this after packing so it only trails one point.

2. Add another Gravity Force node, using negative `@accel` as the force vector.

```js
Force X = -point(-1, 0, "accel", 0)
Force Y = -point(-1, 0, "accel", 1)
Force Z = -point(-1, 0, "accel", 2)
```
Make sure the force is "Set Always"!

**Stabilise (world to local)**
1. Pick a face on the collider you want to stabilise. Blast everything except that face.
2. Time freeze that face with a Time Shift node.
3. Use an Extract Transform node to compare the frozen face to the moving face. That tells you how the collider moves over time, allowing you to cancel out the movement.
4. Pack everything else. Make sure to enable "No Point Velocities".
5. Plug the Pack into a Transform Pieces node, then plug Extract Transform into the other input.
6. You should see the collider frozen in place. If not, try swapping the Time Shift to the other input in your Extract Transform node.
7. Unpack and do your sim in local space.
8. Pack the sim result.
9. Add another Transform Pieces node with the same Extract Transform input. This time, set it to "Invert Transform" to go back to world space.
10. Unpack the world space result.

If you want to deal with open containers, the easiest way is to do a separate sim when the fluid exits the container. This is done by killing points outside the container, then feeding the killed points into the other sim. Make sure to nuke all point attributes to keep it clean for the next sim.

Another tip is use "Central Difference" when calculating the velocity. This gives the fluid more time to move away from the collider.

## Cloth: Fix preroll with FBX
Cloth sims work best with preroll starting in a neutral rest pose. For example, the character starts in an A-pose or T-pose before transitioning into the animation. 

If anim screwed you over, never fear! Preroll can be added in Houdini.

1. Export the animated character as FBX. Make sure to include the skeleton!
2. Import the character with a FBX Character Import node.
3. Use a Skeleton Blend node to blend from the rest skeleton to the animated skeleton. If the rest skeleton has a bad pose, fix it with the Rig Pose node. Alternatively, export another FBX posed to your liking. FBX Character Import that animated skeleton as the rest skeleton.
4. Use the Time Shift node to move the animation forward so it doesn't bleed into the preroll. This can also be done in the "Timing" menu of FBX Character Import.
5. Use Bone Deform to animate the skin based on Skeleton Blend.

## Cloth: Fix preroll without FBX
Sometimes you can't get an FBX of the character. This makes it harder to get a clean blend, but there are still some options.

One option is using a Blend Shapes node, but you'll find the limbs usually clip through the body as it swaps from the T-pose to the animated pose.

My sketchy method of improving this is using Extract Transform and Transform Pieces. Use blast to isolate a face from the character's chest. Next, use Extract Transform and Transform Pieces to transform the T-pose to match the animated pose. In other words, the T-posed character flies over to the animated position and rotates to match it. That gives you an easier time using Blend Shapes, since it only has to move the arms and legs a short distance to match the animated pose.

## Cloth: Fix rest pose clipping
Cloth sims screw up from clipping, especially when clipped from the start. One option is growing the character into the cloth. 

1. Disable gravity in the cloth sim.
2. Use Smooth, Peak or VDB Reshape to shrink the character.
3. Animate the character growing.
4. Enable gravity in the cloth sim.
5. Use the Time Shift node to move the animation forward so it doesn't bleed into the growing. 

## Cloth: Layer stacking
One little known feature of Vellum Cloth (at least to me) is layering. It can improve the physics of overlapping garments, like jackets on top of t-shirts. 

1. In Vellum Configure Cloth, use the "Layer" setting to define the ordering, bottom to top.
2. On the Vellum Solver under "Collisions", enable "Layer Shock". Lower layers are simulated much heavier than higher layers. 

## Karma: Fix motion blur
Motion blur in Karma rarely works properly out of the box, even with manual velocity vectors.

A great fix comes from [CGWiki](https://www.tokeru.com/cgwiki/index.php?title=UsdGuide18): simply add a Cache node set to "Rolling Window". Usually I use 1 frame before and 1 frame after.

This works faster than Karma's motion blur LOP, which caches the entire timeline at once.

## Attribute min/max/average...
Use Attribute Promote set to "Detail" with the appropriate mode.

## Access context geometry inside solver
If you need geometry in a context that doesn't provide it (like the forces of a Vellum Solver), just drop down a SOP Solver. You can use Object Merge inside a SOP Solver to get geometry from anywhere else too. Create feedback loops until your heart's content!

## Pyro: Fix mushrooms
Many techniques work depending on the situation. Sometimes more randomization is needed, other times the velocity needs reducing.

A common technique is cranking up the disturbance. Controlling it by speed helps add it where mushrooms are likely to form.

## Fluids: Fix density loss 
Don't take this section seriously. These are just techniques which seem to work for me. 

Density loss often happens when Surface Tension is enabled. Droplets tend to disappear when bunched too close together, so try disabling it before anything else. 

Grid Scale and Particle Radius Scale also affect the density. It seems to oversmooth sometimes, leading to particles being smoothed out of existence. 

According to [SideFX](https://www.sidefx.com/docs/houdini/nodes/dop/flipsolver.html), if the Particle Radius Scale divided by the Grid Scale is at least sqrt(3)/2, it will never be underresolved. 

In case this affects density, to find the minimum: 
```js
Particle Radius Scale = Grid Scale × (sqrt(3)/2)
Grid Scale = Particle Radius Scale / (sqrt(3)/2)
```
