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
[ext_ffi_tutorial](https://luajit.org/ext_ffi_tutorial.html)
| Idiom                                                | C code                                   | Lua code                                                     |
|------------------------------------------------------|------------------------------------------|--------------------------------------------------------------|
| Pointer dereference<br/>int \*p;                     | x = \*p;<br\>\*p = y;                    | x = p[0]<br/>p[0] = y                                        |
| Pointer indexing<br/>int i, \*p;                     | x = p[i];<br/>p[i+1] = y;                | x = p[i]<br/>p[i+1] = y                                      |
| Array indexing<br/>int i, a[];                       | x = a[i];<br/>a[i+1] = y;                | x = a[i]<br/>a[i+1] = y                                      |
| struct/union dereference<br/>struct foo s;           | x = s.field;<br/>s.field = y;            | x = s.field<br/>s.field = y                                  |
| struct/union pointer deref<br/>struct foo \*sp;      | x = sp-\>field;<br/>sp-\>field = y;      | x = sp.field<br/>s.field = y                                 |
| Pointer arithmetic<br/>int i, \*p;                   | x = p + i;<br/>y = p - i;                | x = p + i<br/>y = p - i                                      |
| Pointer dereference<br/>int \*p1, \*p2;              | x = p1 - p2;                             | x = p1 - p2                                                  |
| Array element pointer<br/>int i, a[];                | x = &a[i];                               | x = a + i                                                    |
| Cast pointer to address<br/>int \*p;                 | x = (intptr_t)p;                         | x = tonumber(ffi.cast("intptr_t", p))                        |
| Functions with outargs<br/>void foo(int \*inoutlen); | int len = x;<br/>foo(&len);<br/>y = len; | local len = ffi.new("int[1]", x)<br/>foo(len)<br/>y = len[0] |
| Vararg conversions<br/>int printf(char \*fmt, ...);  | printf("%g", 1.0);<br/>printf("%d", 1);  | printf("%g", 1)<br/>printf("%d", ffi.new("int"), 1)          |

## 内存管理
`cdata = ffi.gc(cdata, finalizer)`
- 将终结器(`finalizer`)与指针或集合体(`cdata`)关联.
- 原封不动返回`cdata`.
- 这个函数允许将未托管的资源纳入LuaJIT垃圾收集器的自动内存管理中.
- 使用例:
  ``` lua
  local p = ffi.gc(ffi.C.malloc(n), ffi.C.free)
  -- ...
  p = nil  -- p最后的引用被释放
  -- GC最终会运行终结器: ffi.C.free(p)
  ```
- `cdata`终结器与`userdata`对象的`__gc`元方法工作方式相像:
  当`cdata`对象的最后一个引用被释放时，相关联的终结器会被调用，并以`cdata`
  对象作为传入实参。终结器可以是一个Lua函数、一个`cdata`函数抑或一个`cdata`
  函数指针。一个已经存在的终结器可以通过设为`nil`来移除，例如在显示释放资源
  之前:
  ``` lua
  ffi.C.free(ffi.gc(p, nil))  -- 手动释放内存
  ```
