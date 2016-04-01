title: C和C++的面向对象专题（5）——合理使用模板，避免代码冗余
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


## 五、合理使用模板，避免代码冗余

下面我们来讨论一下，如何解决模板的不易封装的问题。

我们提供这样一种思路，对于链表一类的通用类型，我们尽量采取强制类型转换的方式，尽量避免模板的滥用。

同样，我们应该避免对结构体的直接存储，尽量使用类似java的指针传递方式来传递对象。

我们首先来写一个单类型的list

```
#ifndef LIST_C_H
#define LIST_C_H

class list_c_private;
struct list_c_node;

class list_c {
public:
	list_c();
	~list_c();

	void insert(void*);
	int size();
	void* get(int);

protected:
	list_c_private* priv;
};

#endif // LIST_C_H

```

这里我们使用了上面讲到的封装方式，降低了类间的耦合度

```
#include "list_c.h"

class list_c_private
{
public:
	int size;
	list_c_node* head;
};	

struct list_c_node
{
	void* data;
	list_c_node* next;

	list_c_node() {
		data = next = nullptr;
	}
};


list_c::list_c() {
	priv = new list_c_private();
	priv->head = new list_c_node();
}

list_c::~list_c() {
	delete priv;
}

void list_c::insert(void* data) {
	list_c_node* p;
	for (p = priv->head; p->next != nullptr; p = p->next) {}
	p->next = new list_c_node();
	p->next->data = data;
}

int list_c::size() {
	return priv->size;
}

void* list_c::get(int k) {
	int t; list_c_node* p;
	for (p = priv->head->next, t = 0; p != nullptr && t != k ; p = p->next, ++t) {}
	return p->data;
}
```

这是一个简单的链表，只是作为示例使用，写了插入和获取的两个方法。

而为了通用性支持，我们写一个模板，进行类型的强制转换：

```
#ifndef LIST
#define LIST

#include "list_c.h"

template<typename T>
class list {
public:
	list() { clist = new list_c(); }
	~list() { delete clist; }

	void insert(T data) {
		clist->insert((void*)data);
	}

	int size() { return clist->size(); }

	T get(int k) {
		return (T)clist->get(k);
	}

protected:
	list_c* clist;
};

#endif // LIST
```

这样，带来的好处有，首先能够将模板封装操作，其次，能够在封装类中，动态的调整内部实例。
对于一个传入的类型，你可以判断一下，是否适合当前的模板，如果不适合，可以在其中动态的报错。

最后是模板的使用：
```
#include <iostream>
#include "list"
using namespace std;

int main(){
	list<long> testlist;
	testlist.insert(10);
	testlist.insert(20);

	long k = testlist.get(1);
	printf("%d\n", k);
    return 0;
}
```