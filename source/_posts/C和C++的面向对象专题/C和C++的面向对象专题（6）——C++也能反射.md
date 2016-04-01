title: C和C++的面向对象专题（6）——C++也能反射
---

**本专栏文章列表**

一、何为面向对象

二、C语言也能实现面向对象

三、C++中的不优雅特性

四、解决封装，避免接口

五、合理使用模板，避免代码冗余

六、C++也能反射

七、单例模式解决静态成员对象和全局对象的构造顺序难题

八、更为高级的预处理器PHP

九、Gtkmm的最佳实践

本系列文章由 西风逍遥游 原创，转载请注明出处：西风广场 http://sunxfancy.github.io/


## 六、C++也能反射

今天我们来探讨C++的反射问题，缺乏反射机制一直是C++的大问题，很多系统的设计时，需要根据外部资源文件的定义，动态的调用内部的函数和接口，如果没有反射，将很难将外部的数据，转换为内部的方法。

Java和.net的反射机制很容易实现，由于其动态语言的特性，在编译时就存储了大量的**元数据**，而在动态装载时，也是根据这些元数据载入的模块。由于C++缺乏这些信息，往往并不能很好的动态装载和链接。操作系统为了实现C和C++的动态装载功能，特意设计了动态链接库，将符号表保存在动态库中，运行时重定位代码，然后进行链接操作。而这是操作系统实现的，并不能很好的被用在用户工程中，所以我们有必要自己构建一套**元数据**集合，保存反射所需的内容。

### 反射的原理

反射的核心就是根据字符串名字，创建对应的类或者调用对应类的方法，为此，我们使用C++中的map

```
	std::map<std::string, meta_class*>
```

meta_class 是保存一个类中的关键元数据用的类，可以支持反射构造，反射调用函数等功能。

meta_func 是保存一个方法的关键信息类，但由于方法有不定的参数和返回类型，我们使用模板的方式，将一个抽象存储的成员函数指针，转换为我们确定类型的成员函数指针，然后再去调用，达到动态调用的目的：
```
	template <typename T, typename R, typename... Args>
	R Call(T* that, Args... args) {
		R (T::*mp)();
		mp = *((R (T::**)())func_pointer);
		return (that->*mp)(args...);
	}
```

这里的代码十分混乱，如果你没学过C的函数指针的话，建议先去补习一下函数指针的定义和用法。

这里涉及到的是成员函数指针的传递，一会儿将会详细讲解如何传递任意一个函数指针。

### 反射类对象

首先，我们肯定要为类对象建立meta_class的模型，但每个meta_class，应该都能够构建本类的对象，为了实现这一特点，我们想到了模板：

```
template<typename T>
class MetaClass : public IMetaClass {
public:
	virtual void* CreateObject() {
		return new T();
	}
};
```

为了让每个类都能有统一的创建方法，我们将使用IMetaClass接口进行多态调用

```
class IMetaClass {
public:
	virtual void* CreateObject() = 0;

	template<typename T>
	T* Create() {
		return (T*) CreateObject();
	}

	void AddFunc(const std::string s, MetaFunc* c) { func_map[s] = c; }
	MetaFunc* GetFunc(const std::string s) {
		if (func_map.find(s) != func_map.end()) return func_map[s];
		else return NULL;
	}
private:
	std::map<const std::string, MetaFunc*> func_map;
};

```

这里我们在接口类中编写了方法和成员函数，我觉得这是C++的优势，不像Java，为了安全性，而取消了这么简单好用的功能。

接口统一实现相同的类对象构建方式，避免了在实现类中反复编写的困难。

这样，我们只要在每个类的定义时，向我们的类注册器注册该MetaClass对象就可以了

但问题是，如何才能在类定义时编写代码呢？我们的C和C++可是只能在函数中调用代码，不能像脚本一样随时随地执行。


### 利用全局对象的构造函数执行代码

我们发现，C++有一个很有趣的特性，有些代码是可以在main函数执行前就执行的：

```
class test
{
public:
	test() {
		printf("%s\n","Hello Ctor!");
	}
};

test show_message();

int main(){
	printf("Main function run!\n");
    return 0;
}
```

执行代码，哦？好像不大对，貌似我们的对象并没有启动，这有可能是被编译器优化掉了。。。= =！
控制台的显示：
```
sxf@sxf-PC:~/data/workspace/C++/OObyCpp/testCppRunCode$ ./main
Main function run!
```

稍加改动：
```
#include <cstdio>
using namespace std;

class test
{
public:
	test(const char* msg) {
		printf("%s\n",msg);
	}
};

test show_message("Hello Ctor!");

int main(){
	printf("Main function run!\n");
    return 0;
}
```

好的，我们发现构造函数运行在了main函数之前，也就是我们的类型定义的构造期。
```
sxf@sxf-PC:~/data/workspace/C++/OObyCpp/testCppRunCode$ ./main
Hello Ctor!
Main function run!
```

具体想详细了解C++的运行环境的细节，推荐看一本英文的开源书：
【How to Make a Computer Operating System】
这本书讲解如何利用C++开发了一个小型操作系统，而在C++运行时的导入过程中，就介绍了C++全局对象构造函数的运行过程，可以清楚的看出，C++的主函数在汇编层的调用流程是这样的：

```asm
start:
	push ebx
	 
static_ctors_loop: 					; 全局构造函数初始化
   mov ebx, start_ctors
   jmp .test
.body:
   call [ebx]
   add ebx,4
.test:
   cmp ebx, end_ctors
   jb .body
 
   call kmain  						; 调用主函数
 
static_dtors_loop:					; 全局对象的析构函数调用
   mov ebx, start_dtors
   jmp .test
.body:
   call [ebx]
   add ebx,4
.test:
   cmp ebx, end_dtors
   jb .body

```


