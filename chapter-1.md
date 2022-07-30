# 要点1: 理解模板类型推断

对模板中类型的推断有一些小trick,需要特别注意,因为对这种类型的推断可能不是那么智能.

我们从一个例子看起:

```cpp
template<typename T>
void f(ParamType param);
```

然后我们这样调用该模板

```cpp
f(expr)
```

那么在**编译**的过程中, (题外话:分清楚cpp编译期和运行期的区别. 虚函数实现的多态是在运行期,而模板实现的多态是在编译期.) 我们回到正题, 编译器会用 *expr* 去推断 *T* 和 *ParamType*, 而这两者有时候会不一样, 因为*ParamType*会带有 const或者&. 例如:

```cpp
int x = 0;
f(x);
```

在这个例子中,*ParamType*的类型是: **const int&**, 而 T是**int**. 所以大原则是: **T的类型要综合考虑expr的类型和ParamType的类型**

原书中给出了三种情况,我们先看第一种:

**情况1: ParamType的类型是引用或者指针,但不是通用引用**

先看一个例子
```cpp
template<typename T>
void f(T& param);  // ParamType 带有引用
                   // 
int x = 27;        // x 类型是 int
const int cx = x;  // cx 类型是 const int
const int& rx = x; // rx 类型是const int&

//然后我们这样调用
f(x);    
f(cx);    
f(rx);    
```

推断的原则是:
1. 如果expr有引用,先把expr里的引用去掉
2. 再用expr和T的做模式匹配 (找公共部分)

结果如下
```cpp
f(x); // expr的类型就是 int, ParamType类型是 T& (int&), 
      // int就匹配到了T, 所以T就是int

f(cx) //expr类型的const int, ParamType类型是 T&(const int&),
      //const int就匹配到了 T

f(rx) //expr类型是const int&, &被去掉,剩下const int,
      //继续匹配 T&,所以T就是const int
```

接下来有一个小坑, 看例子

```cpp
template<typename T>
void f(const T& param); 
int x = 27; 
const int cx = x; 
const int& rx = x; 
f(x); 
f(cx); 
f(rx);
```

我们注意到ParamType里包含了const, 那么这个时候, T就不用再包含const了

```cpp
f(x)   // expr类型是int, ParamType是const T&, 所以int就匹配到了T, 
       // 而不是 const T
f(cx)  // expr类型是const int, ParamType是 const T&,所以T匹配到 int
f(rx)  // expr类型是const int&, &去掉,剩下const int, 匹配const T&, 
       // 所以T还是int
```

如果param的类型是*,指针, 那么处理法则和引用&一样.

第一种情况到此结束

**情况2: ParamType 是通用引用 T&&**

```cpp
template<typename T>
void f(T&& param); 
int x = 27; 
const int cx = x; 
const int& rx = x; 
f(x); 
f(cx); 
f(rx); 
f(27); 
```

在这种情况里, 有两个要点: 1.如果expr是lvalue, T和ParamType一定是lvalue引用. 2.如果expr是rvalue,我们用情况1的法则处理.也就是说,ParamType是通用引用T&&, T的类型就是用expr的类型和T&&做匹配.


好,我们现在回到正题
```cpp
int x = 27; 
const int cx = x; 
const int& rx = x; 
f(x); // x是int,是lvalue, 所以ParamType是T&(int&), 然后int与其做
      //匹配, 匹配到了 T, 所以T是int, 由于我们刚说过的要点1, 
      //所以再加一个引用&, T的最终答案是int&.

f(cx); // cx是const int, lvalue,所以param是T&(const int&),
       // 然后const int和T&匹配, T匹配到了const int, 再加一个&
       // 于是T的最终答案就是const int&

f(rx); //rx是const int&, lvalue, 所以param是T&, T&和const int&匹配
       //T匹配到了const int, 再加&,所以T最终是const int&

f(27); //27是rvalue, int类型, 根据要点2, param是T&&, 
       //expr的类型int和T&&匹配,T就匹配到了int, 所以T是int
```




