# Vexember 2023
Vexember is a Houdini challenge created by [Paul Esteves](https://www.youtube.com/watch?v=uxevpt_xK68) and some other legends on the CGWiki Discord.

It's covered tons of cool topics so far, here's a writeup on everything I know at the moment. Whether new or old to Houdini, I hope you learn something new!

## Day 1: SDFs
<img src="./images/vexember1.gif" height="320">

The first challenge focuses on Signed Distance Functions (SDFs). So what the hell is an SDF? Great question!

Traditionally, geometry is made up of thousands of little triangles. Triangles are great for sharp pointy objects like pyramids, but if you want a nice smooth object like a sphere, you're out of luck. You'd need infinite triangles to perfectly represent a sphere.

<img src="./images/vexembertriangles.png" height="320">

If only there was a better way. Turns out there is! With signed distance functions, you get a [perfect sphere out of the box](https://www.shadertoy.com/view/3ltSW2).

Let's start by making your first SDF, the SDF of a point.

### SDF of a point

Drop down a grid in Houdini, and set the rows and columns to a large number. Next add a point wrangle. We want to find the distance from our current position to another position. Let's use the world origin `{0, 0, 0}`:

```c
vector p2 = {0, 0, 0};
v@Cd = distance(v@P, p2);
```

You should see a blurred black circle on a white background:

<img src="./images/vexemberdist.png" height="320">

Congratulations, you made your first distance function!

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
