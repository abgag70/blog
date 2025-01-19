## [Compiling DLib's MaxLIPO+TR global optimization algorithm to WASM]()
<sub>January 18th 2024 </sub>
<br>

A few while ago I came across a great post shared by the great [@sasuke___420](https://x.com/sasuke___420) regarding the MaxLIPO+TR algorithm implmented inside DLib's library. I was really excited the first time I read this post. I was awestruck by the sheer power of the algorithm and yet how simple it was.

If you're not familiar with it, it is implemented through the find_min_global function of Dlib and allows the programmer to minimize an objective function. It requires no hyperparameters and only takes in the function to be minimized, upper and lower bounds and a maximum number of function calls.

Here's the Python example provided in the article above :

```python
def holder_table(x0,x1):
    return -abs(sin(x0)*cos(x1)*exp(abs(1-sqrt(x0*x0+x1*x1)/pi)))

x,y = dlib.find_min_global(holder_table, 
                           [-10,-10],  # Lower bound constraints on x0 and x1 respectively
                           [10,10],    # Upper bound constraints on x0 and x1 respectively
                           80)         # The number of times find_min_global() will call holder_table()
```
