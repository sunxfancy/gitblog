title: C和C++的面向对象专题（2）——C语言也能实现面向对象
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



## 二、C语言也能实现面向对象

今天要为大家介绍C语言的面向对象设计方法，正如题记上面所说，面向对象是一种思想，而并非是一种语言，我们将会介绍C语言实现的面向对象开发方式。

### 简单实用的C语言面向对象设计思路

众所周知，C++中的面向对象十分方便，但在C中，并没有类，但我们可以通过C的结构体来实现，然后，手动将this指针传入
目前这个方法，应该是C语言设计中，简便易用的方式，而且能较好的体现面向对象的设计思路，然而遗憾的是，没有继承和多态。

例如，我们这样一个C++类

```cpp
class test {
public:
	test(char* str, int k) {
		this->str = str;
		this->k = k;
	}
	void ShowMessage() {
		for (int i = 0; i < k; ++i)
			printf("%s\n", str);	
	}
private:
	int k;
	char* str;
};
```

那么我们可以这样转换为一个C类

```c
/* test.h */

typedef struct _test {
	int k;
	char* str;
} test;

test* TestCreate();
void ShowMessage(test*);
```

```c
/* test.c */

test* TestCreate() {
	return (test*) malloc(sizeof(test));
}

void ShowMessage(test* t) {
	int i;
	for (i = 0; i < t->k; ++i) {
		printf("%s\n", t->str);
	}
}
```

其实思路也很清晰，思路简单易懂，实现也很清新明快，在各类C工程中使用极为广泛。

### 复杂的基于GObject的面向对象程序设计

如果你希望学习C语言的GUI程序设计，那么，必定要学习的就是GObject的类实现方式。
GObject相当于从C层面上模拟了一个C++的类对象模型，实现当然相对复杂的多。

下面我们来实际看一下一个GTK的窗口类，这是GTK+-3.0的一段样例：

```
/* appwin.h */
#ifndef APPWIN_H
#define APPWIN_H

#include "app.h"
#include <gtk/gtk.h>

/* 该类的类型定义以及类型转换宏 */
#define APP_WINDOW_TYPE (app_window_get_type())
#define APP_WINDOW(obj) (G_TYPE_CHECK_INSTANCE_CAST ((obj), APP_WINDOW_TYPE, AppWindow))

/* 该类分成两部分，一部分是成员，一部分是类本身 */
typedef struct _AppWindow      AppWindow;
typedef struct _AppWindowClass AppWindowClass;

GType      app_window_get_type (void);
AppWindow* app_window_new      (App *app);
void       app_window_open     (AppWindow *win, GFile *file);

#endif // APPWIN_H

```

而其真实的定义是在.c文件中
```c

struct _AppWindow
{
	GtkApplicationWindow parent;
};

struct _AppWindowClass
{
	GtkApplicationWindowClass parent_class;
};


typedef struct _AppWindowPrivate AppWindowPrivate;

struct _AppWindowPrivate
{
	GSettings *settings;
	GtkWidget *stack;
	GtkWidget *search;
	GtkWidget *searchbar;
	GtkWidget *searchentry;
	GtkWidget *gears;
	GtkWidget *sidebar;
	GtkWidget *words;
	GtkWidget *lines;
	GtkWidget *lines_label;
};

G_DEFINE_TYPE_WITH_PRIVATE(AppWindow, app_window, GTK_TYPE_APPLICATION_WINDOW);

/* 后面有具体的实现方法，这里就不一一列举 */

```

我们发现，这种定义方式比C++中的其实更有优势，封装的更加彻底。为何这样说呢？首先，我们的声明文件十分的简洁，如果公开方法不修改的话，那么将其余内容如何改动，都不会影响我们的外部接口。

其次，由于需要显示的向GObject注册，那么动态进行类注册就成为可能，这样的设计优势表现在哪里呢？多语言的互通性就很好了，因为很多动态语言，是支持类的动态加载以及反射加载的。

另外，vala语言就是基于GObject类型的，他是一门新兴的编译时语言，但其也有很多动态语言的特性，用其开发gtk程序，比C具有明显的优势。