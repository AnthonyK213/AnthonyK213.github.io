---
title: "Lua: 协程"
date: 2022-10-20
lastmod: 2023-01-13
draft: false
tags: ["Lua"]
categories: ["Language"]

toc: false

---


# 线程 & 协程
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


# 基本使用
| Method              | Description                                            |
|---------------------|--------------------------------------------------------|
| coroutine.create()  | 创建协程, 接收一个函数并返回`thread`对象，`resume`唤醒 |
| coroutine.resume()  | 重启协程, 和`create`配合使用                           |
| coroutine.yield()   | 挂起协程, 将协程设置为挂起状态，和`resume`配合使用     |
| coroutine.status()  | 查看协程的状态(`normal`, `dead`, `suspend`, `running`) |
| coroutine.wrap()    | 创建协程, 返回一个函数，调用此函数即进入此`coroutine`  |
| coroutine.running() | 返回当前协程, 若当前为主协程则返回`nil`                |

* 如果协程`co`的函数执行完毕，协程正常终止，`resume`返回`true`和函数返回值;  
  如果协程`co`的函数执行过程中，协程让出了(调用了`yield`方法)，那么`resume`返回`true`
  和协程中调用`yield`传入的参数;  
  如果协程`co`的函数执行过程中发生错误，`resume`返回`false`与错误消息.
  > `create`是在保护模式下进行的，`wrap`不是
* 传递给`yield`的参数会作为`resume`的额外返回值
* 如果对该协程不是第一次执行`resume`, `resume`函数传入的参数将会作为`yield`的返回值
* 例： **生产者-消费者** 模式
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


# `async`/`await`
使用`coroutine`与`libuv(vim.loop)`
[实现](https://github.com/AnthonyK213/nvim/blob/master/lua/futures/task.lua)
``` lua
---@class 任务 任务对象.
---@field action function 异步执行的代码.
---@field vararg any 入参.
---@field result any 异步执行结果.
---@field continuation function 回调函数, 参数为异步执行结果.
---@field is_completed boolean 任务是否完成.
local Task = {}

Task.__index = Task

---构造函数
---@param action function 异步执行的代码.
---@param vararg any 入参.
---@return 任务 task Task实例.
function Task.new(action, vararg)
    local task = {
        action = action;
        vararg = vararg;
        is_completed = false;
    }
    setmetatable(task, Task)
    return task
end

---运行任务.
---@return boolean ok 如果任务启动成功则 `true`, 反之 `false`.
function Task:start()
    self.handle = vim.loop.new_work(self.action, vim.schedule_wrap(self.continuation))
    return self.handle:queue(self.vararg)
end

---异步等待任务完成.
---@return any result 任务运行结果.
function Task:await()
    local co = coroutine.running()    -- 获取正在运行的协程.
                                      -- 归还线程池时唤醒此协程. 记为协程a.
    self.continuation = function(r)   -- 为任务添加回调函数(续延), 在结束时
        self.result = r               -- 将结果写入result字段, 并切回
        self.is_complete = true       -- 协程a, 继续执行协程a代码.
        coroutine.resume(co)
    end

    if self:start() then              -- 启动任务, 并挂起当前协程, 当前
        coroutine.yield()             -- 任务让出对主线程的占用并在其它
    end                               -- 线程运行. 任务完成时回调函数重
                                      -- 新启动协程a.
    return self.result
end

---阻塞等待任务完成.
---@return any result 任务运行结果.
function Task:wait()
    self.continuation = function(r)
        self.result = r
        self.is_complete = true
    end

    if self:start() then
        vim.wait(1e8, function()
            return self.is_complete
        end)
    end

    return self.result
end

---运行异步代码块.
---@param async_block function 异步代码块.
local spawn = function(async_block)
    local co = coroutine.create(async_block)
    coroutine.resume(co)
end

--------------------------------------------------------------------------------

vim.cmd.messages [[clear]]

local fibonacci = function(n)
    local function fib(x)
        if x <= 2 then
            return 1
        else
            return fib(x - 1) + fib(x - 2)
        end
    end

    return fib(n)
end

local fib_42 = Task.new(fibonacci, 42)

spawn(function()
    print("Begin")                    -- 未运行至await, 打印 "Begin".
    local result = fib_42:await()     -- 协程挂起, 打印 "EOF".
    print(result)                     -- 协程切回, 打印 "267914296".
    print("End")                      -- 打印 "End", 异步代码块执行完成.
end)

print("EOF")
```
