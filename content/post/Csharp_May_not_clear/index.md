---
title: "C#: 易模糊点"
date: 2022-10-24
lastmod: 2022-10-24
draft: false
tags: ["C#"]
categories: ["Language"]

toc: false

---


记录C#使用中一些可能不太清晰的点.


# 值类型与引用类型

## 传参
若不加`ref`关键字, 不论值类型还是引用类型均为**传值操作**, 其中值类型会传入
**值的副本**, 引用类型会传入**引用的副本**, 也就是对于引用类型来说:
* 引用类型传参相当于传入原指针的副本，使用这个指针可以访问与操作原指针指向的数据
  ``` cs
  namespace Test
  {
      public class Program
      {
          public static void Main()
          {
              A a = new A(0);               // 创建实例
              Console.WriteLine(a.Data);    // 输出: 0
              B(a);
              Console.WriteLine(a.Data);    // 输出: 1
              Console.ReadKey();
          }
  
          static void B(A a)
          {
              a.Data = 1;                   // 使用 `a` 的副本修改数据
          }
      }

      public class A
      {
          public int Data { get; set; }

          public A(int data)
          {
              Data = data;
          }
      }
  }
  ```
* 为这个引用副本重新赋值(重新指定指向的数据)不会影响原指针与原指针指向的数据
  (除非新指向的数据中又包含了原指向数据的引用, 然后又用这个引用操作原数据)
  ``` cs
  namespace Test
  {
      public class Program
      {
          public static void Main()
          {
              A a = new A(0);               // 创建实例
              Console.WriteLine(a.Data);    // 输出: 0
              B(a);
              Console.WriteLine(a.Data);    // 输出: 0
              Console.ReadKey();
          }

          static void B(A a)
          {
              a = new A(1);                 // 为传入的引用副本 `a` 赋值
              a.Data = 2;                   // 使用 `a` 修改其指向的数据
          }
      }

      // ...
  }
  ```
在Rust中, 传引用不会传引用的副本, 因为可能会产生多个可变引用; 若要克隆引用,
需使用带引用计数的智能指针, 显式调用`clone`方法.

## 线性集合使用下标检索元素
* `Array<T>`
  * `array[index]`的含义与C相近, 在C#中可视为引用, `index`代表从首元素指针的偏移
  * 不论`T`为值类型还是引用类型，`array[index].Method()`都可能会对`array`中偏移量`index`的元素产生影响;  
  * `array[index]`可作为`ref`参数;  
  * 但如果`T`为值类型且在调用前被**求值**, 则会获得值的副本.
* `List<T>`
  * `list[index]`是方法调用, 继承自`IList<T>`
    ``` cs
    // ...

    public object this[int index]
    {
        get
        {
            return _contents[index]; // 与Array不同，不论访问器的具体实现是什么,
                                     // 如果不是 `return ref` 就一定会产生副本,
                                     // 所以 `list[index]` 无法作为 `ref` 参数
                                     // (编译器会提示 "属性或索引器不能作为 `out`
                                     // 或 `ref` 参数传递")
        }
        set
        {
            _contents[index] = value;
        }
    }

    // ...
    ```
    <mark>注意</mark> 若`T`为值类型，用下标检索出的是**值的副本**, 已经经过**求值**


# 异常捕捉中的`finally`
总结为以下几点
* <u>`finally`块一定会执行</u>
  * 在`try`块或`catch`块中`return`或`throw`, `finally`块依然会被执行.
* `finally`是在`return`表达式运算完成之后执行的，在执行完`return`时，程序并没有
  跳出，而是进入到`finally`中继续执行. 如果在`finally`中对返回值进行了重新赋值，则
  * 当返回值是值类型(包括`string`类型，虽然它是引用类型)时，返回值不受影响；
  * 当返回值是引用类型时，会影响到返回值(如修改了返回的数组内的元素)
* `finally`中不能有`return`语句(控制不能离开`finally`子句主体)
