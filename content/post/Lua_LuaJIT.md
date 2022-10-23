---
title: "Lua: LuaJIT"
date: 2022-10-20
lastmod: 2022-10-23
draft: false
tags: ["Lua", "LuaJIT"]
categories: ["Language"]

toc: false

---


LuaJIT is a Just-In-Time Compilerfor the Lua programming language.

> 为了尽可能成功JIT, 尽量避免使用未支持JIT的Lua原语与库函数
> ([NYI](http://wiki.luajit.org/NYI))


# FFI
[FFI_Library](https://luajit.org/ext_ffi.html)

## 使用动态链接库
* 用C生成动链库(例如`myffi.c` -> `libmyffi.so`)
  ```c
  int add(int x, int y)
  {
    return x + y;
  }
  ```
  ```sh
  gcc -g -o libmyffi.so -fpic -shared myffi.c
  ```
* Lua调用(LuaJIT)
  ```lua
  local ffi = require("ffi")
  local myffi = ffi.load("libmyffi")
  ffi.cdef[[
  int add(int x, int y);
  ]]
  local res = myffi.add(1, 2)
  print(res)  -- output: 3
  ```

## 使用C数据结构
* `cdata`类型用来将任意C数据保存在Lua变量中.
  这个类型相当于一块原生的内存, 除了赋值和相同性判断,
  Lua没有为这预定义任何操作.
  然而, 通过元表可以为`cdata`自定义一组操作
* `cdata`不能在Lua中创建出来, 也不能在Lua中修改.
  这样的操作只能通过C API -> 这一点保证了宿主程序完全掌管其中的数据
* 可以将C类型与`metamethod(元方法)`关联起来, 这个操作只能进行一次.
  `ffi.metatype`会返回一个该类型的构造函数.
  原始C类型也可以被用来创建数组, 元方法会被自动地应用到每个元素
  > 元表与C类型的关联是永久的, 而且不允许被修改, `__index`元方法也是
* 例
  ```lua
  local ffi = require("ffi")

  ffi.cdef [[
  typedef struct {
      double x;
      double y;
      double z;
  } Vector3d;
  ]]

  local mt = {}

  mt.__add = function (a, b)
      return ffi.new("Vector3d", a.x + b.x, a.y + b.y, a.z + b.z)
  end

  mt.__tostring = function (self)
      return string.format("Vector3d[%f, %f, %f]", self.x, self.y, self.z)
  end

  local Vector3d = ffi.metatype("Vector3d", mt)
  ```

## Lua与C语法对应关系
| Idiom                      | C code        | Lua code     |
|----------------------------|---------------|--------------|
| Pointer dereference        | x = \*p       | x = p[0]     |
| int \*p                    | \*p = y       | p[0] = y     |
| Pointer indexing           | x = p[i]      | x = p[i]     |
| int i, \*p                 | p[i+1] = y    | p[i+1] = y   |
| Array indexing             | x = a[i]      | x = a[i]     |
| int i, a[]                 | a[i+1] = y    | a[i+1] = y   |
| struct/union dereference   | x = s.field   | x = s.field  |
| struct foo s               | s.field = y   | s.field = y  |
| struct/union pointer deref | x = sp->field | x = sp.field |
| struct foo \*sp            | sp->field = y | s.field = y  |
| int i, \*p                 | y = p - i     | y = p - i    |
| Pointer dereference        | x = p1 - p2   | x = p1 - p2  |
| Array element pointer      | x = &a[i]     | x = a + i    |

## 内存管理
TODO
