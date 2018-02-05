---
title: 浅谈匿名函数及Lambda
date: 2016/7/11 22:22:03 
tags: 
     - 匿名函数
     - Lambda
     - 学习笔记
---
## 匿名对象
### 初始化方法

``` csharp
	var obj = new { Name = "cfmy",Sex = "M"} ;//单个匿名对象  
	
	var objs = new[]
	   {
	       new { Name = "cfmy1", Sex = "M" },
	       new { Name = "cfmy2", Sex = "M" },
	       new { Name = "cfmy3", Sex = "M" },
	       new { Name = "cfmy4", Sex = "M" },
	       new { Name = "cfmy5", Sex = "M" }
	   };//匿名对象数组；注意 属性顺序，数量，类型都必须一致，否则提示找不到隐式数组的最佳类型

	var objCobj = new { Name = "cfmy", self = new { Name = "cfmy" } }; //嵌套匿名对象
```
初始化一个匿名对象之后，编译器会生成一个运行时类型(System.RunTimeType)，它是受保护的，它会根据初始化时的属性顺序，数量，类型构造出一个看不到的类型，它的所有属性都是只读的，只能在实例化时赋值
<!--more-->
### 特性
1. 匿名对象只有属性，没有方法
2. 同一个程序集中，如果匿名对象的属性名称、顺序、数量完全相同，那么他们的类型(Type)也是相同的
![](http://7xr9ql.com1.z0.glb.clouddn.com/6-1.png)

### 适用场景
1. 当你想创建一个只用一次的对象，只想用的属性，而不用其方法
2. 上一点的具体化：你的数据源A、B、C三个字段，如果现在你在和另一方进行数据对接，他们需要要A、D两个字段，D是由B和C进行拼接或者某种逻辑处理而成，你现在需要快速创建出新的类型，但是这个类型在你们这个系统是无意义的，不想为其创建Class，那么匿名对象是你的不二选择
![](http://7xr9ql.com1.z0.glb.clouddn.com/6-2.png)

## 匿名函数和Lambda表达式
了解匿名函数和Lambda表达式前首先需要按顺序了解如下概念

方法=》委托=》匿名委托=》匿名函数=》Lambda表达式

### 委托
根据匿名对象例子简单改进成利用委托ConvertList，具体代码如下
``` csharp
	class Person
    {
        public int Age { get; set; }
        public string Name { get; set; }

    }
	       
    class PersonConverted
    {
        public string Name { get; set; }
        public bool IsAdult { get; set; }
    }

    class Test
    {
        public void DoSomething()
        {
            var person = new Person { Name = "cfmy", Age = 10 };
            var list = new List<Person>() { person };

            var convertedList1 = list.ConvertAll(Converter);//直接传递方法

            Converter<Person, PersonConverted> converter = Converter;//Converter是System自带委托类型
            var convertedList2 = list.ConvertAll(converter);//传递委托实例
        }
        PersonConverted Converter(Person person)
        {
            return new PersonConverted { Name = person.Name, IsAdult = person.Age > 18 };
        }
    }
``` 
其中ConvertAll反编译代码如下
``` csharp
	public List<TOutput> ConvertAll<TOutput>(Converter<T,TOutput> converter) {
        if( converter == null) {
            ThrowHelper.ThrowArgumentNullException(ExceptionArgument.converter);
        }
        // @


        Contract.EndContractBlock();

        List<TOutput> list = new List<TOutput>(_size);
        for( int i = 0; i< _size; i++) {
            list._items[i] = converter(_items[i]);
        }
        list._size = _size;
        return list;
    }
``` 
### 匿名委托
可以看出ConvertAll接收一个委托类型Converter<T,TOutput>。这意味着你可以传递该委托类型的函数或函数组，如果在传递委托函数的基础上简化一步代码，可以使用匿名委托，代码如下：
``` csharp
	var convertedList3 = list.ConvertAll(new Converter<Person, PersonConverted>(Converter));//匿名委托 + 显式函数
``` 
如果用了匿名委托，就只能接收一个函数作为参数，这点需要注意，如果你想使用函数组，好像只能自己实例化一个委托，然后添加多个函数进去(笔者自己还无法确定这点)

### 匿名函数
上个例子是匿名委托 + 显式函数，意味着我们还是要实现一个函数，使用匿名函数的话代码可以进一步精简代码如下：
``` csharp
	var convertedList4 = list.ConvertAll(new Converter<Person, PersonConverted>(delegate (Person p) { return new PersonConverted { Name = p.Name, IsAdult = p.Age > 18 }; }));//匿名委托 + 匿名函数
``` 
#### 匿名函数模型
*匿名函数*大致模型如下

	delegate ([入参类型] [入参形参])//可以没有入参
	{
		//方法体
		return [返回值];//可以没有返回
	}

了解模型后还是可以比较容易理解，但是我们了解到，传递委托的地方往往可以直接传递满足委托类型的函数，匿名也是同理，所以在需要传递委托的地方直接传递匿名函数的代码如下：
``` csharp
	var convertedList5 = list.ConvertAll(delegate (Person p) { return new PersonConverted { Name = p.Name, IsAdult = p.Age > 18 }; });//匿名函数
``` 

但是在上个例子中还是用到了PersonConverted对象，如果再利用之前的匿名对象进行优化，可以得到如下代码：
``` csharp
	var convertedList6 = list.ConvertAll(delegate (Person p) { return new  { Name = p.Name, IsAdult = p.Age > 18 }; });//匿名函数 + 匿名对象
``` 
### Lambda表达式
如果仅仅利用匿名的话，这已经是代码精进的极限了，这里我们既不需要建立委托类型，也不要建立返回类型Class，**但是**，Lambda表达式竟然还可以精简一步，先看下最终代码如下：
``` csharp
	var convertedList7 = list.ConvertAll(p => new {Name = p.Name, IsAdult = p.Age > 18});//Lambda表达式
``` 
Lambda表达式实质是匿名函数的一个语法糖，它的完整模型如下

	([入参a],[入参b])=> 
	{
		//方法体
		return [返回值];//可以没有返回值(比较少见)
	}

其中没有入参的时候不能省略圆括号，只有一个入参的情况可以省略左侧圆括号；如果没有方法体的有返回值的时候右侧可以简化为*表达式*或*具体值*，具体如下

	(x)=> x*x;
	 x => x*x;
	() => 1;

### 整个例子流程
代码如下：
``` csharp
	    class Person
        {
            public int Age { get; set; }
            public string Name { get; set; }

        }

        class PersonConverted
        {
            public string Name { get; set; }
            public bool IsAdult { get; set; }
        }

        class Test
        {
            public void DoSomething()
            {
                var person = new Person { Name = "cfmy", Age = 10 };
                var list = new List<Person>() { person };

                var convertedList1 = list.ConvertAll(Converter); //直接传递函数

                Converter<Person, PersonConverted> converter = Converter; //Converter是System自带委托类型
                var convertedList2 = list.ConvertAll(converter); //传递委托实例

                var convertedList3 = list.ConvertAll(new Converter<Person, PersonConverted>(Converter)); //匿名委托 + 显式函数

                var convertedList4 =
                    list.ConvertAll(
                        new Converter<Person, PersonConverted>(
                            delegate (Person p) { return new PersonConverted { Name = p.Name, IsAdult = p.Age > 18 }; }));
                //匿名委托 + 匿名函数

                var convertedList5 =
                    list.ConvertAll(
                        delegate (Person p) { return new PersonConverted { Name = p.Name, IsAdult = p.Age > 18 }; });
                //匿名函数

                var convertedList6 =
                    list.ConvertAll(delegate (Person p) { return new { Name = p.Name, IsAdult = p.Age > 18 }; });
                //匿名函数 + 匿名对象

                var convertedList7 = list.ConvertAll(p => new { Name = p.Name, IsAdult = p.Age > 18 }); //Lambda表达式

            }
            PersonConverted Converter(Person person)
            {
                return new PersonConverted { Name = person.Name, IsAdult = person.Age > 18 };
            }
        }
``` 
### 适用场景
1. 匿名方法 => 所有需要传递委托的地方
2. Lambda表达式=>从某个集合中过滤符合特定条件的集合
## 总结
Lambda表达式虽然大家可能都听说过，但是其实就是一个匿名方法的语法糖，如果你理解匿名方法的话，稍微看一下几种简化形式就可以完全掌握；相反，如果你连委托都不熟悉的话，那么就是任重而道远了

关于Lambda表达式使用场景的话写的不是很好，因为自己也用的不多，以后还会继续补充，另外，如果只是从集合中过滤元素的话，还是LINQ的from in where select 语法组更加优美！
	