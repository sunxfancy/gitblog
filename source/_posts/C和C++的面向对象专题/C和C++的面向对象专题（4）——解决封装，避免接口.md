title: C和C++的面向对象专题（4）——解决封装，避免接口
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


## 四、解决封装，避免接口

恩，今天我们来讨论，如何通过设计，解决C++中的不优雅特性，改进项目的结构，改善编译速度。

上次我们提到，如果一个类的封装不好，容易导致种种不便，那么如何设计能够避免这种现象呢？

```
class test {
public:
	void print() {
		printf("Hello\n");
	}

	void print2() {
		printf("K : %d\n", k);
	}

private:
	int k;
};

```

### 简要的改进，将函数的实现移动到类cpp实现文件中

最简单的想法就是将实现和声明分开，这也是C++提倡的，这样虽然文件会增多，但编译速度和代码的清晰度会提升。

```
/* test.h */
class test {
public:
	void print();
	void print2();
private:
	int k;
};
```

```
/* test.cpp */
#include "test.h"
void test::print() {
	printf("Hello\n");
}

void test::print2() {
	printf("K : %d\n", k);
}
```

很明显的，这样我们改动cpp文件中，.h文件不会受到影响，但假若我的private方法增加了，那么还是需要改动.h文件，进而会影响所有引用我的部分，为了避免这种情况出现，有什么好设计方法么？

### 使用接口降低代码耦合

一种标准的设计模式是使用接口，这在很多库的设计时也被经常采用，核心思想是通过多态调用的方式，避免内部方法的暴露。

接口一般就是C++的多态类：

```
/* Itest.h */
class Itest {
public:
	virtual void print() = 0;
	virtual void print2() = 0;
};

extern ITest* createItest(); // 类似工厂的方式为你构建类
```

```
/* Itest.cpp */
#include "test.h"

ITest* createItest() {
	return new test();
}
```

让test从这个接口继承出来：
```
/* test.h */
class test : public Itest {
public:
	virtual void print();
	virtual void print2();
private:
	int k;
};
```

这样的好处当然十分明显了，将类转成接口的形式，就能方便的修改下面的实现类，无论实现类如何改动，都在模块范围内，接口不变。
但这样做的坏处也很明显，如果C++大量使用这样的方式实现内部封装，那么很多情况下效率比较低，而且代码复杂度就上来了，需要添加很多的接口类。

### 轻便的成员类封装

下面介绍一种简单的方式来实现类封装性的提升，首先还是看这个test类，我们将其提示为test2：
```
/* test2.h */
class test2 {
public:
	void print();
	void print2();
private:
	int k;
};
```
这里的k实际上并不需要写在这里，我们需要的是将private的部分整体的封装成一个类：

```
/* test2.h */
class test2_private;

class test2 {
public:
	test2();
	test2(int);
	~test2();

	void print();
	void print2();

protected:
	test2_private* that;
};
```


```
/* test2.cpp */
#include "test2.h"

class test2_private {
public:
	int k;
};

test2::test2() {
	that = new test2_private();
	that->k = 0;
}


test2::test2(int k) {
	that = new test2_private();
	that->k = k;
}

test2::~test2() {
	delete that;
}

```

这时我们发现，这种封装可以很有效的解决类的接口不便的问题，而由于只使用了类指针，所以我们并不需要前向声明这个私有类，于是这个类可以方便的被修改，从而避免了接口和多态调用的问题。

这种设计还有一个用途，假若你有另外的代码生成器生成的代码，需要和已有的类嵌入使用，那么推荐使用这种方式，Qt中就是这样做的：

```
#ifndef MAINWINDOW_H
#define MAINWINDOW_H

#include <QMainWindow>

namespace Ui {
class MainWindow;
}

class MainWindow : public QMainWindow
{
    Q_OBJECT

public:
    explicit MainWindow(QWidget *parent = 0);
    ~MainWindow();

private:
    Ui::MainWindow *ui;
};

#endif // MAINWINDOW_H

```


```
#include "mainwindow.h"
#include "ui_mainwindow.h"

MainWindow::MainWindow(QWidget *parent) :
    QMainWindow(parent),
    ui(new Ui::MainWindow)
{
    ui->setupUi(this);
}

MainWindow::~MainWindow()
{
    delete ui;
}
```

我们发现这里有一个神奇的代码
```
namespace Ui {
class MainWindow;
}
```

其实这只是另外一个类，和本类并不同名，Ui::MainWindow是qt设计器帮忙生成的类，用来标注UI界面生成的一些代码，为了让代码很好的和我们自己的类统一起来，他们用了这种方式。

