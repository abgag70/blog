## [Playing tricks on Javascript : float16 matrix multiplication in WebGPU]()

Quarterway through my life’s journey, I went astray from the straight road and awoke to find myself alone in a dark wood, trying to render a large RAW image in WebGPU. I thought I found salvation once I learned that the WebGPU Shading Language (WGSL) committee had approved the ```f16``` type, but little did I know this would only be the beginning of my torment.  

RAW images have a dynamic range that extends beyond the traditionnal 8 bit, generally 16 bit unsigned integers, and thus allow for rich photo editing. Parsing a RAW image, passing it to the GPU and rendering it is usually pretty straigthforward to do in a native app. But since WebGPU is a relatively new technology, it doesn't currently offer this flexibility and rendering 16 bit images in the browser presents some challenges.

#### Whats's currently possible ?

The first solution I implemented was just to parse the RAW image in uint16 and cast every value to float32. It takes double the size and it's slow, but it works. Rendering usually happens in three of four passes 