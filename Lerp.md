# Make your own lerp() and fit()
`lerp()` and `fit()` are two functions you'll use constantly. They let you remap the range of a value in a linear way, great for interpolation and extrapolation.

## `lerp()`
```js
float lerp_diy(float value1; float value2; float amount) {
	return (1 - amount) * value1 + amount * value2; // Lerp: Weighted sum (e.g. 25% of value 1, 75% of value 2)
}
```

## `fit()`

`fit()` in Houdini is the same as `linear()` in After Effects.

I remade `linear()` on my [After Effects Fun](https://github.com/MysteryPancake/After-Effects-Fun) page, so let's port it to VEX!

```js
float fit_diy(float value; float omin; float omax; float nmin; float nmax) {
	value = clamp(value, omin, omax);
	float normal = (value - omin) / (omax - omin); // Inverse Lerp: Normalize between 0 and 1 (cannot exceed)
	return (1 - normal) * nmin + normal * nmax; // Lerp: Weighted sum (e.g. 25% of value 1, 75% of value 2)
}
```

```js
// Lerp version
float fit_diy(float value; float omin; float omax; float nmin; float nmax) {
	return lerp(nmin, nmax, invlerp(clamp(value, omin, omax), omin, omax));
}
```

```js
// Imprecise version (rwaldron.github.io/proposal-math-extensions/#sec-math.scale)
float fit_diy(float value; float omin; float omax; float nmin; float nmax) {
	return (clamp(value, omin, omax) - omin) * (nmax - nmin) / (omax - omin) + nmin;
}
```

## `efit()`

```js
float efit_diy(float value; float omin; float omax; float nmin; float nmax) {
	float normal = (value - omin) / (omax - omin); // Inverse Lerp: Normalize between 0 and 1 (can exceed)
	return (1 - normal) * nmin + normal * nmax; // Lerp: Weighted sum (e.g. 25% of value 1, 75% of value 2)
}
```

```js
// Lerp version
float efit_diy(float value; float omin; float omax; float nmin; float nmax) {
	return lerp(nmin, nmax, invlerp(value, omin, omax));
}
```

```js
// Imprecise version (rwaldron.github.io/proposal-math-extensions/#sec-math.scale)
float efit_diy(float value; float omin; float omax; float nmin; float nmax) {
	return (value - omin) * (nmax - nmin) / (omax - omin) + nmin;
}
```

## `fit01()`

```js
float fit01_diy(float value; float nmin; float nmax) {
	float normal = clamp(value, 0, 1); // No inverse lerp needed
	return (1 - normal) * nmin + normal * nmax; // Lerp: Weighted sum (e.g. 25% of value 1, 75% of value 2)
}
```

```js
// Lerp version
float fit01_diy(float value; float nmin; float nmax) {
	return lerp(nmin, nmax, clamp(value, 0, 1));
}
```

```js
// Imprecise version
float fit01_diy(float value; float nmin; float nmax) {
	return nmin + clamp(value, 0, 1) * (nmax - nmin);
}
```

## `fit10()`

```js
float fit10_diy(float value; float nmin; float nmax) {
	float normal = clamp(1 - value, 0, 1); // No inverse lerp needed
	return (1 - normal) * nmin + normal * nmax; // Lerp: Weighted sum (e.g. 25% of value 1, 75% of value 2)
}
```

```js
// Lerp version
float fit10_diy(float value; float nmin; float nmax) {
	return lerp(nmin, nmax, clamp(1 - value, 0, 1));
}
```

```js
// Imprecise version
float fit10_diy(float value; float nmin; float nmax) {
	return nmin + clamp(1 - value, 0, 1) * (nmax - nmin);
}
```

For more info on lerp and fit, be sure to check out [Simon](https://www.youtube.com/watch?v=YJB1QnEmlTs) and [Freya's](https://www.youtube.com/watch?v=aVwxzDHniEw) videos!
