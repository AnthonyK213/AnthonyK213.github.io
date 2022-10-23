---
title: "Lua笔记"
date: 2022-10-20
lastmod: 2022-10-23
draft: false
tags: ["Lua"]
categories: ["Language"]

toc: false

---


# 面向对象
* 实现
  * **对象** 借助 **元表** 实现, **实例化** 即为将元表赋予存放字段的表
    - 设置`__index`元方法为自身(`Object.__index = Object`):  
      实例化出的对象实例可以通过对象元表访问类方法, 也可访问元方法,
      类方法可以通过`self`标识取到调用者的引用.
    - 目前的[LSP](https://github.com/sumneko/lua-language-server)不能检索元表, 需要在[注释](https://keplerproject.github.io/luadoc/)中声明
  * 实例由字段表与类元表构成, 方法储存在元表中
* `Object:func()`与`Object.func(self)`等价,
  `self`可以隐藏传入调用者参数(相当于一些语言的`this`指针)
* 例: `Vector3d`实现
  ``` lua
  ---@class Vector3d
  ---@field x number
  ---@field y number
  ---@field z number
  local Vector3d = {}

  Vector3d.__index = Vector3d

  ---Constructor
  ---@param x number
  ---@param y number
  ---@param z number
  ---@return Vector3d
  function Vector3d.new(x, y, z)
      local vector = {
          x = x,
          y = y,
          z = z,
      }
      setmetatable(o, Vector3d)
      return vector
  end

  ---Addition
  ---@param a Vector3d
  ---@param b Vector3d
  ---@return Vector3d
  Vector3d.__add = function (a, b)
      return Vector3d.new(a.x + b.x, a.y + b.y, a.z + b.z)
  end

  ---Subtraction
  ---@param a Vector3d
  ---@param b Vector3d
  ---@return Vector3d
  Vector3d.__sub = function (a, b)
      return Vector3d.new(a.x - b.x, a.y - b.y, a.z - b.z)
  end

  ---Dot product
  ---@param a Vector3d
  ---@param b Vector3d
  ---@return number
  Vector3d.__mul = function (a, b)
      return a.x * b.x + a.y * b.y + a.z * b.z
  end

  ---Cross product
  ---@param a Vector3d
  ---@param b Vector3d
  ---@return Vector3d
  Vector3d.__concat = function (a, b)
      return Vector3d.new(
      a.y * b.z - a.z * b.y,
      a.z * b.x - a.x * b.z,
      a.x * b.y - a.y * b.x)
  end

  ---Negation
  ---@param a Vector3d
  ---@return Vector3d
  Vector3d.__unm = function (a)
      return Vector3d.new(-a.x, -a.y, -a.z)
  end

  ---Length
  ---Lua5.2支持
  ---@return number
  Vector3d.__len = function (self)
      return math.sqrt(self.x ^ 2 + self.y ^ 2 + self.z ^ 2)
  end

  ---Equal
  ---@param a Vector3d
  ---@param b Vector3d
  ---@return boolean
  Vector3d.__eq = function (a, b)
      return a.x == b.x and a.y == b.y and a.z == b.z
  end

  ---To string
  ---@param self Vector3d
  ---@return string
  Vector3d.__tostring = function (self)
      return string.format("Vector3d[%f, %f, %f]", self.x, self.y, self.z)
  end

  ---Clone
  ---@return Vector3d
  function Vector3d:clone()
      return Vector3d.new(self.x, self.y, self.z)
  end

  ---Zero
  ---@return Vector3d
  function Vector3d.zero()
      return Vector3d.new(0, 0, 0)
  end

  ---Unitize
  ---@return boolean
  function Vector3d:unitize()
      local length = #self
      if length > 0 then
          self.x = self.x / length
          self.y = self.y / length
          self.z = self.z / length
          return true
      end
      return false
  end

  --#region TEST
  local vec_0 = Vector3d.new(0, 1, 2)
  local vec_1 = Vector3d.new(-3, 5, 1)
  local vec_2 = vec_0:clone()

  assert(vec_0 + vec_1 == Vector3d.new(-3, 6, 3), "Addition test.")
  assert(vec_0 - vec_1 == Vector3d.new(3, -4, 1), "Subtraction test.")
  assert(vec_0 * vec_1 == 7, "Dot product test.")
  local vec_z = vec_0 .. vec_1
  assert(vec_z * vec_0 == 0 and vec_z * vec_1 == 0, "Cross product test.")
  assert(vec_0 .. vec_0 == Vector3d.zero(), "Zero test.")
  vec_2:unitize()
  assert(#vec_2, "Unitize test.")
  assert(not Vector3d.zero():unitize(), "Zero unitize test.")
  assert(-vec_0 + vec_0 == Vector3d.zero(), "Negation test.")

  print(vec_0, vec_1)

  vim.pretty_print("Vector3d", Vector3d)

  print("Vector3d test passed!")
  --#endregion
  ```

# 协程

## `thread` & `coroutine`
* 线程(`thread`):  
  每一个线程都代表一个执行序列.  
  当我们在程序中创建多线程的时候，看起来，同一时刻多个线程是同时执行的，不过
  实质上多个线程是并发的，因为只有一个CPU，所以实质上同一个时刻只有一个线程在
  执行。  
  在一个时间片内执行哪个线程是不确定的，我们可以控制线程的优先级，不过真正的线程
  调度由CPU调度决定

* 协程(`coroutine`):  
  协程跟线程都代表一个执行序列。不同的是，协程把线程中不确定的地方尽可能地去掉，
  执行序列间的切换不再由CPU隐藏地进行，而是由程序 **显式** 地进行。
  所以，使用协程实现并发，需要多个协程彼此协作。

## Basics
| Method              | Description                                              |
|---------------------|----------------------------------------------------------|
| coroutine.create()  | 创建协程, 返回`thread`对象，参数为一个函数，`resume`唤醒 |
| coroutine.resume()  | 重启协程, 和`create`配合使用                             |
| coroutine.yield()   | 挂起协程, 将协程设置为挂起状态，和`resume`配合使用       |
| coroutine.status()  | 查看协程的状态(normal, dead, suspend, running)           |
| coroutine.wrap()    | 创建协程, 返回一个函数，调用此函数即进入这个`coroutine`  |
| coroutine.running() | 用来判断当前执行的协程是不是主线程，如果是，则返回`true` |

* 如果协程`co`的函数执行完毕，协程正常终止，`resume`返回`true`和函数返回值;  
  如果协程`co`的函数执行过程中，协程让出了(调用了`yield`方法)，那么`resume`返回`true`
  和协程中调用`yield`传入的参数;  
  如果协程`co`的函数执行过程中发生错误，`resume`返回`false`与错误消息.
  > `create`是在保护模式下进行的，`wrap`不是
* 传递给`yield`的参数会作为`resume`的额外返回值
* 如果对该协程不是第一次执行`resume`, `resume`函数传入的参数将会作为`yield`的返回值
* 例： 生产者-消费者模式
  ``` lua
  local producer = coroutine.create(function ()
      for i = 1, 10 do
          coroutine.yield(i)
      end
  end)

  local consumer = coroutine.create(function ()
      while true do
          local ok, i = coroutine.resume(producer)
          if ok then
              print(i)
          else
              print("Coroutine is dead.")
              break
          end
      end
  end)

  coroutine.resume(consumer)
  ```

## `async`/`await`
* 使用`coroutine`与`libuv(vim.loop)`[实现](https://github.com/AnthonyK213/nvim/blob/master/lua/utility/task.lua)
  ``` lua
  ---@class 任务 任务对象.
  ---@field action function 需要异步执行的函数.
  ---@field callbacks function[] 回调函数队列, 接收异步执行结果为参数.
  ---@field varargs any[] 异步执行函数传入参数.
  ---@field status "Created"|"Running"|"RanToCompletion" 任务运行状态.
  ---@field result any 异步执行结果.
  local Task = {}

  Task.__index = Task

  ---构造函数
  ---@param action function 需要异步执行的函数.
  ---@param ... any 异步执行函数传入参数.
  ---@return 任务 Task实例.
  function Task.new(action, ...)
      local task = {
          action = action,
          callbacks = {},
          varargs = { ... },
          status = "Created",
      }
      setmetatable(task, Task)
      return task
  end

  ---添加回调函数.
  ---@param callback function 回调函数.
  function Task:append_cb(callback)
      table.insert(self.callbacks, callback)
  end

  ---运行任务.
  ---@return boolean ok 如果任务线程启动成功则True, 反之False.
  function Task:start()
      return vim.loop.new_work(self.action, vim.schedule_wrap(function(r)
          for _, f in ipairs(self.callbacks) do  -- 执行队列中的回调函数.
              if type(f) == "function" then
                  f(r)
              end
          end
      end)):queue(unpack(self.varargs))
  end

  ---等待任务完成.
  ---@return any result 任务运行结果.
  function Task:await()
      local _co = coroutine.running()  -- 获取正在运行的协程.
      if not _co or coroutine.status(_co) == "dead" then
          error("Task must await in an alive async block.")  -- 协程需要有效.
      end
      if self.status == "Created" then
          self:append_cb(function(r)           -- 为任务添加回调函数, 在结束时
              self.result = r                  -- 将结果赋予result字段, 并切回
              self.status = "RanToCompletion"  -- 主协程, 继续执行主协程代码.
              coroutine.resume(_co)
          end)
          if self:start() then                 -- 启动任务, 并挂起当前协程, 当前
              self.status = "Running"          -- 任务让出对主线程的占用并在其它
              coroutine.yield()                -- 线程运行. 主协程等待任务完成时
              return self.result               -- 回调函数重新运行主协程.
          end
      end
  end

  ---运行异步代码块.
  ---@param async_block function 异步代码块.
  local async = function(async_block)
      local _co = coroutine.create(async_block)
      coroutine.resume(_co)
  end

  --#region Test
  vim.cmd.messages("clear")

  local fibonacci = Task.new(function(n)
      local function fib(x)
          if x <= 2 then
              return 1
          else
              return fib(x - 1) + fib(x - 2)
          end
      end
      return fib(n)
  end, 42)

  async(function()
      print("Begin")                    -- 未运行至await, 打印 "Begin".
      local result = fibonacci:await()  -- 协程挂起, 打印 "EOF".
      print(result)                     -- 协程切回, 打印 "267914296".
      print("End")                      -- 打印 "End", 异步代码块执行完成.
  end)

  print("EOF")
  --#endregion
  ```


# LuaJIT

> LuaJIT is a Just-In-Time Compilerfor the Lua programming language.

* 为了尽可能走JIT, 尽量避免使用未支持JIT的Lua原语与库函数([NYI](http://wiki.luajit.org/NYI))

## FFI
[FFI_Library](https://luajit.org/ext_ffi.html)
* 使用动态链接库
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
* 使用C数据结构
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
* 内存管理
* Lua与C语法对应关系

| Idiom                      | C code        | Lua code     |
|----------------------------|---------------|--------------|
| Pointer dereference        | x = *p        | x = p[0]     |
| int *p                     | *p = y        | p[0] = y     |
| Pointer indexing           | x = p[i]      | x = p[i]     |
| int i, *p                  | p[i+1] = y    | p[i+1] = y   |
| Array indexing             | x = a[i]      | x = a[i]     |
| int i, a[]                 | a[i+1] = y    | a[i+1] = y   |
| struct/union dereference   | x = s.field   | x = s.field  |
| struct foo s               | s.field = y   | s.field = y  |
| struct/union pointer deref | x = sp->field | x = sp.field |
| struct foo *sp             | sp->field = y | s.field = y  |
| int i, *p                  | y = p - i     | y = p - i    |
| Pointer dereference        | x = p1 - p2   | x = p1 - p2  |
| Array element pointer      | x = &a[i]     | x = a + i    |


# 杂记
* 三目运算
  - `A and B or C`
    > 注意: 当B为`false`或`nil`时, 结果永远为`C`.
* 位运算
  - 由于Lua(5.1)本身只有`number(double)`类型, 所以原生不支持整型的位运算,
    但LuaJIT可使用位运算扩展库[BitOp](https://bitop.luajit.org/)
