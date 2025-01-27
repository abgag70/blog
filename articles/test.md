## [Compiling DLib's MaxLIPO+TR global optimization algorithm to WASM]()
<sub>January 18th 2024 </sub>
<br>

A few while ago I came across a great [post](https://blog.dlib.net/2017/12/a-global-optimization-algorithm-worth.html) shared by the great [@sasuke___420](https://x.com/sasuke___420) regarding the MaxLIPO+TR algorithm implemented inside DLib's library. I was really excited the first time I read this post. I was awestruck by the sheer power of the algorithm and yet how simple it was.

If you're not familiar with it, it is implemented through the find_min_global function of Dlib and allows the programmer to minimize an objective function. It requires no hyperparameters and only takes in the function to be minimized, upper and lower bounds and a maximum number of function calls.

Building on the example in the article above, we can test the find_min_global function in Python with the following code. Dlib is installed by default in Colab so you can try it out quickly.

```python
import dlib

# Is minimized at x0 = 1, x1 = 1
def rosenbrock(x0, x1): 
    return (1 - x0)**2 + 100*(x1 - x0**2)**2

x,y = dlib.find_min_global(
    rosenbrock, 
    [-10,-10],  # Lower bound constraints on x0 and x1 respectively
    [10,10],    # Upper bound constraints on x0 and x1 respectively
    150        # Number of function calls (epochs)
)
```

This gives the following on my platform 
­­­
```
 x = [0.9999999994699013, 0.999999999065295]
y = 1.8558415650416274e-18
```
<br>

#### Interfacing with Javascript

Since the goal is to reproduce this behavior in Javascript, we start by creating a function called maxLipoPlusTr that will be able to minimize any objective Javascript function that returns a numerical value. Here's what we want it to look like.

```js
import { maxLipoPlusTr } from "./max-lipo-plus-tr.js"

// Rosenbrock function for 3D space, is minimized at x0 = 1, x1 = 1, x2 = 1
function rosenbrock3D(x0, x1, x2) {
    return 100 * ((x1 - x0**2)**2 + (x2 - x1**2)**2) + (1 - x0)**2 + (1 - x1)**2;
}

const lower_bounds = [-100, -100, -100];
const upper_bounds = [100, 100, 100];
const max_calls = 300;

let result = await maxLipoPlusTr(rosenbrock3D, lower_bounds, upper_bounds, max_calls);

console.log(result);
// This gives :
// {
//   "x": [0.9999998222686982, 0.999999575414733, 0.9999992156860394], 
//   "y": 0.00000000000091659844
// }
```

Under the hood, maxLipoPlusTr loads the WASM module, wraps the input function inside an object and sends it to the WASM environnement to be minimized inside DLib's find_min_global function.

Here's ```max-lipo-plus-tr.js```

```js
import createMaxLipoTrPlusModule from './find_min_global_o3.js'

var Module = null;

export async function maxLipoPlusTr(theFunction,
                                    lower_bounds,
                                    upper_bounds,
                                    max_calls) { 

    if (!Module) { // load Module if not loaded yet
        Module = await createMaxLipoTrPlusModule();
    }

    // Create an instance of JsFunction with the Rosenbrock function
    const jsFunc = new JsFunction(theFunction);

    const result = await Module.max_lipo_plus_tr(jsFunc, lower_bounds, upper_bounds, max_calls);

    return { x: result.x, y: result.y };
}
```

The ```JsFunction``` object wraps up the function to be minimized.

```js
class JsFunction {
    constructor(func) {
        this.func = func;
        this.args = new Float32Array(func.length);
    }
    bang() {
        return this.func(...this.args); // unpack args Array and call function
    }
    setArg(i, arg) {
        this.args[i] = arg;
    }
}
```
The ```setArg``` method, called from the C++ code using Emscripten, allows us to rapidly change the arguments of the function during the optimization process, avoiding unnecessary memory allocations and the need to create an array each time the function is called. Plus, since we know all our values will be of float 32 type, we can use a ```FLoat32Array``` created with a fixed length. This allows us to benefit from the fact that a Javascript ```TypedArray``` uses contiguous memory allocation by default.


#### Compiling to WASM

Dlib is written in C++ and offers a Python API. I've been f-ing around WASM and Emscripten lately and wanted a small and interesting project to learn so I decided to give a shot at compiling Dlib to WASM.

First make sure you have [Emscripten installed](https://emscripten.org/docs/getting_started/downloads.html) with.

```bash
emcc --version
```

⚠️ I'm currently using Emscripten 3.1.74 because of a [compiling issue](https://github.com/davisking/dlib/issues/3045) with version 4.0.0.

Let's start by cloning the latest DLib and cd-ing into the directory

```bash
git clone https://github.com/davisking/dlib.git
cd dlib
```

The real challenge was porting a Javascript function to Emscripten to be minimized from there. For this I wrote a C++ wrapper that can interface with a Javascript function inside the environnement through Emscripten.

Create this **find_min_global_wrapper.cpp** file inside the dlib/ folder.

```cpp
#include <dlib/global_optimization.h>
#include <dlib/matrix.h>
#include <emscripten/bind.h>
#include <emscripten/val.h>
#include <functional>

using namespace emscripten;
using namespace dlib;

// Convert a dlib matrix to a Javascript Array object
val dlib_mat_to_js_array(const dlib::matrix<double, 0, 1>& vec) {
    val js_array = val::array();
    for (std::size_t i = 0; i < vec.nr(); ++i) {
        js_array.call<void>("push", vec(i));  // Push each element into the js array
    }
    return js_array;
}


// Convert a Javascript Array object to a dlib matrix
dlib::matrix<double, 0, 1> js_array_to_dlib_mat(const val& js_array) {
    dlib::matrix<double, 0, 1> vec(js_array["length"].as<std::size_t>());
    for (std::size_t i = 0; i < vec.nr(); ++i) {
        vec(i) = js_array[i].as<double>();  // Get each element from the JS array and assign to matrix
    }
    return vec;
}


// Simplified wrapper for vector functions with max_function_calls
val max_lipo_plus_tr(val jsFunction,
                     val lower_bounds, // as Javascript Arrays
                     val upper_bounds,
                     size_t max_calls) {
    std::function<double(const dlib::matrix<double, 0, 1>&)> func;

    func = [jsFunction](const dlib::matrix<double, 0, 1>& vec) -> double {
        val args_js_array = dlib_mat_to_js_array(vec);
        return jsFunction.call<double>("bang", args_js_array);
    };

    // Convert vector bounds to dlib matrices
    dlib::matrix<double, 0, 1> lb = js_array_to_dlib_mat(lower_bounds);
    dlib::matrix<double, 0, 1> ub = js_array_to_dlib_mat(upper_bounds);

    // Call Dlib's find_min_global with max_function_calls
    auto result = dlib::find_min_global(func, lb, ub, max_function_calls(max_calls));

    val output_js_object = val::object();
    val x_js_array = dlib_mat_to_js_array(result.x); // val::array();

    // Set 'x' attribute in the JavaScript object
    output_js_object.set("x", x_js_array);

    // Set 'y' attribute as a float
    output_js_object.set("y", result.y);

    return output_js_object;
}

// Main function as an entry point for the WebAssembly module
int main() {
    std::cout << "WebAssembly module loaded successfully!" << std::endl;
    return 0;
}


// Binding the function for WebAssembly
EMSCRIPTEN_BINDINGS(max_lipo_tr_plus_module) {
    function("max_lipo_plus_tr", &max_lipo_plus_tr);
}
```
