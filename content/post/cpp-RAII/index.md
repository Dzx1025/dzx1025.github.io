---
title: RAII
description:
date: 2023-02-22T12:18:59Z
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

RAII(Resource Acquisition Is Initialization) is a C++ programming idiom that ties resource management to object lifetime.

## When Objects Should Not Be Stack-Allocated

Objects should be heap-allocated when:

- Object size is very large
- Object size cannot be determined at compile time
- Object is a function return value but should not be returned by value due to specific reasons

## Resource Management

RAII uses stack-based objects and destructors to effectively manage system resources (including heap memory). For objects created with new inside stack scope, they should be wrapped in a class to ensure automatic destruction and resource release when the function returns.

Note: For any new-allocated objects on the stack, they should be wrapped in a class to ensure automatic cleanup through the destructor when going out of scope.
