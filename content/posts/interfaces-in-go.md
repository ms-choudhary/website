+++
title ="Interfaces in Go"
date = "2018-06-02"

[taxonomies]
tags=["golang"]
+++

Go is a strongly typed language, which doesn't support generics. Although not your common object oriented language like java, go does support types and methods on those types. There are no constructors, inheritance. The whole idea is to intentionally keep it light weight. Interface is something similar to what you've in other languages. This is how you define them:

```
package io

type Writer interface {
    Write(p []byte) (n int, err error)
}
```

Interfaces are contracts:

> There is another kind of type in Go called an interface type. An interface is an abstract type. It doesn’t expose the representation or internal structure of its values, or the set of basic operations they support; it reveals only some of their methods. When you have a value of an interface type, you know nothing about what it is; you know only what it can do, or more precisely, what behaviors are provided by its methods.

As is expected, if a function expects `io.Writer` as argument. We can pass any of the concrete type which implements `io.Writer` to it. And based on the actual type, same function can either write to `os.Stdout` or a file or a string buffer. Thus, exhibiting polymorphism.

So how do concrete types implement a particular interface. Well it need not to. Go interfaces are implicitly satisfied if a particular concrete type implements the methods of the interface (For example, if `os.File` implements a `Write` method with signature similar to above, it satisfies that interface). This way you can define a new interface and existing types will still satisfy it, if they support all methods. This is particularly usefully if you don't own those types.

# Internals

Interface internally consists of two values: dynamic type and dynamic value. Like other types, interfaces are initialized, by default, to well defined value, where both *type* and *value* are `nil`:

```
var w io.Writer
```

![Full-width image](https://i.ibb.co/Nn8jnnb/f7JtK8G.png)

An interface is `nil` when it's *type* value is `nil`. Just like pointers & other aggregate types, interfaces can be compared to `nil` (i.e., `w == nil` or `w != nil`).

Interfaces can only be assigned to types which support all methods defined by the interface.

```
var w io.Writer
// os.Stdout is *os.File pointing to standard output
w = os.Stdout
```


On assignment, *type* of interface value get set to the actual concrete type & *value* to it's actual value.

![](https://i.ibb.co/drgKgbv/O8uK3cF.png)

When we execute a method on interface,

```
w.Write([]byte("hello"))
```
compiler infers the address of the method from the actual concrete type and calls it with *value* as receiver. So `w.Write` internally executes `(*os.File).Write`.

# Equality

Interface can be compared for equality using `==` & `!=`. Two interfaces are equal if both their *type* and *value* are equal.

