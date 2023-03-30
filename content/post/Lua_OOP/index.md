---
title: "Lua: 面向对象"
date: 2022-10-20
lastmod: 2022-10-23
draft: false
tags: ["Lua", "OOP"]
categories: ["Language"]

toc: false

---


# 实现
* **对象**借助**元表**实现, **实例化**即为将元表赋予存放字段的表
* 设置`__index`元方法为自身(`Object.__index = Object`):  
  实例化出的对象实例可以通过对象元表访问类方法, 也可访问元方法,
  类方法可以通过`self`标识取到调用者的引用.
* 目前的[LSP](https://github.com/sumneko/lua-language-server)不能检索元表,
  需要在[注释](https://keplerproject.github.io/luadoc/)中说明.
* 实例由字段表与类元表构成, 方法储存在元表中
* `Object:func()`与`Object.func(self)`等价,
  `self`可以隐藏传入调用者参数(相当于一些语言的`this`指针)


# 使用例
` Vector3d`实现
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
