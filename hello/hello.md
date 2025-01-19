## [Compiling DLib's MaxLIPO+TR global optimization algorithm to WASM]()
<sub>January 18th 2024 </sub>
<br>

A few while ago I came across a great [post](https://blog.dlib.net/2017/12/a-global-optimization-algorithm-worth.html) shared by the great [@sasuke___420](https://x.com/sasuke___420) regarding the MaxLIPO+TR algorithm implemented inside DLib's library. I was really excited the first time I read this post. I was awestruck by the sheer power of the algorithm and yet how simple it was.

If you're not familiar with it, it is implemented through the find_min_global function of Dlib and allows the programmer to minimize an objective function. It requires no hyperparameters and only takes in the function to be minimized, upper and lower bounds and a maximum number of function calls.

Building on the example in the article above, we can test the find_min_global function with the following. Dlib is installed by default in Colab so you can try it quickly :

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

#### Compiling to WASM

Dlib is written in C++ and offers a Python API. I've been f-ing around WASM and Emscripten lately and wanted a small and interesting project to learn so I decided to give a shot at compiling Dlib to WASM.

The real challenge was porting a Javascript function to Emscripten to be minimized from there.