于是我们可以这样编写一个类，专门用来注册一个类：

```
class reflector_class {
public:
	reflector_class(const char* name, IMetaClass* meta_class) {
		ClassRegister::Add(name, meta_class);
		printf("define class: %s\n", name);
	}
};
```

这个类的对象在构造时，会去调用ClassRegister类中的静态方法，向其中添加类名和类元数据

### 利用宏定义处理类的注册

我们希望每个类对象能够方便的找到自己的meta_class，最简单的方式就是将其添加为自己的成员，为何不用继承机制呢？首先继承较为复杂，并且父类也同样可能拥有meta_class, 我们希望每个类型都能方便的找到meta_class，那么可以建一条Reflectible宏，让大家写在class中

```
#ifndef Reflectible
#define Reflectible \
public:\
	static IMetaClass* meta_class;\
private:
#endif
```

为了避免放置在最上面时，影响下面成员的private的默认定义，所以写成这样。

我们在写一个宏，让用户添加到类的cpp文件中，真正定义该meta_class对象：
```
#ifndef ReflectClass
#define ReflectClass(class_name) \
	IMetaClass* class_name :: meta_class = new MetaClass< class_name >(); \
	reflector_class class_name##reflector_class( #class_name , class_name::meta_class)
#endif
```

这里我们用到了两个宏技巧：
	## 表示将两个符号连接在一起，由于词法分析中，宏是按照词的顺序分隔的，如果直接连接，往往会造成符号分析不清。
	#something 表示将该内容展开成字符串的形式 => "something data"，所以我们可以很方便的用这个宏将宏符号转为字符串传入到函数中。

### 反射成员函数

首先编写一个能调用成员函数的模板类，根据我们的反射原理，将一个函数指针转换为成员函数的指针：
```
class MetaFunc {
public:
	MetaFunc(void* p) { func_pointer = p; }
	void setFuncPointer(void* p) { func_pointer = p; }
	template <typename T, typename R, typename... Args>
	R Call(T* that, Args... args) {
		R (T::*mp)();
		mp = *((R (T::**)())func_pointer);
		return (that->*mp)(args...);
	}
private:
	void* func_pointer;

};
```
我在这里使用了C++11的新特性，可变参数的模板，这样可以更方便的接受目标参数
如果我们直接对成员函数取地址，返回的是一个return_type (ClassName::*)(args)这样的成员函数指针。
注意，成员函数指针不能直接被传递，成员函数指针由于包含了很多其他数据信息，并不能被被强制类型转换成void*,一个显而易见的例子是，成员函数指针，往往比较大，最大的指针甚至可以达到20byte。

为了能够传递函数指针，我们可以将成员函数指针赋值给一个该成员函数指针类型的对象，然后再对这个指针对象取地址

```
	auto p = &test::print;
	&p  //这个地址可以被轻松传递
```

这个地址是一个指针的指针`return_type (ClassName::**)(args)`
于是就有了我们前面代码中，强制类型转换的方法

### 利用C语言的可变参数函数来定义函数

我们目前要将地址传递过来，但是我们并不知道每个类中有多少个函数，所以我们要使用C语言的宏，对可变参数进行处理。

下面将reflector_class进行一下修改，支持多个参数
```
class reflector_class {
public:
	reflector_class(const char* name, IMetaClass* meta_class, ...) {
		ClassRegister::Add(name, meta_class);
		printf("define class: %s\n", name);

		va_list ap;
		va_start(ap, meta_class);  
		for (int arg = va_arg(ap, int); arg != -1; arg = va_arg(ap, int) ) 
		{
			std::string name(va_arg(ap, const char*));  
			void* p = va_arg(ap, void*);
			if (arg == 0) {
				printf("\tdefine func: %s\n", name.c_str());
				MetaFunc* f = new MetaFunc(p);
				meta_class->AddFunc(name, f);
			} else {
				printf("\tdefine prop: %s\n", name.c_str());
			}
		}  
		va_end(ap);  
	}
};
```

	va_list ap; 可变参数列表

	va_start(ap, meta_class);  这里的第二个参数，是当前函数的最后一个固定参数位置

	void* p = va_arg(ap, void*); 可以用来获得一个固定类型的参数

使用过后释放资源：
	
	va_end(ap);  



为了支持函数和属性两种声明，我们定义如下宏：
```
#ifndef DefReflectFunc
#define DefReflectFunc(class_name, func_name) \
	auto func_name##_function_pointer = &class_name::func_name
#endif

#ifndef ReflectClass
#define ReflectClass(class_name) \
	IMetaClass* class_name :: meta_class = new MetaClass< class_name >(); \
	reflector_class class_name##reflector_class( #class_name , class_name::meta_class ,
#endif

#ifndef ReflectFunc
#define ReflectFunc(func_name) \
	0, #func_name, _F(func_name##_function_pointer) ,
#endif

#ifndef ReflectProp
#define ReflectProp(prop_names) \
	1, #prop_names, _F(prop_names) ,
#endif

#ifndef _F
#define _F(x) reinterpret_cast<void*>(&x)
#endif

#ifndef End
#define End -1 )
#endif
```

这样我们在cpp中使用这些宏时只需要：
```
DefReflectFunc(test2,print2);
ReflectClass(test2)
	ReflectFunc(print2)
End;
```

### 类的全局注册难题

好的，关键的部分已经都清楚了，但目前我们还欠缺一个很重要的类，就是类的全局注册器。

```
class ClassRegister
{
public:
	static std::map<const std::string, IMetaClass*> class_map;
	static void Add(const std::string s, IMetaClass* k) {
		class_map[s] = k;
	}
}
```

但这个类有一个严重的漏洞，会造成程序崩溃，我们在接下来的章节中，将会介绍这个尴尬的问题的发生原因。