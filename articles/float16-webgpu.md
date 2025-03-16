## [Playing tricks on Javascript : float16 matrix multiplication in WebGPU]()

Quarterway through my life’s journey, I went astray from the straight road and awoke to find myself alone in a dark wood, trying to render a large RAW image in WebGPU. I thought I found salvation once I learned that the WebGPU Shading Language (WGSL) committee had approved the ```f16``` type, but little did I know this would only be the beginning of my torment.  

RAW images have a dynamic range that extends beyond the traditionnal 8 bit, generally 16 bit unsigned integers, and thus allow for rich photo editing. Parsing a RAW image, passing it to the GPU and rendering it is usually pretty straigthforward to do in a native app. But since WebGPU is a relatively new technology, it doesn't currently offer this flexibility and rendering 16 bit images in the browser presents some challenges.

#### Whats's currently possible ?

The first solution I implemented was just to parse the RAW image in uint16 and cast every value to float32. It takes double the size and it's slow, but it works. Rendering usually happens in three of four passes. Here are the steps :

1. The RAW image is first decoded into a Javascript ```Uint16Array```.
   
2. The array is cast into a ```Float32Array```, so every value is converted to its 32bit floating point representatation (modern Javascript is JIT-compiled and incredibly optimized so a simple for loop shall do).

```js
let rawImage = new Uint16Array([0, 1, 2, 3, 4, 65535]);
let f32RawImage = new Float32Array(rawImage.length);

for (let i = 0; i < f32RawImage.length; i++) {
    f32RawImage[i] = rawImage[i];
}
```

3. This new ```Float32Array``` is then passed to the GPU buffer, which maps nicely to the ```array<f32>``` WGSL type.

**Why not cast to a ```Uint8Array```?** : even if the values of the image (and every image) are squashed to an 8 bit dynamic range when displayed on a screen, we need to keep the full 16 bit dynamic for image processing, before any rendering on a screen happens.

**Why not just keep the ```Uint16Array```?** : WebGPU has no compatible integer type (so ```u16``` doesn't exist in WGSL). 

<br/>
