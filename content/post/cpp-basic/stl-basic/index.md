---
title: 浅谈STL
description: 浅谈cpp中好用的一些stl
date: 2024-10-23 02:00:00+0000
categories:
    - cpp
tags:
    - CPP
---

## 前言

除了一些新颖的语法，cpp中仍然有一部分多于c的地方，便是stl，基于template技术为我们实现了一些可直接使用的数据结构。例如栈，队列，map等，本文就为大家介绍部分stl。

## 栈(stack)

[栈的概念](https://baike.baidu.com/item/%E6%A0%88/12808149)  摘自百度百科

在百科中，给出了栈的基本概念及实现方法。但在cpp中，有实现好的栈，位于头文件stack中。

```cpp
#include <stack>
```

通过stack<T>的形式即可声明一个栈。T为数据类型。

例如：

```cpp
stack<int> s;
```

通过push方法和pop方法分别可以实现进栈和出栈的操作。

通过top方法则可以获取栈顶元素。

请看示例代码。

```cpp
#include <iostream>
#include <stack>
using namespace std;
int main()
{
    stack<int> test;
    int t = 5;
    test.push(5);
    test.push(6);
    test.push(t);
    t = 10;
    test.push(9);
    test.pop();
    cout << test.top();
    return 0;
}
```

## vector

Vector在我眼里可以视作一个动态数组，对于无法确定数组具体上限的情况下，vector无疑是个较好的选择。vector位于头文件vector中，下面为大家介绍它的相关使用方法。

```cpp
#include <vector>
```

### 定义

定义方法与栈类似。vector<T>声明一个vector，例如

```cpp
vector<int> v;
```

### push_back方法

通过push_back方法可以往vector中塞入元素，既然我们把它当做动态数组来理解，那么就是往动态数组中加入元素。例如：

```cpp
v.push_back(1);
v.push_back(2);
```

就是往v中加入了两个元素，分别是1，与2。

### pop_back方法

通过pop_back方法可以删除最后加入的元素，也就是说

```cpp
v.pop_back();
```

此时v中的2被弹出，只剩下1一个元素。

### 遍历vector

此时需要引入iterator的概念，可以理解为指向vector中的元素的指针。

通过begin方法和end方法，可以获取指向第一个元素和最后一个元素，通过指针自增的方法可以实现从第一个元素一直遍历到最后一个元素的效果。

例如，如果需要遍历v，可以操作如下。

```cpp
for(auto i = v.begin();i!=v.end();i++)
{
    cout << (*i) << " "; // i可以理解为此时遍历到的元素的指针，*i即可取出元素。
}
```

[auto](https://baike.baidu.com/item/auto/10128)关键字可以自动根据赋值的类型推导出新定义变量的类型。属于c++11新增的特性。

此时对于i来说，推导出的类型为vector<int>::iterator。

对于倒序遍历，vector提供了两个很有意思的函数rbegin和rend。通过把上面示例代码中的**begin**替换成**rbegin**，把**end**替换成**rend**，即可实现倒序遍历的效果。

### resize方法

resize方法可以调整vector的容量，从而实现拓展或者缩减vector的效果。

如果resize后新的容量小于原vector中的元素的容量，则会删除多出来的相对加入位置靠后的元素。

例如

```cpp
v.resize(0);
```

将容器大小调整为零，还有一个可以理解的意义，即清空vector。

### 示例

```cpp
#include <iostream>
#include <vector>
using namespace std;
int main()
{
    vector<int> test;
    test.push_back(5);
    test.pop_back();
    test.push_back(4);
    test.push_back(5);
    test.push_back(2);
    test.push_back(3);
    test.resize(3);
    for(auto i = test.begin();i!=test.end();i++)
    {
        cout << (*i) << " ";
    }
    return 0;
}
```

## 集合(set)

set位于头文件set中

```cpp
#include <set>
```

set虽然感觉在我日常开发或者写题中使用不多，~~可能是我之前并不怎么了解这个的原因~~，但它确实是一个很有趣的东西，在我看来，它在形式上类似于vector，但是要多出两个特性。

- 不允许重复元素的存在，即你可以插入很多次同一个元素(合法)，但set中只会保留一个。

- 自动排序，按照自小到大的顺序。因此，使用自定义类型时，应该注意实现<符号。

### 定义

set<T>，T为数据类型

```cpp
set<int> s;
```

### 插入元素

insert方法，例如

```cpp
s.insert(1);
s.insert(2);
s.insert(3);
```

### 删除元素

erase方法可以删除元素，例如

```cpp
s.erase(1);
```

### 遍历

与vector相同，不在赘述，但值得注意的是，vector是按照插入顺序排列元素，而set则是按照升序排列元素。

### 示例代码

```cpp
#include <iostream>
#include <set>
using namespace std;
int main()
{
    set<int> test;
    test.insert(5);
    test.insert(4);
    test.insert(5);
    test.insert(6);
    test.insert(5);
    test.insert(1);
    for(auto i = test.begin();i!=test.end();i++)
    {
        cout << *i << " ";
    }
    cout << endl; 
    for(auto i = test.rbegin();i!=test.rend();i++)
    {
        cout << *i << " ";
    }
    return 0;
}
```

## 队列

队列的概念：[队列（常用数据结构之一）_百度百科](https://baike.baidu.com/item/%E9%98%9F%E5%88%97/14580481)

cpp中实现了队列，位于头文件queue中。

```cpp
#include <queue>
```

定义方法queue<T>，T为类型名。

### 插入与弹出

即为push方法与pop方法，与前面介绍的几种结构类似，不在赘述，请大家自行理解，也可以参照示例代码。

### 获取队顶元素和队尾元素

分别是front方法和back方法，详见示例。

### 优先队列

优先队列与队列类似，只是多出了个升序排列的特性，类型名为**priority_queue**。

使用方法与queue完全相同，只是多出了一个类似于自动排序的功能。

### 示例代码

```cpp
#include <iostream>
#include <queue>
using namespace std;
int main()
{
    queue<int> test;
    test.push(123);
    test.push(123121);
    test.push(222);
    test.pop();
    cout << test.back() << test.front();
    cout << endl;
    priority_queue<int> test2;
    test2.push(5);
    test2.push(9);
    test2.push(2);
    test2.push(4);
    test2.push(1);
    test2.push(3);
    test2.pop();
    cout << test2.top();
    return 0;
}
```

## map

map本质上就是一个键值对，位于头文件map中 

```cpp
#include <map>
```

它的定义形式为map<T1,T2>，T1为键的类型，T2为值的类型。

一个确定的键对应一个确定的值。

例如：

```cpp
map<string,int> test;
test["a"] = 1;
test["b"] = 2;
test["a"] = 3;
```

该段代码中先是定义了一个map变量。

然后去添加键值对。由于键与值是一一对应的，现在map中实际上只存在两个键值对，即a->3，b->2。

该例子很好的体现了键值对的操作方法。即变量名[键] = 值的方法进行操作。

用于判断map中是否存在特定的键，提供了count方法，可以或许对应键的数量。（其实也只有1和0，因为一个键只对应一个值。~~所以为什么不设计成exist的形式呢~~）

下面是示例代码

```cpp
#include <iostream>
#include <map>
using namespace std;
int main()
{
    map<string,int> test;
    test["a"] = 1;
    test["b"] = 2;
    test["a"] = 3;
    cout << test.count("a") << endl;
    cout << test["a"] << endl;
    return 0;
}
```

## 相关练习

### uva442

题目链接：[Matrix Chain Multiplication - UVA 442 - Virtual Judge](https://vjudge.net/problem/UVA-442)

涉及到栈和map的使用，给出示例代码，请大家自行理解。

```cpp
#include <iostream>
#include <map>
#include <stack>
using namespace std;
struct Matrix
{
    Matrix(int a=0,int b=0):a(a),b(b){}
    int a;
    int b;
};
int main()
{
    map<char,Matrix> MatrixMap;
    int n;
    cin >> n;
    for(int i=1;i<=n;i++)
    {
        char cnt1;
        int cnt2,cnt3;
        cin >> cnt1 >> cnt2 >> cnt3;
        MatrixMap[cnt1] = Matrix(cnt2,cnt3);
    }
    string inputCom;
    while(cin >> inputCom)
    {
        stack<Matrix> cheng;
        int ans = 0;
        bool success = true;
        for(int i=0;i<inputCom.length();i++)
        {
            if(inputCom[i]!='(' && inputCom[i]!=')')
            {
                cheng.push(MatrixMap[inputCom[i]]);
            }
            if(inputCom[i] == ')')
            {
                Matrix m1 = cheng.top();
                cheng.pop();
                Matrix m2 = cheng.top();
                cheng.pop();
                if(m1.a != m2.b)
                {
                    success = false;
                    break;
                }
                ans += m2.a * m2.b * m1.b;
                cheng.push(Matrix(m2.a,m1.b));
            }
        }
        if(!success)
        {
            cout << "error" << endl;
        }
        else
        {
            cout << ans << endl;
        }
    }
    return 0;
}
```

### uva514

题目链接：[Rails - UVA 514 - Virtual Judge](https://vjudge.net/problem/UVA-514)

涉及到队列和栈的使用

```cpp
#include<iostream>
#include<stack>
#include<queue>
using namespace std;
int main()
{
    int n;
    while(cin >> n && n!=0)
    {
        int cnt;
        while(cin >> cnt && cnt != 0)
        {
            queue<int> rail;
            stack<int> realRail;
            rail.push(cnt);
            for(int i=2;i<=n;i++)
            {
                int _cnt;
                cin >> _cnt;
                rail.push(_cnt);
            }
            for(int i=1;i<=n;i++)
            {
                realRail.push(i);
                while(!realRail.empty() && !rail.empty() && realRail.top() == rail.front())
                {
                    rail.pop();
                    realRail.pop();
                }
            }
            if(!rail.empty())
            {
                cout << "No" << endl;
            }
            else
            {
                cout << "Yes" << endl;
            }
        }
        cout << endl;
    }
    return 0;
}
```

### 总结

上述题目离开stl当然能做，但使用给出的stl显得可读性更高并且更加方便，使开发效率大大增强。在认识到stl便利性的同时应该了解stl的效率相对不高，在部分oi题中可能会导致超时等问题，需要斟酌使用。
