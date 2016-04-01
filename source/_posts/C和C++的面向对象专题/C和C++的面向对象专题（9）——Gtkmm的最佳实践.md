title: C和C++的面向对象专题（9）——Gtkmm的最佳实践
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


## 九、Gtkmm的最佳实践

在跨平台的gui开发中，Qt一直是非常受欢迎的GUI开发框架，但Qt一个是依赖反射，需要特殊的预处理步骤，一个是库太过庞大，这就造成了一些不便的地方。今天介绍给大家的是Gtk库的C++绑定，Gtkmm，一个方便的跨平台GUI开发框架。

由于是C++的封装，GTK不再那么的难以使用，变得简洁优雅，而且效率非常高，编译也较QT快许多。
虽然C也能编写，而且我们之前也介绍过了GObject的使用。但比较其实现起来较为繁琐，代码行数较C++多一些，而且每个成员函数都要手动传入this指针，较为不便。

现在C++如果合理的封装和按照之前的设计思想进行设计，结构十分紧凑，而且书写非常方便，非常易用。

### Gtkmm版的2048程序设计

为了更好的实践，我们举一个简单的2048小游戏的程序作为实例。大家会发现，合理的设计，能够使代码既清晰明了，又方便维护，可靠性很高。
我们简要的进行一下程序设计，这里我们不是要学会2048如何制作，而是要体会程序设计中的思想，以及设计中的美感和艺术感。

首先，2048作为一个简单的小游戏，广受大家喜欢，原理很简单，在一个4×4的数组中，让数字不断向各个方向合并，每次进行后，随机位置创建新数字。

程序界面，一个窗口，上面一行标签书写当前得分，下面一个绘图界面，自由绘图，画上4×4个的矩阵，上面书写内容。

程序结构设计，按照一般程序架构设计，可以用MVC的结构，一个界面类，负责显示，一个控制类，负责游戏逻辑，一个模型类，负责数据的存储与管理。

但由于数据的管理太过简单，就放弃了模型类，直接使用一个4×4的矩阵就完成任务。

### 程序实践

由于Gtkmm的良好封装，我们并不需要太多复杂的处理，首先是Main文件引导程序的启动，所有gtk程序集合都是这样引导。

```c
/* 
* @Author: sxf
* @Date:   2015-05-07 12:22:40
* @Last Modified by:   sxf
* @Last Modified time: 2015-05-20 21:58:16
*/

#include <gtkmm.h>
#include "game.h"
int main(int argc, char *argv[])
{
	Glib::RefPtr<Gtk::Application> app = Gtk::Application::create(argc, argv, "org.abs.gtk2048");

	Game game;
	//Shows the window and returns when it is closed.
	return app->run(game);
}
```


Game类作为最核心的窗口类，也是游戏的主要控制类，并不需要暴露什么方法给外部成员使用，所以它的定义很简单：
```cpp
/* 
* @Author: sxf
* @Date:   2015-05-19 11:20:42
* @Last Modified by:   sxf
* @Last Modified time: 2015-05-20 21:58:35
*/

#ifndef GAME_H
#define GAME_H

#include <gtkmm/window.h>

class Game_private;
class Game : public Gtk::Window
{
public:
	Game();
	virtual ~Game();

private:
	Game_private* priv;
};


#endif // GAME_H

```

这种写法，就是在前几章提到的增强代码封装性的方法，通过一个priv指针，解决了C++封装不完善的问题。

这样还有一个很大的好处就是，由于priv指针的书写较为繁琐，如果在public方法中，反复的通过priv指针调用函数，就会显得无比麻烦，但这正提醒你，你的写法有问题，因为一般的方法，要尽可能写成内部的private的，这样你在不自觉的时刻，就形成了最大化private方法，最小化接口的设计习惯，这对于提升程序的内聚性，很有意义。

于是我们的Game类的内部定义就变得十分复杂，但这就使得代码内聚性更高，暴露给外层的接口就更简单。

```
class Game_private
{
public:
	Game_private(Game* game);
	~Game_private();
	
 	Game* game;
	MyArea m_area;		// 渲染类
	Gtk::Box m_box;		// 布局控件
	Gtk::Label m_score;	// 得分标签
	int score;			// 得分具体数字
	const static int fx[4][2];

	int data[4][4];		// 数据模型

	bool combine(int i, int j, int k); 	// 将一个位置的数字向下合并

	bool randomNew();	// 随机创建新数字

	void cleanData();	// 删除全部数字，用来开局初始化

	void gameWin();		// 显示用户胜利

	void gameOver();	// 显示游戏结束

	void gameRun(int k);	// 游戏控制循环

	bool on_key_press_event(GdkEventKey* event); // 监听键盘事件响应
};

```

