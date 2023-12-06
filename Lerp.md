# Houdini Lerp and Fit
`lerp()` and `fit()` are two functions you'll use constantly. They let you remap a value linearly, great for interpolation and extrapolation.

If you've ever wondered how they work, look no further! I remade them on my [After Effects Fun](https://github.com/MysteryPancake/After-Effects-Fun) page, so let's port them to VEX!

For a great resource on deriving them yourself, check out [Simon](https://www.youtube.com/watch?v=YJB1QnEmlTs) and [Freya's](https://www.youtube.com/watch?v=aVwxzDHniEw) videos!

## `lerp()`
Lerp takes two values and performs a weighted sum based on a factor.

For example, to get 50% between two values, you'd add half of each:

```js
0.5 * value1 + 0.5 * value2
```

To get 75% between two values, you'd add 1/4 of the first and 3/4 of the second:

```js
0.25 * value1 + 0.75 * value2
```

Generalising this idea gives us the formula for lerp:

```js
float lerp_diy(float value1; float value2; float amount) {
	return (1 - amount) * value1 + amount * value2; // Lerp: Weighted sum (e.g. 25% of value 1, 75% of value 2)
}
```

## `invlerp()`
Inverse lerp takes a value and a range, then normalizes the value so it lies between 0 and 1.

Given a minimum and maximum value, the range is `max - min`. To normalize a value we can divide it by the range.

```js
float invlerp_diy(float a; float min; float max) {
	return (a - min) / (max - min); // Inverse Lerp: Normalize between 0 and 1 (cannot exceed)
}
```

## `fit()`
`fit()` is the combination of `invlerp()` and `lerp()`.

It can interpolate but can't extrapolate, because `value` is clamped between `omin` and `omax`.

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
`efit()` is the same as `fit()`, except `value` isn't clamped.

This means it can interpolate and extrapolate `value` outside of `omin` and `omax`.

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
`fit01()` is the same as `fit()` except the range is hardcoded as 0 to 1. It's the same as `lerp()` except `value` is clamped.

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
`fit01()` is the same as `fit()` except the range is hardcoded as 1 to 0.

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
