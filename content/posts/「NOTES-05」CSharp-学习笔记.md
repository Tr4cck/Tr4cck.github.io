---
title: 「C#」学习随笔
date: 2021-08-19 23:06:35
tags: CS
categories: TECHNOLOGY
mathjax: true
toc: true
---


## Summary-About-101

- params int[] array

- public delegate double MyDelegate (double p1, double p2);

	MyDelegate d = ...

- To stop the process: System.Console.ReadKey(); <=> getchar();

- int? a;

## 异常处理

异常处理结构如下：

```c#
try {
    ...
}
catch (<ExceptionType> e) {
    ...
}
finally {
    ...
}
```

**注意：**catch块和finally块必须包含其中之一，且两者可以同时存在。catch块可以有多个，finally块0或1个。

其中执行顺序与python一致。

catch块不写的话，表示捕捉任意异常。

例子如下：

```c#
try {
    int[] array = {1, 2, 3, 4};
    int element = array[4];
}
catch (IndexOutOfRangeException e) {
    Console.WriteLine("catch");
}
finally {
    Console.WriteLine("finally");
}
```

## OOP

### 类的定义与声明

在C#中，同样存在未来要创建的实例指针`this`，我们可以通过它在类的内部给参数赋值

### 构造函数

**构造函数可以有多个，它们的不同只能在参数列表上体现**

#### 派生类的构造函数

如何在子类中调用父类的构造函数？有两种情况

1. 无参

	```c#
	using System;
	
	namespace Main {
	    public class Father {
	        public Father() {
	            Console.WriteLine("Father's constructor with no parameters.");
	        }
	    }
	    
	    public class Son : Father {
	        public Son() : base() {
	            Console.WriteLine("Son's constructor with no parameters.");
	        }
	    }
	}
	```

2. 有参

	```c#
	using System;
	
	namespace Main {
	    public class Father {
	        private int x;
	        public Father(int x) {
	            this.x = x;
	            Console.WriteLine("Father's constructor with one parameters.");
	        }
	    }
	    
	    public class Son : Father {
	        private int x, y;
	        public Son(int x, int y) : base(x) {
	            this.y = y;
	            Console.WriteLine("Son's constructor with two parameters.");
	        }
	    }
	}
	```

**这两种情况都会先调用父类的构造函数，再调用子类的构造函数**

### 属性

#### 用法

