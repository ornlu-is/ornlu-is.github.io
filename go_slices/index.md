# A deep dive on Golang slices


Slices are Go's bread and butter. However, more often than not, small mistakes happen because it isn't exactly clear what actually is a slice. Since they are so prevalent in Go code, I decided to dive a bit deeper into their internal structure, how they are handled in different situations, and how some of the issues that arise can be avoided.

## But first, arrays

Before diving into slices, it is important to know that Go also has the concept of arrays. An array is basically a numbered sequence of elements of the same type, known as the element type, and are useful when you are sure of how many memory positions you require. You can declare an array in the following ways.

```go
x := [3]int{}
y := [3]int{1, 2, 3}
z := [...]int{4, 5, 6, 7}
```

The first example creates an array of size three with all its elements initialised as the element type's default value. One very important detail here is that the size of an array is part of its type, which consequently means that you cannot specify an array's size via a variable because its type must be resolved at compile time, and it also means that you cannot write functions that work with arrays of any size. The second example creates an array of size three with the given values and the last example foregoes the explicit integer specification of the array's size in detriment of inferring it from the number of elements in the given slice literal.

The last two details that we should be aware when working with arrays is that, in Go, arrays are values, which means that if you assign one array to another, you are copying all of its elements. Moreover, arrays in Go are comparable, meaning you can write the following:

```go
x := [...]int{2, 3}
y := [...]int{3, 4}
if x == y {
    // do something
}
```

## What is a slice in Go?

In Go, arrays are rarely used and are primarily a building block for the concept of slices. So... what are slices? A slice isn't one thing, it's actually three things:
* A pointer to an underlying array;
* The length of the underlying array;
* And the capacity of the underlying array.

When you create a slice, you declare the slice's element type, its length, and, optionally, its capacity. The Go runtime takes the given element type and capacity and allocates a contiguous memory segment of capable of holding a number of elements equal to the given capacity, and addresses the pointer to the beginning of this memory segment. Note that if you only specify the length and not the capacity of the slice, Go will assume that the capacity is equal to the length.

Unlike arrays, slices are not comparable except with the `nil` value, which is the default value for a slice.

## Initialising slices

There are a few ways one can declare a slice:

```go
a := []int{1, 2}
b := []int{}
var c []int
d := make([]int, 3)
e := make([]int, 0, 2)
```

Let us go through these one by one. The first declaration uses a slice literal to initialise the slice, meaning that it creates a slice populated with the given values in the order they are presented. This will infer both the length and capacity of the slice from the number of elements in the slice literal. The second one behaves much like the first one, but we are initialising a slice with an empty slice literal, which means that, if we print out the contents of `b`, we get the following:
```plaintext
[]
```
which is an empty slice. You might think that the third slice declaration produces a similar result to the second one, and that's where you're wrong. The third only declares a slice of integers, it does not perform any initialisation, meaning that its value is the slice default value, which is `nil`.

The last two use the built-in `make` function to create slices. When creating slices, `make` expects either two or three arguments. The first is the type of slice you are trying to create, the second is that slice's length, and the third is its capacity. When you use `make` to create a slice with non-zero length, keep in mind that you will be creating a slice with that number of elements with their values set to the element type's default value. In other words, the output of the fourth slice declaration is:
```plaintext
[0, 0, 0]
```
However, if you just wish to allocate the memory for a slice, you can specify zero length and some non-zero capacity, much like the last declaration. Note that this will create an empty slice, on par with the second declaration, but the underlying allocated array will have the specified capacity.

## Growing slices and why you should initialise them with `make`

Let us say you have a slice of integers that is currently holding the numbers `1` and `2`, and you want to add a third one. In Go, this is performed via the built-in `append` function, which takes as arguments the slice to which you wish to append new elements, and a variable number of new elements to append, and returns a copy of the resulting slice, which you tipically will use to override the original one:

```go
x := []int{1, 2}
x = append(x, 3)
```

If `x` was initialised from a slice literal that had two elements, then `x` has capacity and length both equal to two. But when I appended `3` to it, it now has length three which is more than the previous capacity. Which prompts the question: so what is the role of a slice's capacity?

To understand this, we need to know what goes on under the hood when we append something to a slice that is already at its capacity limit. When you append to a slice, you are adding one or more values to it, and each of these values will logically increase the slice's length by one. The interesting bit happend when the length is already equal to the capacity. In this case, your slice has run out of space in the underlying array's memory to add new elements. As such, the Go runtime will allocate a new slice with larger capacity, copy the original slice to the new one, add the new elements to it, and the new slice is then returned by `append`.

Logically, these operations all take time. You are no longer just adding one element to a slice, you are creating a new slice, allocating memory, copying all elements to the new slice and only then you add a new element to it. Moreover, Go's garbage collector will now have the additional task of freeing up the memory used by the old slice. As such, it is a good practice to create a slice with an upper bound on its capacity whenever possible:

```go
x := make([]int, 0, upperBound)
```

This will avoid performing all the aforementioned extra computations and help you squeeze a tiny bit more of performance from your application.

## Reslicing, a.k.a., how you'll probably shoot yourself in the foot

One operation on slices that is supported in Go is the slicing operation. This allows you to obtain a subset of a given slice:

```go
x := []int{1, 2, 3, 4}  // [1, 2, 3, 4]
y := x[:2]              // [1, 2]
```

But there is one caveat with slicing: it does not create a copy of the data. Instead, the new slice object created by slicing has a new length, but the same capacity as the original slice and its pointer is pointing to an element of the same underlying array. This effectively means that if you rewrite one of the elements of `y`, *e.g.*, `y[1]=666`, it will also change the element in the same position for `x`. And the same holds true if you take a slice of a slice of a slice (and so on).

Slicing is a powerful tool, but must be used with care, espectially when performing value assignments, since it might result in some unexpected behaviour.

## Using `copy`

We have seen slicing as a way of copying parts of a slice and potentially shoot ourselves in the foot. The obvious question that arises is: how can I safely copy the contents of a slice, to another slice, without having to deal will all these pointer shenanigans? Go has got your back with the built-in `copy` function. 

This function takes two arguments, a destination slice and a source slice, and copies as many elements of the source slice to the destination slice as the destination slice's length allows. Additionally, it also returns the number of copied elements.

For example, if you want to copy just the first two elements of a given slice into an entirely independent slice, you'd simply write:
```go
x := []int{1, 2, 3, 4}
y := make([]int, 2)
_ = copy(y, x)
```

Now you can freely manipulate `y` without having to worry about what will happen to `x`.

## References

* https://go.dev/ref/spec#Slice_types
* https://go.dev/blog/slices-intro
* https://go.dev/doc/effective_go#slices
* Jon Bodner, *"Learning Go"*, O'Reilly, 1st Edition

