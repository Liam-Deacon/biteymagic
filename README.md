# Bitey Magic

Bitey is a tool to import LLVM bitcode directly into Python. Bitey magic is a small IPython extension which adds a %%bitey cell magic for automatically compiling C (or C++) code into LLVM bitcode and loading the bitcode with Bitey.

Requirements:
 - Bitey, importable from Python as `import bitey`
 - clang, executable as clang in the shell

Load extension within IPython:

```%load_ext biteymagic```

## Install

```pip install git+https://github.com/Lightslayer/biteymagic```

## Demos
The following demos are taken, with slight modification, from the examples in the Bitey source.
The Basics
Calculate the Fibonacci numbers using a simple recursive formula.

```c
%%bitey
int fib(int n) {
  if (n < 3) {
    return 1;
  } else {
    return fib(n-2) + fib(n-1);
  }
}
```

```python
>>> map(fib, xrange(1,10))
[1, 1, 2, 3, 5, 8, 13, 21, 34]
```
Bitey understands most basic C datatypes including integers, floats, void, pointers, arrays, and structures. Because it builds a ctypes based interface, you would access the code using the same techniques. Here is an example that mutates a value through a pointer.

```c
%%bitey
void mutate_int(int *x) {
  *x *= 2;
}
```

```python
import ctypes
x = ctypes.c_int(2)
mutate_int(x)
x.value

4
```

Structs
An example involing a structure. One subtle issue with structure wrapping is that LLVM bitcode doesn't encode the names of structure fields. So, Bitey simply assigns them to an indexed element variable like this:
``` python
>>> p1.e0         # (Returns the .x component)
3
>>> p1.e1         # (Returns the .y component)
4
```

```c
%%bitey
#include <math.h>

struct Point {
  double x;
  double y;
};

double distance(struct Point *p1, struct Point *p2) {
  return sqrt((p1->x - p2->x)*(p1->x - p2->x) + 
	      (p1->y - p2->y)*(p1->y - p2->y));
}
```

```python
>>> p1 = Point(3,4)
>>> p2 = Point(6,8)
>>> print distance(p1,p2)
>>> print p1.e0, p1.e1
5.0
3.0 4.0
```

Performance
The performance profile of Bitey is going to be virtually identical that of using ctypes. LLVM bitcode is translated to native machine code and Bitey builds a ctypes-based interface to it in exactly the same manner as a normal C library.

```c
%%bitey -03
/* A function that determines if an integer is a prime number or not.
   This is just a naive implementation--there are faster ways to do it */

int isprime(int n) {
  int factor = 3;
  /* Special case for 2 */
  if (n == 2) {
    return 1;
  }
  /* Check for even numbers */
  if ((n % 2) == 0) {
    return 0;
  }
  /* Check for everything else */
  while (factor*factor < n) {
    if ((n % factor) == 0) {
      return 0;
    }
    factor += 2;
  }
  return 1;
}
```

```python
>>> %%timeit
>>> isprime(10143937)
10000 loops, best of 3: 36.8 us per loop
```

Additional optimization flags given after %%bitey are passed directly to clang. For example, consider a simple function which sums a vector.

```c
%%bitey
double sum1d(double* x, int n)
{
  int i;
  double ret;
  ret = 0.0;
  for(i=0; i<n; i++)
    ret = ret + x[i];
  return ret;
}
```

```c
%%bitey -O2
double sum1d_O2(double* x, int n)
{
  int i;
  double ret;
  ret = 0.0;
  for(i=0; i<n; i++)
    ret = ret + x[i];
  return ret;
}
```

```python
import numpy as np
n = 1000
A = np.random.rand(n)
x = np.ctypeslib.as_ctypes(A)
print('sum1d:')
%timeit sum1d(x, n)
print('sum1d_O2:')
%timeit sum1d_O2(x, n)
sum1d:
100000 loops, best of 3: 7.39 us per loop
sum1d_O2:
100000 loops, best of 3: 3.29 us per loop
```

C++ Support
Bitey does not support C++ functions, but C++ code can be compiled if the function is decorated with extern "C". In addition, you will need to pass -x c++ to clang to specify that the language is C++.

```c
%%bitey -O2 -x c++
#include <numeric>
extern "C" double sum1d_cpp(double* x, int n)
{
  return std::accumulate(x, x+n, 0.0);
}
```
  
```python
print('sum1d_cpp')
%timeit sum1d_cpp(x, n)
sum1d_cpp
100000 loops, best of 3: 3.33 us per loop
```
