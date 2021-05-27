---
title: 'std::function与std::bind使用总结'
date: 2021-05-28 01:35:49
tags: 
- std::function
- std::bind,函数指针
- C++11
---
C++中函数指针的用途非常广泛，例如回调函数，接口类的设计等，但函数指针始终不太灵活，它只能指向全局或静态函数，对于类成员函数、lambda表达式或其他可调用对象就无能为力了，因此，C++11推出了std::function与std::bind这两件大杀器。

## std::function vs 函数指针

C++函数指针相信大家用的很多了，用法最广泛的应该就是先定义函数指针的类型，然后在声明一个函数指针的变量作为另一个函数的入参，以此作为回调函数，如下列代码所示：

```
typedef void (*PrintFinCallback)();
void print(const char *text, PrintFinCallback callback) {
    printf("%s\n", text);
    callback();
}

void printFinCallback() {
    cout << "hhh" << endl;
}
print("test", printFinCallback);
```

毫无疑问，函数指针的用法非常简单，但是它只能指向全局或静态函数，这有点太不灵活了，而且我们都知道在C/C++中，全局的东西都很可怕，稍有不慎就会被篡改或随便调用。幸好，在C++11之后，我们多了一种选择，std::function，使用它时需要引入头文件functional。std::function可以说是函数指针的超集，它除了可以指向全局和静态函数，还可以指向彷函数，lambda表达式，类成员函数，甚至函数签名不一致的函数，可以说几乎所有可以调用的对象都可以当做std::function，当然对于后两个需要使用std::bind进行配合，而至于指向其他类型可以参考以下代码：

```
typedef std::function<void ()> PrintFinFunction;
void print(const char *text, PrintFinFunction callback) {
    printf("%s\n", text);
    if (callback)
        callback();
}
// 普通函数
void printFinCallback() {
    cout << "Normal callback" << endl;
}
// 类静态函数
class Test {
public:
    static void printFinCallback() {
        cout << "Static callback" << endl;
    }
};
// 仿函数，重载()运算符
struct Functor {
    void operator() () {
        cout << "Functor callback" << endl;
    }
};
print("test 1", printFinCallback);
print("test 2", Test::printFinCallback);
print("test 3", Functor());
print("test 4", [] () {
	cout << "Lambda callback" << endl;
});
```

当然，任何东西都会有优缺点，std::function填补了函数指针的灵活性，但会对调用性能有一定损耗，经测试发现，在调用次数达10亿次时，函数指针比直接调用要慢2秒左右，而std::function要比函数指针慢2秒左右，这么少的损耗如果是对于调用次数并不高的函数，替换成std::function绝对是划得来的。

## std::function与std::bind双剑合璧

刚才也说道，std::function可以指向类成员函数和函数签名不一样的函数，其实，这两种函数都是一样的，因为类成员函数都有一个默认的参数，this，作为第一个参数，这就导致了类成员函数不能直接赋值给std::function，这时候我们就需要std::bind了，简言之，std::bind的作用就是转换函数签名，将缺少的参数补上，将多了的参数去掉，甚至还可以交换原来函数参数的位置，具体用法如下列代码所示：

```
typedef std::function<void (int)> PrintFinFunction;
void print(const char *text, PrintFinFunction callback) {
    printf("%s\n", text);
    if (callback)
        callback(0);
}
// 类成员函数
class Test {
public:
    void printFinCallbackInter(int res) {
        cout << "Class Inter callback" << endl;
    }
};
// 函数签名不一样的函数
void printFinCallback2(int res1, int res2) {
    cout << "Different callback " << res1 << " " << res2 << endl;
}
Test testObj;
auto callback5 = std::bind(&Test::printFinCallbackInter, testObj, std::placeholders::_1);
print("test 5", callback5); //函数模板只有一个参数，这里需要补充this参数
auto callback6 = std::bind(&printFinCallback2, std::placeholders::_1, 100);
print("test 6", callback6); //这里需要补充第二个参数
```

从上面的代码中可以看到，std::bind的用法就是第一个参数是要被指向的函数的地址，为了区分，这里std::bind语句的左值函数为原函数，右值函数为新函数，那么std::bind方法从第二个参数起，都是新函数所需要的参数，缺一不可，而我们可以使用std::placeholders::\_1或std::placeholders::\_2等等来使用原函数的参数，\_1就是原函数的第一个参数，如此类推。

值得注意的有两点：

- 一旦bind补充了缺失的参数，那么以后每次调用这个function时，那些原本缺失的参数都是一样的，举个栗子，上面代码中callback6，我们每次调用它的时候，第二个参数都只会是100。
- 正因为第一点，所以假如我们是在iOS程序中使用std::bind传入一个缺失参数，那么我们转化后的那个function会持有那些缺失参数，这里我们需要防止出现循环引用导致内存泄漏。

## lambda表达式

lambda表达式其实也就是匿名函数，而Python、Java都有了自己lambda表达式，那么作为古老的语言C++同样也不能落后，C++11也推出了自己的lambda表达式语法，如下所示：

**capture->return-type{body}**

语法分析：

1. 方括号内是匿名函数内要捕获的外部变量，而且分为值捕获和引用捕获，下面列出了几种捕获变量的写法：
    - =  匿名函数内所用到的外部变量都按值传递
    - & 匿名函数内所用到的外部变量都按引用传递
    - &, a, b 匿名函数内除了a和b是按值传递，其他变量都是按引用传递
    - =, &a, &b 匿名函数内除了a和b是按引用传递，其他变量都是按值传递
    - a, &b 匿名函数只捕获了a和b两个外部变量，其中a是按值传递，b是按引用传递
2. 圆括号内是匿名函数的所需要的参数，也可以分为按值传递和按引用传递两种方式，某种意义上说，方括号中捕获的外部变量其实也可以作为参数传入，只是更为方便罢了。
3. 箭头后面是返回值类型，如果返回值类型为void，箭头和返回值类型都可以省略，如第一部分给出的例子一样。
4. 函数体在花括号范围内。
跟std::bind一样，如果我们在iOS中使用lambda表达式，而且函数体内捕获了外部变量，我们需要注意避免出现循环引用。

## 总结

以上是笔者这几天学习C++11关于函数指针的改进技术的总结，希望各位大神不吝指正，也希望对各位的开发有所帮助。