## [Optimizing a Javascript function with DLib's MaxLIPO+TR global optimization algorithm using Emscripten and WASM]()
<sub>January 18th 2025 </sub>
<br>

A few months ago I came across a great [article](https://blog.dlib.net/2017/12/a-global-optimization-algorithm-worth.html) regarding the MaxLIPO+TR algorithm implemented inside DLib's library. I was really excited the first time I read this post. I was amazed by the power of the algorithm and yet how simple it was.

If you're not familiar with it, it is implemented through the ```find_min_global``` function of Dlib and allows you to minimize an objective function. It requires no hyperparameters and only takes in the function to be minimized, upper and lower bounds and a maximum number of function calls.

Building on the example in the article above, we can test the ```find_min_global``` function in Python with the following code. Dlib is installed by default in Colab so you can try it out quickly.

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

Which should give you something like the following, where ```x``` is an array of our minimized arguments and ```y``` is the minima found by the algorithm.
­­­
```
x = [0.9999999994699013, 0.999999999065295]
y = 0.00000000000000000018
```

Dlib is written in C++ and offers a Python API. Our goal is to reproduce this behavior in Javascript and be able to minimize a similar function directly inside a browser environnement. For this, a small C++ wrapper around ```dlib::find_min_global``` does the job, allowing us to call a Javascript function directly from a WASM environnement.

#### Setting up the Javascript interface

But first, since the goal is to reproduce this behavior in Javascript, we start by creating a function called ```maxLipoPlusTr``` that will be able to minimize any numerical Javascript function (i.e. that takes in and returns a numerical value). Here's what we want it to look like.

```js
import { maxLipoPlusTr } from "./max-lipo-plus-tr.js"

// Rosenbrock function for 3D space, is minimized at x0 = 1, x1 = 1, x2 = 1
function rosenbrock3D(x0, x1, x2) {
    return 100 * ((x1 - x0**2)**2 + (x2 - x1**2)**2) + (1 - x0)**2 + (1 - x1)**2;
}

const lowerBounds = [-100, -100, -100];
const upperBounds = [100, 100, 100];
const maxCalls = 300;

let result = await maxLipoPlusTr(rosenbrock3D, lowerBounds, upperBounds, maxCalls);

console.log(result);

```

The value of  ```result``` should give us something like the following, where again ```x``` is an array of our minimized arguments and ```y``` is the minima found by the algorithm.

```
 {
   "x": [0.9999998222686982, 0.999999575414733, 0.9999992156860394], 
   "y": 0.00000000000091659844
 }
```

Under the hood, the ```maxLipoPlusTr``` function loads the WASM module, wraps the input function inside an object and sends it to the WASM environnement to be minimized inside DLib's ```find_min_global``` function. 

Here's ```max-lipo-plus-tr.js```.

```js
import createMaxLipoTrPlusModule from './find_min_global_o3.js'

var Module = null;

export async function maxLipoPlusTr(theFunction,
                                    lowerBounds,
                                    upperBounds,
                                    maxCalls) { 

    if (!Module) { // load Module if not loaded yet
        Module = await createMaxLipoTrPlusModule();
    }

    // Create an instance of JsFunction with the Rosenbrock function
    const jsFunc = new JsFunction(theFunction);

    const result = await Module.max_lipo_plus_tr(jsFunc, lowerBounds, upperBounds, maxCalls);

    return { x: result.x, y: result.y };
}
```

Notice the minimized function is wrapped into a ```JsFunction``` object. It's this object that is then passed into the ```max_lipo_plus_tr``` call.

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

#### Writing a C++ wrapper

Once our Javascript is set up, we create a C++ wrapper to interact with dlib and compile it using Emscripten to make it accessible inside a web environnement.

Here the ```find_min_global_wrapper.cpp``` file.

```cpp
#include <dlib/global_optimization.h>
#include <dlib/matrix.h>
#include <emscripten/bind.h>
#include <emscripten/val.h>
#include <functional>

using namespace emscripten;
using namespace dlib;

// Convert a dlib matrix to a Javascript Array object
emscripten::val dlib_mat_to_js_array(const dlib::matrix<double, 0, 1>& vec) {
    emscripten::val js_array = emscripten::val::array();
    for (std::size_t i = 0; i < vec.nr(); ++i) {
        js_array.call<void>("push", vec(i));
    }
    return js_array;
}

// Convert a Javascript Array object to a dlib matrix
dlib::matrix<double, 0, 1> js_array_to_dlib_mat(const emscripten::val& js_array) {
    dlib::matrix<double, 0, 1> vec(js_array["length"].as<std::size_t>());
    for (std::size_t i = 0; i < vec.nr(); ++i) {
        vec(i) = js_array[i].as<double>();
    }
    return vec;
}

emscripten::val max_lipo_plus_tr(emscripten::val jsFunction,
                     emscripten::val lower_bounds, // bounds are Javascript Arrays
                     emscripten::val upper_bounds,
                     size_t max_calls) {

    std::function<double(const dlib::matrix<double, 0, 1>&)> func;

    func = [jsFunction](const dlib::matrix<double, 0, 1>& vec) -> double {
        for (size_t i = 0; i < vec.nr(); ++i) {
            jsFunction.call<void>("setArg", i, vec(i));
        }
        return jsFunction.call<double>("bang");
    };

    dlib::matrix<double, 0, 1> lb = js_array_to_dlib_mat(lower_bounds);
    dlib::matrix<double, 0, 1> ub = js_array_to_dlib_mat(upper_bounds);

    auto result = dlib::find_min_global(func, lb, ub, max_function_calls(max_calls));

    emscripten::val output_js_object = emscripten::val::object();
    emscripten::val x_js_array = dlib_mat_to_js_array(result.x);

    output_js_object.set("x", x_js_array);
    output_js_object.set("y", result.y);

    return output_js_object;
}

EMSCRIPTEN_BINDINGS(max_lipo_tr_plus_module) {
    function("max_lipo_plus_tr", &max_lipo_plus_tr);
}
```

All of the above C++ code is pretty straightforward, except maybe the following segment, which is where the magic happens and deserves more detailed attention.

```cpp
std::function<double(const dlib::matrix<double, 0, 1>&)> func;

func = [jsFunction](const dlib::matrix<double, 0, 1>& vec) -> double {
    for (size_t i = 0; i < vec.nr(); ++i) {
        jsFunction.call<void>("setArg", i, vec(i));
        }
        return jsFunction.call<double>("bang");
};
```

DLib's ```find_min_global``` expects to receive a C++ function as argument, so we need a way to "cast" our Javascript function, encapsulated inside the ```jsFunction``` variable, to a C++ function, which is why we need to introduce a lambda, defined as the ```func``` variable with ```std::function<...>```. Then, ```[jsFunction]``` inside the lambda’s brackets captures the variable ```jsFunction``` from the outer scope so that it can be used inside the lambda body at every iteration of the optimization.

#### Compiling to WASM

To compile ```find_min_global_wrapper.cpp``` to our imported ```find_min_global_o3.js```, the following does the job. Notice multi-threading is deactivated by default, but it should work if one was to activate multi-threading with ```-s USE_PTHREADS=1```.

```
emcc -O3 find_min_global_wrapper.cpp dlib/global_function_search.o dlib/thread_pool_extension.o -I./ -o find_min_global_o3.js \
    -s MODULARIZE=1 \
    -s EXPORT_NAME="createMaxLipoTrPlusModule" \
    -s EXPORT_ES6=1 \
    -s ALLOW_MEMORY_GROWTH=1 \
    -s ALLOW_TABLE_GROWTH=1 \
    -s INITIAL_MEMORY=131072 \
    -s "EXPORTED_RUNTIME_METHODS=['ccall', 'cwrap']" \
    --bind
```
Which should output a ```find_min_global_o3.js``` and ```find_min_global_o3.wasm```, to include both inside your project.

And that's it !

You can try it out [here](https://dany-demise.github.io/max-lipo-plus-tr-js) and get the code [here](https://github.com/dany-demise/max-lipo-plus-tr-js).

⚠️ I'm currently using Emscripten 3.1.74 because of a [compiling issue](https://github.com/davisking/dlib/issues/3045) with version 4.0.0.



<br>
