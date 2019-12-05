# Nested Arrays

We follow [Pyodide](https://hacks.mozilla.org/2019/04/pyodide-bringing-the-scientific-python-stack-to-the-browser/)'s example by expressing our multi-dimensional numerical data as Arrays of TypedArrays. NestedArrays are always fully loaded in-memory, think of them as `numpy.ndarray` in Python. 
> In the future we could consider writing an alternative implementing strided arrays and views like NumPy (and `scijs/ndarray`) is implemented.

## Background
Javascript has support for one-dimensional array data that contain a single numeric type through [TypedArrays](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray). There is no native multi-dimensional array support, nor is there a library like [NumPy](https://numpy.org/) that is the obvious choice for multi-dimensional arrays. Multi-dimensional indexing needs to be built on top.

> There exist some packages like [scijs/ndarray](https://github.com/scijs/ndarray) that try to address this need. This package however seems to have grown stale and its source code gets [generated by string concatenation](https://github.com/scijs/ndarray/blob/master/ndarray.js) at runtime (yikes).
As the scijs ecosystem does implement a lot of operations you may want to perform on your data, so we make sure that our alternative, NestedArrays, will be easily convertible to the `scijs/ndarray` equivalent.


## Creating NestedArrays

NestedArray is a class that wraps the multi-dimensional array in this format with some metadata (such as its shape). Let's create one!

```javascript
import { NestedArray } from "zarr";

const binaryData = Int32Array.from([0,1,2,3,4,5]);

// The second argument is the shape
const nArray = new NestedArray(binaryData, [2, 3]);
console.log(nArray);
```
```output
NestedArray {
  shape: [ 2, 3 ],
  dtype: '<i4',
  data: [ Int32Array [ 0, 1, 2 ], Int32Array [ 3, 4, 5 ] ]
}
```

### Creating from binary data

Alternatively we can also create one from a raw buffer of data, we will have to specify its datatype.

```javascript
const binaryData = new ArrayBuffer(8*2*3*64);
// We will have to specify the type as it can not be inferred.
const nArray = new NestedArray(binaryData, [2, 3, 64], "<f8");

console.log(nArray);
```
```output
NestedArray {
  shape: [ 2, 3, 64 ],
  dtype: '<f8',
  data: [
    [ [Float64Array], [Float64Array], [Float64Array] ],
    [ [Float64Array], [Float64Array], [Float64Array] ]
  ]
}
```
> These data type strings (`DtypeString`) are the same as the ones found in NumPy. The currently supported types can be found in [this file](https://github.com/gzuidhof/zarr.js/blob/master/src/types.ts).

## Slicing
Python (and NumPy in particular) allows you to slice and index arrays by using the square brackets notation (e.g. `myArray[0, :2]`). There are no such language features in JS, so our own implementation is provided for this.

NestedArrays expose two methods for getting and setting respectively, let's first create an array:
```javascript
import { NestedArray, slice } from "zarr";

const binaryData = Int32Array.from([0,1,2,3,4,5]);
const n = new NestedArray(binaryData, [2, 3]);

console.log(n);
/**
 * NestedArray {
 *   shape: [ 2, 3 ],
 *   dtype: '<i4',
 *   data: [ Int32Array [ 0, 1, 2 ], Int32Array [ 3, 4, 5 ] ]
 * }
 */
```

Slicing a NestedArray with its `get` function returns either another NestedArray or a number. It takes a single argument: the `selection` you want from the array. 

> For readability we state the `data` field value in the comment next to the examples in case of NestedArray result.

#### Selecting along the first dimension

```javascript
    // n[0] in python
    n.get([0]); // Int32Array [ 0, 1, 2 ]
    n.get(0); // Int32Array [ 0, 1, 2 ]
```


#### Selecting a single value
```javascript
    // n[0,0]
    n.get([0, 0]); // 0
    n.get([-1, 0]); // 3
    n.get([-1, 1]); // 4
```

#### Selecting everything, like `n[:]` in python
```javascript
    n.get(null); // [ Int32Array [ 0, 1, 2 ], Int32Array [ 3, 4, 5 ] ]
    n.get([null]); // [ Int32Array [ 0, 1, 2 ], Int32Array [ 3, 4, 5 ] ]
    n.get(":"); // [ Int32Array [ 0, 1, 2 ], Int32Array [ 3, 4, 5 ] ]
```

#### Using slices
```javascript
    // n[:, 0:2]
    n.get([slice(null), slice(0, 2)]); // [ Int32Array [ 0, 1 ], Int32Array [ 3, 4 ] ]
    // n[::-1, 0:2]
    n.get([slice(null, null, -1), slice(0, 2)]); // [ Int32Array [ 3, 4 ], Int32Array [ 0, 1 ] ]
```

#### Using ellipsis (`...` in python)
```javascript
    // n[..., 0:2]
    n.get(["...", slice(0, 2)]); // [ Int32Array [ 0, 1 ], Int32Array [ 3, 4 ] ]
```

## Setting NestedArray values

Setting values works the same way, `set` takes two arguments: the `selection` and the value you want to set it to.

Let's imagine that after each of these commands we reset n to its initial value. In reality we are of course mutating `n`!

#### Setting to a scaslar

```javascript
n.set([1, 1], 100); // [ Int32Array [ 0, 1, 2 ], Int32Array [ 3, 100, 5 ] ]
n.set([1, slice(0, 2)], 100); // [ Int32Array [ 0, 1, 2 ], Int32Array [ 100, 100, 5 ] ]

n.set([1, 1], 100); // [ Int32Array [ 0, 1, 2 ], Int32Array [ 3, 100, 5 ] ]
n.set([null], 100); // [ Int32Array [ 100, 100, 100 ], Int32Array [ 100, 100, 100 ] ]
```

#### Setting to a different NestedArray

```javascript
// n[:] = n[:, ::-1]
n.set([null], n.get([null, slice(null, null, -1)])); // [ Int32Array [ 2, 1, 0 ], Int32Array [ 5, 4, 3 ] ]

// n[:, 0:2] = n[:, 1:3]
n.set([null, slice(0,2)], n.get([null, slice(1,3)])); // [ Int32Array [ 1, 2, 2 ], Int32Array [ 4, 5, 5 ] ]

// TODO this currently errors due to false shape mismatch error..
// n[0] = n[1]
n.set(0, n.get(1));
```

> ### Instead of `get` and `set` can we use square brackets notation such as  `array[0,2]`? 
> Since recently it is possible to override the behaviour of the square brackets indexing and setting through Proxy objects, this comes with a small performance cost. We can provide a Proxy wrapper for NestedArrays in the future, but these can not be provided for Zarr Arrays as it would be impossible to verify setting a value succeeded succesfully using an asynchronous store. 