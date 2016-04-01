title: C和C++的面向对象专题（3）——C++中的不优雅特性
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


## 三、C++中的不优雅特性

今天来说一说C++中不优雅的一些问题，C++虽然是面向对象的设计语言，但也有很多缺陷和弊病，我们将会讨论如何通过良好的设计解决这些问题。

### C++编译缓慢

C++编译慢已经成为了业界共识，一个大型C++项目甚至要用专用的服务器编译好久才能完成，Java和.net作为大型开发平台， 却也没发现编译如此缓慢的问题，那么究竟是什么，导致了C++编译难的问题呢？

### 模板的纠结

C++中模板有个很神奇的问题，就是实现和声明都必须被使用者引用，这段模板代码才有效，也就是说，模板是在编译时展开的代码生成机制。

我们不妨做个实验，这是类的声明：
```
template<class T>
class CObject
{
public:
	CObject(T k) {obj = k;}
	~CObject() {}
	T getObj();
private:
	T obj;
};
```

下面是类的实现：
```
#include "CObject.h"

template<class T>
T CObject<T>::getObj(){
	return this->obj;
}
```

主函数中调用：
```
#include <cstdio>
#include "CObject.h"

using namespace std;

int main(){
	CObject<int> Obj(10);
	int k = Obj.getObj();
	printf("%d\n", k);
    return 0;
}
```
一切看起来是那么的顺利，但是！我的电脑给我显示如下错误信息：
```
Scanning dependencies of target template_test
[ 50%] Building CXX object CMakeFiles/template_test.dir/src/CObject.cpp.o
[100%] Building CXX object CMakeFiles/template_test.dir/src/main.cpp.o
Linking CXX executable template_test
CMakeFiles/template_test.dir/src/main.cpp.o：在函数‘main’中：
main.cpp:(.text+0x22)：对‘CObject<int>::getObj()’未定义的引用
collect2: error: ld returned 1 exit status
make[2]: *** [template_test] 错误 1
make[1]: *** [CMakeFiles/template_test.dir/all] 错误 2
make: *** [all] 错误 2
```
链接器告诉我，我们找不到一个叫做‘CObject<int>::getObj()’的函数，恩？为何，我们不是将类实现链接进来了么？

如果你这样想就错了，上网查找解决方案，得到的回复居然是这样：
`#include "CObject.h"` => `#include "CObject.cpp"`

omg，那我还不如把两个文件写成一个hpp来的方便呢，其实C++也是推荐你这样做的，理由就是——模板是编译时，在用到的时候进行代码展开得到的
如果不这样做，链接器是不会找到对应的代码的。

那么也找到了很多大型工程如boost库，为何编译缓慢的直接原因，大量的模板展开消耗了巨大的资源，而且模板展开是很不利于代码复用的，同样的算法，换一种类型，必须全部编译，生成新的代码，并且这类模板生成的代码，不能提前编译成二进制库，这样的结果就是，项目哪里改动一点，好多文件重复编译，造成编译十分缓慢。

### 封装的问题

C++的类并没有很好的将代码封起来，这和上次讲到的GObject对比可以发现，C++的私有变量是一同放置在类的声明中，而我们知道，一个类的声明，是会被很多其他类引用的。
那么，思考我们的C++编译过程，很多类都引用了一个.h文件，那么这个.h文件一旦发生更改，那么所有引用这个文件的cpp文件都将被触发重复编译，而我们在实现一个类时，对类的成员函数小修小补是很平常的，但由于封装的不彻底，那么我们的项目又将被反复编译，带来编译的速度缓慢。

而且，如果是库的话，那么私有成员的更新甚至还会影响用户使用，非常麻烦。
例如下面这段代码：
```
class Test {
public:
	Test();
	~Test();

	void Show();

private:
	std::string message;
	int pointer;
	void formatMessage(std::string&);
};
```

很明显，一般的C++类，私有成员都会比公开成员多，那么私有成员修改一点，哪怕只是一不小心多了个空格，都会带来这个文件的更新，触发makefile的重编译，带来了低效率。

### 缺乏反射机制

最新的C++11，引入了众多的新特性，包括好用的auto关键字以及模板元编程特性等，但这些，还是不能弥补反射机制缺失带来的影响。反射是对象串行化、GUI界面事件响应和根据数据动态调用代码等技术的核心，缺乏反射机制，会使得C++很多地方十分的不便。

很多大型软件，如firefox，在实现中，往往搭建了反射框架，供系统使用。但由于C++本身语法的问题，缺乏反射依旧会使得类书写变得困难。


### 跨平台困难

C++的跨平台性真的不好，甚至很多编译器上都会出现匪夷所思的问题，例如在不同平台上，基本类型的大小会随CPU字长而变化，如果有跨平台需求的软件，最好使用跨平台定义的类型。
C++的结构体中数据往往有内存对齐的问题，有些编译器还能通过编译器指令对其设置，这些问题最好还是能避开就避开。

跨平台时，还应小心异常处理的代码，因为有些版本的C++编译器对抛出的异常规格并不很遵守规范。
另外，不同平台的宽字符集也是大问题，往往并不能轻松统一，另外MinGW里貌似就没有宽字符- -

