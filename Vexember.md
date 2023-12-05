# Vexember 2023
Vexember is a Houdini challenge created by [Paul Esteves](https://www.youtube.com/watch?v=uxevpt_xK68) and some other legends on the CGWiki Discord.

It's covered tons of cool topics so far, here's a writeup on everything I know at the moment. Whether new or old to Houdini, I hope you learn something new!

## Day 1: SDFs
The first challenge focuses on Signed Distance Functions (SDFs). So what the hell is an SDF? Great question!

Traditionally, geometry is made up of thousands of little triangles. Triangles are great for sharp pointy objects like pyramids, but if you want a nice smooth object like a sphere, you're out of luck. You'd need infinite triangles to perfectly represent a sphere.

<img src="./images/vexembertriangles.png" height="320">

If only there was a better way. Turns out there is! With SDFs, you get a [perfect sphere out of the box](https://www.shadertoy.com/view/3ltSW2).

Let's start by making your first SDF, the SDF of a point.

### SDF of a point

Drop down a grid in Houdini, and set the rows and columns to a large number. Next add a point wrangle. We want to find the distance from our current position to another point. Let's use the world origin `{0, 0, 0}`:

```c
v@Cd = distance(v@P, {0, 0, 0});
```

You should see a blurred black circle on a white background:

<img src="./images/vexemberdist.png" height="320">

Congratulations, you made your first SDF! Wasn't too hard, was it? The black part is where the distance is smallest, and it gets brighter as the distance grows. It's only 2 lines of VEX, but let's see if we can make it even shorter.

Surprisingly, most SDFs (like [the classics by Inigo Quilez](https://iquilezles.org/articles/distfunctions2d/)) don't use `distance()`. Instead they use `length()`.

`length()` gets the magnitude of a vector, meaning how far it is from `{0, 0, 0}`. This is exactly what we were doing before:

```c
// Using length()
v@Cd = length(v@P);

// Using distance()
v@Cd = distance(v@P, {0, 0, 0});
```

Now we're down to a single line of VEX! What if we want to move our point to a different location? Turns out `length()` works for that too!

```c
// Move 1 unit along the X axis
vector p2 = {1, 0, 0};

// Using length()
v@Cd = length(v@P - p2);

// Using distance()
v@Cd = distance(v@P, p2);
```

You can think of this as centering the geometry before measuring, or moving the camera versus moving the object.

Now let's make your second SDF, the SDF of a circle.

### SDF of a circle
Let's start from the SDF of a point.

```c
v@Cd = length(v@P);
```

We know the distance grows in a circle away from `{0, 0, 0}`, so let's shade only the area above a threshold, for example 0.5:

```c
v@Cd = length(v@P) > 0.5;
```

<img src="./images/vexemberdist2.png" height="320">

```c
// Get the position of the other point (wrangle input 1).
vector p2 = point(1, "P", 0);

// Get the distance from us to the other point. This is the signed distance function of a point.
float sdf = distance(v@P, p2);

// We can visualise the SDFs using a sine wave. This is commonly found on ShaderToy
float frequency = 50; // Distance between peaks in the wave
float phase = @Time * -10; // How fast it wobbles over time
float wave = sin(sdf * frequency + phase);

// Sine waves range from -1 to 1, so remap it to 0 to 1 since RGB is 0 to 1
wave = fit11(wave, 0, 1); // Or wave * 0.5 + 0.5

v@Cd = wave; // Color based on the wave
v@P += v@N * wave * 0.02; // Peak based on the wave. 0.02 is the distance to peak
```
<img src="./images/vexember1.gif" height="320">
