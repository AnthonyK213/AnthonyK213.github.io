---
title: "Lua: 杂记"
date: 2022-10-20
lastmod: 2022-10-23
draft: false
tags: ["Lua"]
categories: ["Language"]

toc: false

---


# 三目运算
- `A and B or C`
  > 注意: 当B为`false`或`nil`时, 结果永远为`C`.


# 位运算
- 由于Lua(5.1)本身只有`number(double)`类型, 所以原生不支持整型的位运算,
  但LuaJIT可使用位运算扩展库[BitOp](https://bitop.luajit.org/)
