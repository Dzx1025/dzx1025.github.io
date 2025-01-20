---
title: Template Type Deduction
description: 
date: 2023-03-12T12:18:59Z
image:
math:
license:
hidden: false
comments: true
draft: false
tags:
  - c/c++
categories:
  - notes
---

## First, let's distinguish different cases of const

```cpp
// Variables
const int x;       // top-level const
// References
const int& r1 = x; // low-level const
int& const r2 = x; // this syntax doesn't exist
// Pointers
const int* p1 = &x; // low-level const
int* const p2 = &x; // top-level const
```

For top-level const, it indicates that the variable **itself** cannot be modified. As the name suggests, "top-level" refers to the topmost layer, which is the variable itself.

For low-level const, it indicates that the space **pointed to** by the reference/pointer cannot be modified.

Why can't references be declared with top-level `const`? This is because **all references are inherently top-level `const`** - once a reference is initialized, it cannot be changed to reference another object. Since references already have the top-level `const` property, there's no need for us to explicitly declare it. As we can see, the most common references/pointers we encounter are low-level `const`, meaning they modify the immutability of the variables they point to.

With this understanding, let's look at template type deduction.

## Type Deduction

First, let's introduce common terminology. In template function calls using `f(expr)`, we have arguments and parameters. Here, we'll mainly study the parameters and the typename's value type, which are `T` and `ParamType`.

C++ has three forms of `ParamType` declarations, representing three different deduction rules:

### _ParamType_ is a Reference/Pointer, but not a Universal Reference

```cpp
template <typename T>
void f(T& param);
// For pointer ParamType:
void f(T* param);
```

First, **if the type of `expr` is a reference, the reference part is ignored**, so `int&` and `int` should yield the same deduction result. Since `ParamType` already specifies that the function parameter is a reference, when both `expr` and `paramtype` are references, `expr`'s reference part is ignored.

The same principle applies when both are pointers - `int*` and `int` are treated equally because `ParamType` already indicates that this template wants to use a reference/pointer type parameter.

Note that when `ParamType` is a reference type and `expr` is a pointer type, the pointer part isn't ignored - `T`'s type will match `expr`, as shown below. However, in normal circumstances, why would you pass a pointer to a reference? Conversely, when `ParamType` is a pointer and `expr` is a reference type, calling `f(expr)` requires passing the address of the reference to match, which is essentially the same as passing a variable's address.

```cpp
int x=27;
const int cx=x;
const int& rx=x;
const int * p1=&x;
int* const p2=&x;
```

```cpp
f(x);  // T-> int           f(int&)
f(cx); // T-> const int     f(const int&)
f(rx); // T-> const int     f(const int&)
f(p1); // T-> const int *   f(const int *&)
f(p2); // T-> int *const    f(int *const &)
```

For callers, when passing a `const` object to a reference parameter, they expect the object to maintain its immutability. If the passed parameter cannot modify the content it points to, the template should obviously also protect that space from modification.

Therefore, the parameter will also be reference-to-const. This makes it safe to pass a const object to a template with a `T&` type parameter: the object's constness is preserved as part of `T`.

We can see that `T` preserves both top-level and low-level const attributes from the argument `expr`, while `ParamType` simply adds a `&` after the deduced `typename T`, becoming `const T&`.

### _ParamType_ is a Universal Reference

```cpp
template<typename T>
void f(T&& param);
```

In this case, `ParamType` is called a universal reference. For universal references, we must discuss two situations based on `expr`'s **value category**:

1. When `expr` is an lvalue: Both `T` and `ParamType` are deduced as lvalue references. This is **the only case in template type deduction where `T` is deduced as a reference**.

   In this case, `ParamType`'s final result is actually consistent with scenario 1's deduction result. The only difference is that in scenario 1, `&` is added to the parameter after `T` is deduced as a base type, while for universal references, `&` is added when deducing `T` - only `T` differs.

   ```cpp
   int x=27;
   const int cx=x;
   const int& rx=cx;
   ```

   ```cpp
   f(x);           //x is lvalue    T-> int&,        f(int&)
   f(cx);          //cx is lvalue   T-> const int&    f(const int&)
   f(rx);          //rx is lvalue   T-> const int&    f(const int&)
   ```

2. When `expr` is an rvalue: Use normal deduction rules (same as **scenario 1**)

   Here, `T` will be the "base" type, while `ParamType` will be an rvalue reference, i.e., `T&&`

   ```cpp
   f(27);              // T-> int      f(int&&)
   ```

### _ParamType_ is Neither a Pointer/Reference Nor a Universal Reference

```cpp
template<typename T>
void f(T param);
```

This situation is similar to pass-by-value handling. This means the template actually generates a complete new object copy.

1. As before, if `expr`'s type is a reference, ignore the reference part
2. Unlike scenario 1, `expr`'s `const` and `volatile` attributes are also ignored here, which is very reasonable because the original object's attributes shouldn't affect the copy's attributes - after all, the template is only responsible for passing types

Therefore, the above `x cx rx` will all be deduced as `int`

However, note that `const` is only ignored when passing by value to parameters. For reference-to-`const` and pointer-to-`const` parameters, besides themselves, they also contain the space they point to. What happens to this pointed space?

```cpp
const char* const ptr="Hello";   // ptr is a const pointer pointing to const object

const int x=12;

f(ptr);                          // T-> const char *
```

In this case, only the **top-level** `const` of `expr` is ignored, while the **low-level** `const` is preserved. Therefore, when we call `f("hello")`, `T` is deduced as `const char *`. It's like we copied a pointer - the template is still responsible for ensuring that the data pointed to by two identical pointers cannot be modified.

### Array Function Decay to Pointer

```cpp
template<typename T>
void f(T param);

const char name[] = "J. P. Briggs";     // name's type is const char[13]
f(name)  // T-> const char * i.e., f(const char*)
```

But if we make a small change...

```cpp
template<typename T>
void f(T& param);

f(name)  // T-> const char[13]  f(const char(&)[13])
```

This type includes the array's size. In this example, `T` is deduced as `const char[13]`, and the type of `f`'s parameter (reference to this array) is `const char (&)[13]`

But since you're writing in C++, it's recommended to use array instead....