我不喜欢比脸还长的函数，但这里的函数设计的还是不是那么尽如人意，虽然如此，这里也是本着简单易懂的方式设计的。

这里的combine方法设计的很特殊，因为合并时，还有可能出现游戏胜利的情况，所以里面包含了判断胜利的条件。

```
bool combine(int i, int j, int k) {
	int ni = i; int nj = j;
	ni += fx[k][0];	// fx是方向数组
	nj += fx[k][1];
	int obji = i, objj = j;
	while ( 0 <= ni && ni < 4 &&
			0 <= nj && nj < 4 )		
	// 防止越界，我这里比较贪便宜，很多人处理越界是通过在数组外增加一个特殊值的外边框来处理的
	{
		if (data[ni][nj] == 0) {
			obji = ni; objj = nj;
		} else {
			if (data[ni][nj] == data[i][j]) {
				score += (1 << data[ni][nj]);	// 处理得分
				++data[ni][nj];
				data[i][j] = 0;
				if (data[ni][nj] == 12) return true; // 处理胜利条件
				return false;
			} else break;
		}
		ni += fx[k][0];
		nj += fx[k][1];
	}
	if (!(obji == i && objj == j)) {	// 未能合并的情况
		data[obji][objj] = data[i][j];
		data[i][j] = 0;
	}
	return false;
}
```

这个函数设计的很健壮，考虑了许多边界条件，这么做是在模拟物体碰撞时，碰撞面不断挤压的情况。例如下面的情况：
1 1 2 4 
0 0 0 0
0 0 0 0
0 0 0 0
向左合并，能够一次就被合成为8，但这也是和外层的合并顺序控制是分不开的
在游戏主循环控制时，是这样处理的，对于不同的方向，循环顺序是不一样的：
```
void gameRun(int k) {
		bool winflag = false;
		switch (k) {
			case 0 : 
				for (int i = 0; i < 4; ++i)
					for (int j = 0; j < 4; ++j)
						if (combine(i,j,k)) winflag = true;
			break;
			case 1 : 
				for (int j = 3; j >= 0; --j)
					for (int i = 0; i < 4; ++i)
						if (combine(i,j,k)) winflag = true;
			break;
			case 2 : 
				for (int i = 3; i >= 0; --i)
					for (int j = 0; j < 4; ++j)
						if (combine(i,j,k)) winflag = true;
			break;
			case 3 : 
				for (int j = 0; j < 4; ++j)
					for (int i = 0; i < 4; ++i)
						if (combine(i,j,k)) winflag = true;
			break;
		}

		// 判断胜负条件
		if (winflag) {
			gameWin();
			return;
		}
		if (!randomNew()) {
			gameOver();
		}

		Glib::RefPtr<Gdk::Window> win = game->get_window();
		if (win)
		{
			m_area.setData(data);
		    Gdk::Rectangle r(0, 0, 600, 600);
		    win->invalidate_rect(r, false);
			m_area.show();
			char score_text[20]; memset(score_text, 0, 20);
			sprintf(score_text, "Score : %d", score);
			m_score.set_text(score_text);
		}
	}

```

而创建新数字的方式也很清晰，但这里使用模拟栈的方式进行了处理。
设计很独特，由于目前的位置数目有限，直接rand的方式，效率较低，我们先扫描所有可能的位置，然后将其入栈，random时，直接找一个位置，然后直接随机从其中找一个就可以了。数组模拟栈的方式，主要是希望避免vector的低效率，实现较简易。而且扩展较方便，如果你想random跟多，修改起来也十分方便。

```
bool randomNew() {
	int empty_block[17][2]; int sum = 0;
	for (int i = 0; i < 4; ++i) {
		for (int j = 0; j < 4; ++j) {
			if (data[i][j] == 0) {
				empty_block[sum][0] = i; 
				empty_block[sum][1] = j;
				++sum;
			}
		}
	}
	if (sum < 1) return false;

	int t = rand() % sum;
	data[ empty_block[t][0] ][ empty_block[t][1] ] = (rand() % 4) < 3 ? 1 : 2;
	empty_block[t][0] = empty_block[sum-1][0];
	empty_block[t][1] = empty_block[sum-1][1]; 
	--sum;
	
	return true;
}
```

渲染类十分简单，主要就是根据数组中的数值，渲染出对应的图像，设计思想就是不断将问题抽象，不断简化，将复杂的问题从上层一层层拨开，这样就使得结构更加简洁优雅。

整个项目完整代码已经放到Github上了，需要的可以参考：
[【github仓库】](https://github.com/sunxfancy/Gtk2048)