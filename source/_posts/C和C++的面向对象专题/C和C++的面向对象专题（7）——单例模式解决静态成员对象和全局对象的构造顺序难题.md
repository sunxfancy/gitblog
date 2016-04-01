title: C和C++的面向对象专题（7）——单例模式解决静态成员对象和全局对象的构造顺序难题
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


## 七、单例模式解决静态成员对象和全局对象的构造顺序难题

上回书说道，我们的程序有一个隐藏的漏洞，如果ClassRegister这个类所在的.o文件，如果在所有.o文件中是第一个被链接的的，那么就不会出问题。
这么说太抽象了，让我们画个图表

```
ClassRegister.o
--------------------
Meta.o
--------------------
Main.o
```

这样的结构，也就是链接顺序要这样指定
```
	gcc -o main ClassRegister.o Meta.o Main.o
```
就不会出问题，但如果调换一下顺序就可能出问题。

思考一下原因，ClassRegister中的map对象，是一个全局对象，而我们注册类的时候，也使用了全局对象的构造函数，两个谁先执行呢？这个就不得而知了，C++并未说明两个谁先谁后，而一般链接器，都是从前往后链接代码，而构造函数的执行顺序，也往往和链接时的顺序有关。

但这样实现就很不好，我们的系统居然要靠链接器的顺序才能正确编译执行，太不可靠了，万一用户没注意到这一点，直接编译链接，就会出现未知的错误。

那么如何避免这种情况呢？

### C++的单例模式

C++中有一种极好的设计模式很适合这种情况，那就是用单例，单例模式也很容易理解，核心就是推迟构造，如果没有使用时，就不会被构造，被用到时，对象就会构造，并且仅一次，最常见的写法就是：

```
    class CSingleton  
    {  
    private:  
        CSingleton()   //构造函数是私有的  
        {  
        }  
        static CSingleton *m_pInstance;  
    public:  
        static CSingleton * GetInstance()  
        {  
            if(m_pInstance == NULL)  //判断是否第一次调用  
                m_pInstance = new CSingleton();  
            return m_pInstance;  
        }  
    };  
```

当然，我们这里并没有考虑多线程，因为多线程时单例模式一般要加锁来保障不会多次构造引发冲突。

于是经过简要修改，就能用单例模式设计一个类注册器了：
```
class ClassRegister
{
private:
	ClassRegister() { printf("register\n"); }  //构造函数是私有的   
	std::map<const std::string, IMetaClass*> class_map;
public:
	static ClassRegister * GetInstance() {  
		static ClassRegister instance;   //局部静态变量  
		return &instance;
	}
	static void Add(const std::string s, IMetaClass* k) {
		GetInstance()->class_map[s] = k;
	}

	static IMetaClass* Get(const std::string s) {
		std::map<const std::string, IMetaClass*>& m =  GetInstance()->class_map;
		if (m.find(s) != m.end()) return m[s];
		else return NULL;
	}
};
```
这个类注册器简单实用，采用的设计方式和指针的模式稍有不同，这里用到了局部静态变量的概念。
局部静态变量能够改变对象的生存周期，这样就能很好的符合我们的要求。