见源码与[官方文档](https://docs.microsoft.com/zh-cn/dotnet/csharp/programming-guide/classes-and-structs/properties)

主要关注特殊的用法，例如`new{X = ..., Y = ...}`等等

还有就是由属性可以自动生成相应的字段

#### 特性

使用特性可以有效将元数据或声明性信息与代码（程序集、类型、方法、属性等）相关联。将特性与程序实体相关联后，可以在运行时使用**反射**技术查询特性。

特性具有以下属性：

- 特性向程序添加元数据。 **元数据** 是程序中定义的类型的相关信息。 所有`.NET`程序集都包含一组指定的元数据，用于描述程序集中定义的类型和类型成员。 可以添加自定义特性来指定所需的其他任何信息。 有关详细信息。
- 可以将一个或多个特性应用于整个程序集、模块或较小的程序元素（如类和属性）。
- 特性可以像方法和属性一样接受自变量。
- 程序可使用反射来检查自己的元数据或其他程序中的元数据。

### 虚函数与抽象类

#### 注意

虚函数：

- 类型声明为父类的实例可以调用子类的构造方法，这样就可以实现**父类实例调用子类中的重载方法**
- 不能用父类的构造函数声明子类实例
- 如果不override，那么使用父类的virtual方法

抽象类：

- 不能实例化
- 类和方法都要加上`abstract`关键字
- 在子类中override后，可以用子类的构造函数创建抽象父类的实例，调用时等价于调用子类的相应方法

#### 关于隐藏方法

使用`new`修饰符配合，只需要知道**与虚函数的交错声明相反**即可，因为这个东西个人感觉比较混乱，应该不太会在项目中用到

### 密封类

#### 定义

C#允许把类和方法加上`sealed`关键字。对于类，表示其他类不能继承自该类，对于方法，表示不能重载该方法

#### 注意

只有重载的方法才可以被声明为sealed

### 修饰符

|       修饰符       |          应用于          |                             说明                             |
| :----------------: | :----------------------: | :----------------------------------------------------------: |
|       public       |       所有类或成员       |                    任何代码均可以访问该项                    |
|     protected      | 该类和内嵌于类的所有成员 |    与private不同之处在于：它修饰的东西可以在派生类能访问     |
|      internal      |       所有类或成员       |                  只能在包含它的程序集中访问                  |
|      private       | 该类和内嵌于类的所有成员 |                    只能在它所属的类内访问                    |
| protected internal | 该类和内嵌于类的所有成员 |              只有派生类和在包含它的程序集中访问              |
|       static       |         所有成员         |                   成员不作用于类的具体实例                   |
|        new         |         函数成员         |                          见隐藏方法                          |
|      override      |        仅函数成员        |                成员重写了继承的虚拟或抽象成员                |
|      virtual       |        仅函数成员        |                           见虚方法                           |
|       extern       |        仅静态方法        |                  成员在外部以另一种语言实现                  |
|       sealed       |      类、方法、属性      |                           见密封类                           |
|      abstract      |         函数和类         |                           见抽象类                           |
|      readonly      |           字段           | 构造函数退出后，不能分配 `readonly` 字段。 此规则对于值类型和引用类型具有不同的含义 |

（修饰符比较多，遇到一个学习一个即可）。

注意一下几点：

- static声明的类里面必须全是public

### 接口

使用`public interface xxx { public void xxxx( ... ); }`来定义一个接口，这与抽象类有极大的相似度。

#### 接口与类的关系

接口的实现要具体到类，即**实现**是接口与类之间的关系。结构如下：

```c#
public class What : AInterface {
    ...
}
```

#### 注意

- 只能包含方法、属性、索引器和事件的声明。

- 一个接口中不能有字段、构造函数。

- 接口内不允许运算符重载。

- 接口中的成员都是public的。

#### 接口之间的关系

也可以继承

### 泛型

#### 集合类List\<T>

首先是如何实例化，有如下两种方法：

```c#
// 1 空列表 后续用Add方法添加值
List<int> scorelist = new List<int>();
// 2 有初始值
List<int> scorelist = new List<int>() {1, 2, 3};
```

一般声明的类型用`var`

列表的Add方法类似python中的append

注意区分capacity和count，前者表示list被分配的大小，也等价于底层数组的`Length`属性，后者表示其元素个数。并且它分配的逻辑也比较有趣，空列表加元素时，会先将capacity扩大到4，直到count为5，capacity置为8，以此类推

然后还有一些方法，不写了，能记住一些常用的，其它要用再查就好了。

#### 泛型类定义

意思是类中某些字段的类型是不确定的，并且要在类构造的时候确定下来，例如：

```c#
using System;

namespace GenericThings
{
    class Program
    {
        static void Main(string[] args)
        {
            var a = new ClassA<string>{A="www.", B="trackonyou.com"};
            Console.WriteLine(a.GetSum());
        }
    }
    public class ClassA<T> {
        public T A {get; set;}
        public T B {get; set;}
        public string GetSum()
        {
            return A + "" + B;
        }
    }
}
```

#### 泛型方法定义

与上面类似，就是指的方法的参数不定（**注意不是返回值不定**），直到调用时才被确定下来

**PS：**可以在`<T, U>`里面这样，包含多种类型

### MyList2Practice

描一个轮子

#### 官方实现

```c#
// 首先是几个变量，一个默认容量为4
private const int DefaultCapacity = 4;
// _items是一个底层数组，是这个类的核心
internal T[] _items;
// _size私有变量，是元素个数
internal int _size;
// TODO
private int _version;
// 提前定义一个私有的空数组
private static readonly T[] s_emptyArray = new T[0];

// 接着是构造函数，有三个，主要分析前两个，第三个暂时看不太懂😂
// 无参的默认new空数组
public List()
{
    _items = s_emptyArray;
}
// 传递参数为分配空间
public List(int capacity)
{
    if (capacity < 0)
        ThrowHelper.ThrowArgumentOutOfRangeException(ExceptionArgument.capacity, ExceptionResource.ArgumentOutOfRange_NeedNonNegNum);

	if (capacity == 0)
        _items = s_emptyArray;
	else
        _items = new T[capacity];
}
// 分配空间属性
public int Capacity
{
    // get方法用表达式主体的方式，然后动态获取数组的Length，这样比较智能
    get => _items.Length;
    // set方法，首先判断预期值与当前元素个数的大小关系，如果前者小于后者则触发异常；然后处理预期值不等于当前容量的情况，大于就重新分配一个临时数组，并把原来的东西复制过去，同时改变一下_items的指向，如果小于等于0就分配空数组
    set
    {
        if (value < _size)
        {
            ThrowHelper.ThrowArgumentOutOfRangeException(ExceptionArgument.value, ExceptionResource.ArgumentOutOfRange_SmallCapacity);
        }

        if (value != _items.Length)
        {
            if (value > 0)
            {
                T[] newItems = new T[value];
                if (_size > 0)
                {
                    Array.Copy(_items, newItems, _size);
                }
                _items = newItems;
            }
            else
            {
                _items = s_emptyArray;
            }
        }
    }
}
// 设置Count为get访问器，返回个数
public int Count => _size;

public void Add(T item)
{
    _version++;
    // 保存当前数组的状态
    T[] array = _items;
    int size = _size;
    // 如果有多余空间
    if ((uint)size < (uint)array.Length)
    {
        _size = size + 1;
        array[size] = item;
    }
    // 没有的话就resize
    else
    {
        AddWithResize(item);
    }
}

private void AddWithResize(T item)
{
    // 一样，保存一下size，底层数组以参数形式传递进来
    int size = _size;
    // 调用函数改容量，然后下文就是一样操作
    EnsureCapacity(size + 1);
    _size = size + 1;
    _items[size] = item;
}
// 下界是size+1，因为我们要add一个元素
private void EnsureCapacity(int min)
{
    // 处理容量小于下界的情况
    if (_items.Length < min)
    {
        int newCapacity = _items.Length == 0 ? DefaultCapacity : _items.Length * 2;
        // Tr4ck: internal const int MaxArrayLength = 0X7FEFFFFF;
        if ((uint)newCapacity > Array.MaxArrayLength) newCapacity = Array.MaxArrayLength;
        if (newCapacity < min) newCapacity = min;
        Capacity = newCapacity;
    }
}

```

### 反射类型

可以通过反射的方法用类的实例获取类型对象的实例，也就是获取到了实例的类型信息

## LINQ

### Sequences-operation

直接来个例子

```c#
var wordA = new string[] {"blackbird", "deebato", "cyxq"};
var wordB = new string[] {"blackbird", "d3ebato", "cyxq"};

bool IsMatch = wordA.SequenceEqual(wordB);

Console.WriteLine($"Is equal or not? >>> {IsMatch}");
```

### TakeWhile

传入的参数为`lambda`表达式例如`n => n < 6`，返回一个布尔判断式

### `Skip()-Take()`

它们都是`IEnumerable<T>`接口的扩展方法，其中前者表示从指定下标开始截取（包含），后者表示截取到指定下标为止（不包含）。

用这两个函数配合使用可以轻易实现获取列表的子列。

**值得注意的是：**它们都不会对越界操作甚至负操作有任何的报错，而且结果也很符合预期。这与另一个实现该功能的函数不尽相同，`SubString()`，它对于越界的索引都会抛出异常。

### `Zip()`

类似python中的zip函数，即将两个序列进行元组级的组合。C#使用时对其中任一序列调用该方法，传入的第一个参数是另一个序列，第二个参数是`lambda表达式`，返回组合后希望进行的运算。调用Zip方法将得到一个序列，可以直接在后面调用Sum方法之类的。

### `Sum()`

字面意思，对一个序列进行`Sum`操作。

### Select-from-multiple-input-sequences

类似sql语句的东西。注意如果两个from前后出现，则类似双重循环。这个语句类似函数，定义的时候并不会直接执行，必须在循环里遍历对应的变量，才能实现相应的功能。注意下select，它是返回满足选择条件的值，即where。但如果不写where语句，则它会无条件返回对应的值。这实际是一种懒加载，如果直接在后面加`ToList`方法，就可以完全变为字符串 

## Blazor

### App.razor

管理路由，并控制访问页面的内容

## System.Security.Cryptography
