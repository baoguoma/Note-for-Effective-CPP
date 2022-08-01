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
**情况3 ParamType既不是普通引用, 也不是通用引用, 也不是指针**

看个例子
```cpp
template<typename T>
void f(T param); 
```

因为T不是指针和引用, 所以param是传值. 其实也就是一个copy, 是一个全新的变量.
这个时候对T的推断,一句话概括: 把const, &, valiatle这种修饰符全部去掉, 然后再匹配.
看例子

```cpp
int x = 27; 
const int cx = x; 
const int& rx = x; 
f(x); //x是 int, param是T, 所以int匹配到了T, T就是int
f(cx); //cx是const int, param是T, 那么const被忽略, 剩下int, 匹配到T
f(rx); //rx是const int&, param是T, const, & 全被忽略, 剩下int, 匹配到T
```

有一个例外
```cpp
template<typename T>
void f(T param); 
const char* const ptr =  "Fun with pointers";
f(ptr); //param 是T, ptr类型是const char* const,第二个const被忽略,但是第一个
        //被保留, 所以T就是const char* 
```

还有一个要点, 关于数组类型. 其例子如下
```cpp
const char name[] = "J. P. Briggs"; // name 的类型是 const char[13],
                                    // 
const char * ptrToName = name;  //ptrToName的类型是const char*.
```
在这种情况下我们看一下模板传值和传引用的区别
如果我们写了这样一个模板
```cpp
template<typename T>
void f(T param); 
```
然后我们调用
```cpp
f(name)
```
那么T的类型是啥?首先param的类型是T(先写一个抽象的放在这里).然后name的类型虽然是const char[13], 但是因为这里模板是传值,name会被看成指针类型,所以name退化成了const int*,所以T就是const int*,那么param的具体类型也是const int*.

如果我们有这样的模板
```cpp
template<typename T>
void f(T& param); 
```
我们再调用f(name), 那么name的类型是啥呢? 首先param的类型是T& (先写一个抽象的放在这里). 其次因为这里的模板是传引用, 所以name的原始类型const char[13]不会退化, const char[13]直接和T&匹配, 所以T就是const char[13],数组的长度得以保留. 那么param的具体类型是啥? T&就是 const char (&)[13].

根据这一点,我们可以用模板来找出数组的长度
```cpp
template<typename T, std::size_t N> 
constexpr std::size_t arraySize(T (&)[N]) noexcept{
       return N;
} 
```

可以这样调用
```cpp {.line-numbers}
int keyVals[] = { 1, 3, 7, 9, 11, 22, 35 }; 
int mappedVals[arraySize(keyVals)]; 
```

constexpr关键字使得arrySize在编译期就执行了.所以第二行的定义是合法的 

最后一个点, 关于传函数
```cpp
void someFunc(int, double); //someFunc的类型是void (int, double)
```

我们如果有这样两个模板
```cpp
template<typename T>
void f1(T param);

template<typename T>
void f2(T& param);
```
然后
```cpp
f1(someFunc) //param的类型就是void (*)(int, double)
f2(someFunc) //param的类型是 void (&)(int, double)
```

要点1到此全部结束.

# 要点2 理解auto是如如何推导类型的

一句话概括, auto推导类型和模板推导类型是一样的.
看例子

```cpp
auto x = 27;
const auto cx = x;
const auto& rx = x;
```

那么对于第一例子, 我们可以想象编译器会生成以下模板
```cpp
template<typename T> 
void func_for_x(T param); 
func_for_x(27); 
```

27的类型是int, 那么按照要点1, int和T做匹配, T就是int, auto也会推导出来是int

对于第二个例子,我们想象有以下模板

```cpp
template<typename T> 
void func_for_cx(const T param); 
func_for_cx(x);
```

第三个例子我们想象有以下模板

```cpp
template<typename T> 
void func_for_rx(const T& param); 
func_for_rx(x); 
```

这两个例子中x的类型都是int, 然后和 ParamType匹配, T都是int

那么我们回忆要点1, 它把类型推断分成了3类: 1. 是指针或者引用,但不是通用引用. 2. 是通用引用, 3. 既不是引用也不是指针 (不是引用那肯定不是通用引用)

原书中对类1和类3给出了这么几个例子:
```cpp
auto x = 27; // 类3,
const auto cx = x; // 类3
const auto& rx = x; // 类1
```

我们现在回忆一下对于第3类, 模板是怎么推断的呢?
第三类的模板长这个样子
```cpp
template<typename T>
void f(T param); 
```

回忆第三类推断的规则,变量如果有const, 去掉. 如果有&, 去掉. 那么x就剩下了auto, auto和T匹配, 那也就是int.

对于第一类, 模板长这样
```cpp
template<typename T>
void f(T& param); 
```

回忆第一类的推断规则, 如果有&,去掉(注意这里const不会去掉). 那么x就剩下auto去匹配T&, 所以T就是auto, 也就是int.

那么对于第二类, 原书给出了以下例子
```cpp
auto&& uref1 = x; // x是lvalue所以auto推断出来应该是
                  //一个lvalue的引用, 那也就是int&
 
auto&& uref2 = cx; // cx是lvalue, 那么auto推断出来
                   //应该是lvalue引用, const要被保留,
                   //所以是const int&
 
auto&& uref3 = 27; // 27是rvalue, 那么auto推断出来应该是
                   //rvalue的引用, 那就是int&&
```

还有对于数组和函数的类型推断,比较简单, 直接看例子
```cpp
const char name[] =  "R. N. Briggs"; //name类型const char[13]
auto arr1 = name; // arr1类型是const char*, 数组退化成了指针,长度没了
auto& arr2 = name; // arr2类型是 const char (&)[13], 长度保留
void someFunc(int, double); // someFunc 类型是 void(int, double)
auto func1 = someFunc; // func1 类型是 void (*)(int, double)函数名也退化成了指针
auto& func2 = someFunc; // func2's type is void (&)(int, double)
```

以上是auto和模板类型推断的相似点, 下面我们来说一下不同点:
一句话概括:如果用了大括号{}, 那么auto推断出来的结果就和模板不一样了, 看例子:

```cpp
auto x1 = 27; // auto就是int
auto x2(27); // auto就是int
auto x3 = { 27 }; // auto推断出来的类型是std::initializer_list<int>,
                  // 值是 is { 27 }
auto x4{ 27 }; // 和上例一样
```

