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

<br>

**Why not just keep the ```Uint16Array``` ?** : WebGPU has no compatible integer type (so ```u16``` doesn't exist in WGSL). 

**Why not cast to a ```Uint8Array``` ?** : even if the values of the image (and every image) are squashed to an 8 bit dynamic range when displayed on a screen, we need to keep the full 16 bit dynamic for image processing on the GPU, before any rendering on a screen happens.

**Why not cast to a ```Float16Array``` ?** : Great question ! The ```Float16Array``` is [a valid Javascript type](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Float16Array) however it is still not implemented in Google Chrome and Chromium based browser (Edge, Opera, etc.). It is only implemented inside Safari and Firefox at the moment. 

As for WebGPU, it's currently only supported inside Chromium based browsers, not supported inside Firefox, and has beta support inside Safari.

Go figure 🙃

That means that to be able to pass 16 bit floating point values to WebGPU as an ```array<f16>```, we're gonna have to roll out our own ```Float16Array``` type that extends the regular ```Uint16Array``` type and that we'll soberly call ```Float16ArrayLike```.

The rest of this article will go over the implementation ```Float16ArrayLike``` in Javascript and we'll test it with a simple 4x4 matrix multiplication inside a WebGPU compute shader. It should map nicely with the WGSL  ```array<f16>``` type.

I expect this article to be made redundant once Chrome rolls out the ```Float16Array``` type, but this is still a fun exercise to do in the meantime anyway.

#### The ```Float16ArrayLike``` type

The ```Float16ArrayLike``` type that we're implementing : 

1. extends (inherits from) the ```Uint16Array``` type, since they both occupy a 16 bit memory space per value.
2. "overrides" the bracket ```[]``` operator, so that we catch every value that is set / get and casts them appropriately to a 16 bit floating point value.

Operator overridding does not exist in Javascript, but in the case of a ```TypedArray```, we can catch calls to the bracket operators using a ```Proxy```.

Here's what it looks like.

```js
export class Float16ArrayLike extends Uint16Array {
  constructor(length) {
    super(length);
    return new Proxy(this, {
      get: (target, prop, receiver) => {
        const index = Number(prop);
        if (!Number.isNaN(index) && Number.isInteger(index)) {
          return castFloat16ToFloat32(target[index]);
        }
        if (prop == "byteLength") {
          return target.byteLength
        }
        if (prop == "buffer") {
          return target.buffer
        }
        if (prop == "byteOffset") {
          return target.byteOffset
        }
        if (prop == "length") {
          return target.length
        }
      },
      set: (target, prop, value, receiver) => {
        const index = Number(prop);
        if (!Number.isNaN(index) && Number.isInteger(index)) {
          return Reflect.set(target, index, castFloat32ToFloat16(Number(value)), receiver);
        }
      }
    });
  }
}
```
 
 Notice we use two util functions : ```castFloat32ToFloat16``` and ```castFloat16ToFloat32```. Since the function names are pretty explicit, I will spare the reader of having to read through these eyesores, but the interested _afficionado_ can find the [full implementation on Github](https://github.com/dany-demise/dany-demise.github.io/blob/main/float16-webgpu/assets/float16-array.js).

 #### Passing the values to WebGPU
 Our ```Float16ArrayLike``` type maps nicely to the ```array<f16>``` WGSL type. Here's a compute shader for a simple 4 x 4 matrix multiplication.

 ```wgsl
 enable f16;
@group(0) @binding(0) var<storage, read> matrixA : array<f16>;
@group(0) @binding(1) var<storage, read> matrixB : array<f16>;
@group(0) @binding(2) var<storage, read_write> matrixC : array<f16>;

@compute @workgroup_size(4, 4, 1)
fn main(@builtin(global_invocation_id) gid : vec3<u32>) {
    let row : u32 = gid.y;
    let col : u32 = gid.x;
    let index : u32 = row * 4u + col;
    var sum : f16 = 0.0;
    for (var k : u32 = 0u; k < 4u; k = k + 1u) {
        sum = sum + matrixA[row * 4u + k] * matrixB[k * 4u + col];
    }
    matrixC[index] = sum;
}
 ```

 Notice the ```enable f16;``` on top and the fact that both our read matrices A and B and our written to matrix C are ```array<f16>```. I will again spare all the Javascript to WebGPU pipelining code for the reader, but know that you just need to treat a ```Float16ArrayLike``` exactly as you would a regular ```Float32Array```, as both are ```TypedArray```s.

### Try it out

 
 <br/>
 <br/>
