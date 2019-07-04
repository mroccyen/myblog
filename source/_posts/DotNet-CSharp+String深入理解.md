---
title: 深入理解C#中的String
date: 2017-05-23 10:01:51
tags: 
- C#
- .Net
categories: 
- [C#]
- [Java]
- [database]
---

## 关于C#中的类型
在C#中类型分为值类型和引用类型，引用类型和值类型都继承自System.Object类,几乎所有的引用类型都直接从System.Object继承，而值类型具体一点则继承System.Object的子类，即继承System.ValueType。而String类型却有点特别，虽然它属于引用类型，但是他的一些特性却有点类似值类型。

## 关于C# String
### 1、不变性
我们先来看看一个例子：
```
static void Main(string[] args)
{
    string str1 = "string";
    string str2 = str1;
    Console.WriteLine(object.ReferenceEquals(str1, str2));
    str2 += "change";
    Console.WriteLine(object.ReferenceEquals(str1, str2));
    Console.ReadKey();
}
```

输出结果是True、False。为什么呢？我们来看看IL。
```
.entrypoint
  // 代码大小       48 (0x30)
  .maxstack  2
  .locals init ([0] string str1,
           [1] string str2)
  IL_0000:  nop
  IL_0001:  ldstr      "string"
  IL_0006:  stloc.0
  IL_0007:  ldloc.0
  IL_0008:  stloc.1
  IL_0009:  ldloc.0
  IL_000a:  ldloc.1
  IL_000b:  ceq
  IL_000d:  call       void [mscorlib]System.Console::WriteLine(bool)
  IL_0012:  nop
  IL_0013:  ldloc.1
  IL_0014:  ldstr      "change"
  IL_0019:  call       string [mscorlib]System.String::Concat(string,string) 
  IL_001e:  stloc.1
  IL_001f:  ldloc.0
  IL_0020:  ldloc.1
  IL_0021:  ceq
  IL_0023:  call       void [mscorlib]System.Console::WriteLine(bool)
  IL_0028:  nop
  IL_0029:  call       valuetype [mscorlib]System.ConsoleKeyInfo [mscorlib]System.Console::ReadKey()
  IL_002e:  pop
  IL_002f:  ret

```

+=在内部调用了Concat函数，将str2和"change"连接起来直接生成了一个新的字符串，和原来的字符串是不同的对象。Trim、Remove函数都是会直接生成一个新的对象，字符串一经定义，就不能改变。

其实字符串具有原子性（也就是不变性），任何改变字符串的值的行为都不会成功，只会创建一个新的字符串对象。在实际编程中，我们会大量的使用字符串，这样就会导致不停地创建新的字符串对象和分配内存，可能导致垃圾回收器GC不停地进行垃圾回收，大大降低性能，并且伴随着内存溢出的危险。所以.Net对字符串进行了的特殊的处理，这就是字符串驻留池。

在字符串驻留池，保存着字符串字面值和指向的引用。每次有新的字符串创建，都会在驻留池中查找是否存在字面值相同的字符串，如果存在就将其指向已经存在的字符串的引用，不存在就直接新建一个字符串，然后指向一个新的地址。

### 2、作为函数参数的处理
在函数的参数传递中，值类型直接拷贝变量保存的值，传递的是一个值得副本，而引用类型传递的是地址的一个副本，所以在函数中改变引用参数中属性的值会直接改变函数外真实类型对象的值。
```
static void Main(string[] args)
{
    People people = new People() { Name = "Jack" };
    Console.WriteLine(people.Name);
    Change(people);
    Console.WriteLine(people.Name);
    Console.ReadKey();
}

static void Change(People p)
{
    p.Name = "Eason";
}

class People
{
    public string Name { get; set; }
}
```

程序先输出Jack，后输出Eason，可以说明引用类型传递的是引用地址，函数改变的参数对象和外部传递进来的对象是一个对象。

那么我们来看看String作为参数的情况：
```
static void Main(string[] args)
{
    string str = "string";
    Console.WriteLine(str);
    Change(str);
    Console.WriteLine(str);
    Console.ReadKey();
}

static void Change(string str)
{
    str = "change";
    Console.WriteLine(str);
}
```

结果输出string、change、string。调用Change函数后str的值还是"string"，由于字符串类型的不变性，在Change函数中对str进行赋值会重新创建一个新的字符串对象，然后为这个新的对象附上引用。所以虽然字符串类型是引用类型，但是在参数传递时它其实相当于值类型。

### 3、相等比较处理
先看一个例子：
```
string str1 = "string";
string str2 = "string";
string str3 = "stringstring";
string str4 = "string" + "string";
string str5 = str1 + "string";
Console.WriteLine(ReferenceEquals(str1, str2));
Console.WriteLine(str1 == str2);
Console.WriteLine(ReferenceEquals(str3, str4));
Console.WriteLine(str3 == str4);
Console.WriteLine(ReferenceEquals(str3, str5));
Console.WriteLine(str3 == str5);
Console.ReadKey();
```

不出意外结果都应该为True，True，True，True，True，True，但是结果却是True，True，True，True，False，True，str3和str5不是一个对象，他们不是指向同一个地址，为什么呢？经过查看IL代码发现，str5在IL代码中调用了Concat函数将str1和"string"进行了拼接，那这个Concat函数到底做了什么。
```
public static string Concat(string str0, string str1)
{
    if (IsNullOrEmpty(str0))
    {
        if (IsNullOrEmpty(str1))
        {
            return Empty;
        }
        return str1;
    }
    if (IsNullOrEmpty(str1))
    {
        return str0;
    }
    int length = str0.Length;
    string dest = FastAllocateString(length + str1.Length);
    FillStringChecked(dest, 0, str0);
    FillStringChecked(dest, length, str1);
    return dest;
}
```

FastAllocateString函数负责分配长度为str0.Length+str1.Length的空字符串dest，FillStringChecked分别将str0和str1复制到dest中，最后生成由str0和str1连接成的字符串，这样不会再去字符串驻留池中查找是否存在和dest相同的字符串，而是直接生成一个新的对象。所以字符串变量和字符串常量进行拼接后会直接生成一个新的对象，绕过驻留池检查。

而字符串常量拼接不会产生新的字符串，除非驻留池中没有与之拼接后字面值相等的字符串。我们来看看IL代码：
```
  IL_0001:  ldstr      "string"
  IL_0006:  stloc.0
  IL_0007:  ldstr      "string"
  IL_000c:  stloc.1
  IL_000d:  ldstr      "stringstring"
  IL_0012:  stloc.2
  IL_0013:  ldstr      "stringstring"
  IL_0018:  stloc.3
  IL_0019:  ldloc.0
  IL_001a:  ldstr      "string"
  IL_001f:  call       string [mscorlib]System.String::Concat(string,string)
  IL_0024:  stloc.s    str5
  IL_0026:  ldloc.0
  IL_0027:  ldloc.1
```

str3和str4的字面值是相等的，都是"stringstring"，str3先于str4被初始化，当str4被初始化的时候，由于其字面值和str3相等，所以CLR会将str3指向的地址赋给str4，所以str3和str4引用是相等的。

至于"=="操作符的得到的结果都是True是因为"=="操作符会调用String.Equal方法，IL代码如下：
```
  IL_0032:  call       bool [mscorlib]System.String::op_Equality(string,string)
```
op_Equality最终会调用String.Equal函数，Equal函数的比较步骤是先比较两个对象的引用是否相等，不相等的话再对值进行比较，比较值时是按位比较的。