title: C和C++的面向对象专题（8）——更为高级的预处理器PHP
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


## 八、更为高级的预处理器PHP

C++的宏在某些情况下非常难用，例如将代码展开成为这样：

Macro( A, B, C, D )

=>

func("A", A);
func("B", B);
func("C", C);
func("D", D);

test(A);
test(B);
test(C);
test(D);

这对于宏来说，太困难了，为了能实现复杂的宏展开，我们希望用更高级的预处理器来实现该功能。

我们这里使用PHP进行代码的预处理工作，将PHP代码当做C++的宏使用。
当然，你也可以用python做代码生成工作，但由于php是内嵌式的，处理起来可能更方便一些，当然，其他语言配上模板也是可以的。

```c
/* main.php */
<?php $return_m = "return a + b;" ?>

#include <iostream>

using namespace std;

int func(int a, int b) {
	<?php echo $return_m; ?> 
}
int main() {
	cout << func(1, 2) << endl;
	return 0;
}
```

我们用如下指令生成C++代码：

	php main.php > main.cpp

好的，下面就和正常的项目编译一样了，你甚至可以将php的命令写入到makefile中，自动化生成