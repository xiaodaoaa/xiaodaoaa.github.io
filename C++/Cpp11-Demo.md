---
title: C++11-Demo
date: 2021-12-06 10:49:38
tags: C++11
categories: C/C++
description: C++11示例代码，乃学习测试时编写，如有问题请留言指正。
---

```cpp
#include <iostream>
#include <map>
#include <vector>
#include <string>
#include <functional>
#include <regex>
#include <iterator>
#include <thread>
#include <chrono>
#include <atomic>
#include <ctime>
#include <future>
#include <tuple>
#include <random>
#include <string>

using namespace std;

namespace TestFunctional {
std::function< int(int)> Functional;

// 普通函数
int TestFunc(int a)
{
    return a;
}

// Lambda表达式
auto lambda = [](int a)->int{ return a; };

// 仿函数(functor)
class Functor
{
public:
    int operator()(int a)
    {
        return a;
    }
};

// 1.类成员函数
// 2.类静态函数
class TestClass
{
public:
    int ClassMember(int a) { return a; }
    static int StaticMember(int a) { return a; }
};

void TestFunctional()
{
    // 普通函数
    Functional = TestFunc;
    int result = Functional(10);
    cout << "普通函数："<< result << endl;

    // Lambda表达式
    Functional = lambda;
    result = Functional(20);
    cout << "Lambda表达式："<< result << endl;

    // 仿函数
    Functor testFunctor;
    Functional = testFunctor;
    result = Functional(30);
    cout << "仿函数："<< result << endl;

    // 类成员函数
    TestClass testObj;
    Functional = std::bind(&TestClass::ClassMember, testObj, std::placeholders::_1);
    result = Functional(40);
    cout << "类成员函数："<< result << endl;

    // 类静态函数
    Functional = TestClass::StaticMember;
    result = Functional(50);
    cout << "类静态函数："<< result << endl;
}
}
namespace TestEnum {
void TestEnum()
{
    enum class Options {None, One, All};
    Options o = Options::All;
    if(Options::All == o)
    {
        cout << "Options::All" << endl;
    }
}
}
namespace TestForeach {
void TestForeach()
{
    std::map<std::string, std::vector<int>> map;
    std::vector<int> v;
    v.push_back(1);
    v.push_back(2);
    v.push_back(3);
    map["one"] = v;

    for(const auto& kvp : map)
    {
        std::cout << kvp.first << std::endl;

        for(auto v : kvp.second)
        {
            std::cout << v << std::endl;
        }
    }

    int arr[] = {1,2,3,4,5};
    for(int& e : arr)
    {
        e = e*e;
    }
}
}
namespace TConditionVariable {
std::mutex mtx;
std::condition_variable cv;
bool ready = false;

void print_id (int id) {
    std::unique_lock<std::mutex> lck(mtx);
    while (!ready) cv.wait(lck);
    // ...
    std::cout << "thread " << id << '\n';
}

void go() {
    std::unique_lock<std::mutex> lck(mtx);
    ready = true;
    cv.notify_all();
}

void TestConditionVariable()
{
    std::thread threads[10];
    // spawn 10 threads:
    for (int i=0; i<10; ++i)
        threads[i] = std::thread(print_id,i);

    std::cout << "Ten threads be created.\n";
    this_thread::sleep_for(std::chrono::seconds(3));
    std::cout << "10 threads ready to race...\n";
    go();                       // go!

    for (auto& th : threads) th.join();
}
}
namespace TestRegex {
void TestRegex()
{
    std::string s = "Some people, when confronted with a problem, think "
                    "\"I know, I'll use regular expressions.\" "
                    "Now they have two problems.";

    std::regex self_regex("REGULAR EXPRESSIONS",
                          std::regex_constants::ECMAScript | std::regex_constants::icase);
    if (std::regex_search(s, self_regex)) {
        std::cout << "Text contains the phrase 'regular expressions'\n";
    }

    std::regex word_regex("(\\S+)");
    auto words_begin =
            std::sregex_iterator(s.begin(), s.end(), word_regex);
    auto words_end = std::sregex_iterator();

    std::cout << "Found "
              << std::distance(words_begin, words_end)
              << " words\n";

    const int N = 6;
    std::cout << "Words longer than " << N << " characters:\n";
    for (std::sregex_iterator i = words_begin; i != words_end; ++i) {
        std::smatch match = *i;
        std::string match_str = match.str();
        if (match_str.size() > N) {
            std::cout << "  " << match_str << '\n';
        }
    }

    std::regex long_word_regex("(\\w{7,})");
    std::string new_s = std::regex_replace(s, long_word_regex, "[$&]");
    std::cout << new_s << '\n';
}

}
namespace TestLambda {
//[capture list] (parameter list) ->return type { function body }
class CLambda
{
public:
    CLambda() : m_nData(20) { NULL; }
    void TestLambda()
    {
        vector<int> vctTemp;
        vctTemp.push_back(1);
        vctTemp.push_back(2);

        // 无函数对象参数，输出：1 2
        {
            for_each(vctTemp.begin(), vctTemp.end(), [](int v){ cout << v << endl; });
        }

        // 以值方式传递作用域内所有可见的局部变量（包括this），输出：11 12
        {
            int a = 10;
            for_each(vctTemp.begin(), vctTemp.end(), [=](int v){ cout << v+a << endl; });
        }

        // 以引用方式传递作用域内所有可见的局部变量（包括this），输出：11 13 12
        {
            int a = 10;
            for_each(vctTemp.begin(), vctTemp.end(), [&](int v)mutable{ cout << v+a << endl; a++; });
            cout << a << endl;
        }

        // 以值方式传递局部变量a，输出：11 13 10
        {
            int a = 10;
            for_each(vctTemp.begin(), vctTemp.end(), [a](int v)mutable{ cout << v+a << endl; a++; });
            cout << a << endl;
        }

        // 以引用方式传递局部变量a，输出：11 13 12
        {
            int a = 10;
            for_each(vctTemp.begin(), vctTemp.end(), [&a](int v){ cout << v+a << endl; a++; });
            cout << a << endl;
        }

        // 传递this，输出：21 22
        {
            for_each(vctTemp.begin(), vctTemp.end(), [this](int v){ cout << v+m_nData << endl; });
        }

        // 除b按引用传递外，其他均按值传递，输出：11 12 17
        {
            int a = 10;
            int b = 15;
            for_each(vctTemp.begin(), vctTemp.end(), [=, &b](int v){ cout << v+a << endl; b++; });
            cout << b << endl;
        }
        // 操作符重载函数参数按引用传递，输出：2 3
        {
            for_each(vctTemp.begin(), vctTemp.end(), [](int &v){ v++; });
            for_each(vctTemp.begin(), vctTemp.end(), [](int v){ cout << v << endl; });
        }
        // 空的Lambda表达式
        {
            [](){}();    []{}();
        }
    }
private:
    int m_nData;
};

void TestLambda()
{
    CLambda o;
    o.TestLambda();
}

}
namespace TestInitializerList {
class MyNumber
{
public:
    MyNumber(const std::initializer_list<int> &v) {
        for (auto itm : v) {
            mVec.push_back(itm);
        }
    }

    void print() {
        for (auto itm : mVec) {
            std::cout << itm << " ";
        }
    }
private:
    std::vector<int> mVec;
};

void TestInitializerList()
{
    MyNumber m = {1, 2, 3, 4, 5};
    m.print();
}

}
namespace TestAtomic {
atomic_int g_Count{0};
atomic_bool g_Ready{false};
atomic_flag g_Winner{ATOMIC_FLAG_INIT};

void IncreaseCount(int id)
{
    cout << "Thread #" << id << " wait main process ready...\n";

    while (!g_Ready) {
        std::this_thread::yield();
    } // 等待主线程中设置 ready 为 true.

    cout << "Thread #" << id << " running..." << endl;

    for (int i = 0; i < 1000; ++i) {
        g_Count++;
        //cout << "g_Count:" << g_Count << endl;
    } // 计数.

    cout << "Thread #" << id << " count over." << endl;

    // 如果某个线程率先执行完上面的计数过程，则输出自己的 ID.
    // 此后其他线程执行 test_and_set 是 if 语句判断为 false，
    // 因此不会输出自身 ID.
    if (!g_Winner.test_and_set()) {
        std::cout << "Thread #" << id << " won!\n";
    }
}

void TestAtomic()
{
    std::cout << "Number of threads = " <<  std::thread::hardware_concurrency() << std::endl;

    std::vector<std::thread> threads;
    std::cout << "spawning 10 threads that count to 1 million...\n";
    for (int i = 1; i <= 10; ++i)
        threads.push_back(std::thread(IncreaseCount, i));

    std::this_thread::sleep_for(std::chrono::seconds(3));

    cout << "Ready count.\n";
    g_Ready = true;

    for (auto &th : threads)
        th.join();
}

}
namespace TestFuture {
// a non-optimized way of checking for prime numbers:
bool isPrime (int x)
{
    for (int i = 2; i < x; ++i)
    {
        //this_thread::sleep_for(std::chrono::milliseconds(1));
        if (x % i == 0)
            return false;
    }
    return true;
}

void TestFuture()
{
    // call function asynchronously:
    std::future<bool> fut = std::async(isPrime, 444444443);
    //std::shared_future<bool> shafut = fut.share();

    // do something while waiting for function to set future:
    std::cout << "checking, please wait";
    std::chrono::milliseconds span (100);
    while (fut.wait_for(span) == std::future_status::timeout)
        std::cout << '.' << std::flush;

    bool x = fut.get();     // retrieve return value

    std::cout << "\n444444443 " << (x ? "is" : "is not") << " prime.\n";

    //std::cout << "shafut.get : " << (shafut.get() ? "true" : "false") << endl;
}

}
namespace TestSharedptr {
void TestSharedptr()
{
    std::shared_ptr<int> p = nullptr;
    std::shared_ptr<int> pp = nullptr;
    {
        std::shared_ptr<int> p1 = std::make_shared<int>(10);
        std::shared_ptr<int> p2(new int[10], std::default_delete<int[]>());

        auto deleteInt = [](int *p){ delete []p; cout << "Delete Int array." << endl; };
        std::shared_ptr<int> p3(new int[10], deleteInt);

        cout << "p3.use_count : " << p3.use_count() << endl;
        cout << "p.use_count : " << p.use_count() << endl;
        p = std::move(p3);
        cout << "p3.use_count : " << p3.use_count() << endl;
        cout << "p.use_count : " << p.use_count() << endl;

        std::shared_ptr<int> p4(new int[10], deleteInt);
        cout << "p4.use_count : " << p4.use_count() << endl;
        cout << "pp.use_count : " << pp.use_count() << endl;
        pp = p4;
        cout << "p4.use_count : " << p4.use_count() << endl;
        cout << "pp.use_count : " << pp.use_count() << endl;
    }

    cout << "O-pp.use_count : " << pp.use_count() << endl;

    cout << "TestSharedptr end." << endl;
}

}
namespace TestUsing {
class CSingleton
{
public:
    static CSingleton &GetInstance(){ static CSingleton ins; return ins; }

    void Print(){ cout << "CSingleton~!" << endl; }
};

struct tagUsingStruct
{
    int a;
    tagUsingStruct() {}
};
using UsingStruct = tagUsingStruct;

using g_CSingleton = CSingleton;
void TestUsing()
{
    g_CSingleton::GetInstance().Print();

    UsingStruct stUsingStruct;
    stUsingStruct.a = 99;
    cout << stUsingStruct.a << endl;
}

}
namespace TestDecltype {
void TestDecltype()
{
    int i = 33;
    decltype(i) j = i * 2;

    std::cout << "i and j are the same type? " << std::boolalpha
              << std::is_same_v<decltype(i), decltype(j)> << '\n';

    std::cout << "i = " << i << ", "
              << "j = " << j << '\n';

    auto f = [](int a, int b) -> int
    {
        return a * b;
    };

    decltype(f) g = f; // the type of a lambda function is unique and unnamed
    i = f(2, 2);
    j = g(3, 3);

    std::cout << "i = " << i << ", "
              << "j = " << j << '\n';
}
}
namespace TestUniqueptr {
struct Task {
    int mId;
    Task(int id ) :mId(id) {
        std::cout<<"Task::Constructor"<<std::endl;
    }
    ~Task() {
        std::cout<<"Task::Destructor"<<std::endl;
    }
};

void TestUniqueptr()
{
    // 空对象 unique_ptr
    std::unique_ptr<int> ptr1;

    // 检查 ptr1 是否为空
    if(!ptr1)
        std::cout<<"ptr1 is empty"<<std::endl;

    // 检查 ptr1 是否为空
    if(ptr1 == nullptr)
        std::cout<<"ptr1 is empty"<<std::endl;

    // 不能通过赋值初始化unique_ptr
    // std::unique_ptr<Task> taskPtr2 = new Task(); // Compile Error

    // 通过原始指针创建 unique_ptr
    std::unique_ptr<Task> taskPtr(new Task(23));

    // 检查 taskPtr 是否为空
    if(taskPtr != nullptr)
        std::cout<<"taskPtr is  not empty"<<std::endl;

    // 访问 unique_ptr关联指针的成员
    std::cout<<taskPtr->mId<<std::endl;

    std::cout<<"Reset the taskPtr"<<std::endl;
    // 重置 unique_ptr 为空，将删除关联的原始指针
    taskPtr.reset();

    // 检查是否为空 / 检查有没有关联的原始指针
    if(taskPtr == nullptr)
        std::cout<<"taskPtr is  empty"<<std::endl;

    // 通过原始指针创建 unique_ptr
    std::unique_ptr<Task> taskPtr2(new Task(55));

    if(taskPtr2 != nullptr)
        std::cout<<"taskPtr2 is  not empty"<<std::endl;

    // unique_ptr 对象不能复制
    //taskPtr = taskPtr2; //compile error
    //std::unique_ptr<Task> taskPtr3 = taskPtr2;

    {
        // 转移所有权（把unique_ptr中的指针转移到另一个unique_ptr中）
        std::unique_ptr<Task> taskPtr4 = std::move(taskPtr2);
        // 转移后为空
        if(taskPtr2 == nullptr)
            std::cout << "taskPtr2 is  empty" << std::endl;
        // 转进来后非空
        if(taskPtr4 != nullptr)
            std::cout<<"taskPtr4 is not empty"<<std::endl;

        std::cout << taskPtr4->mId << std::endl;

        //taskPtr4 超出下面这个括号的作用于将delete其关联的指针
    }

    std::unique_ptr<Task> taskPtr5(new Task(66));

    if(taskPtr5 != nullptr)
        std::cout << "taskPtr5 is not empty" << std::endl;

    // 释放所有权
    Task * ptr = taskPtr5.release();

    if(taskPtr5 == nullptr)
        std::cout << "taskPtr5 is empty" << std::endl;

    std::cout << ptr->mId << std::endl;

    delete ptr;
}
}
namespace TestWeakptr {
//strong reference
class B;
class A
{
public:
    shared_ptr<class B> m_spB;
};

class B
{
public:
    shared_ptr<class A> m_spA;
};

//weak reference
class WeakB;
class WeakA
{
public:
    weak_ptr<class WeakB> m_wpB;
};

class WeakB
{
public:
    weak_ptr<class WeakA> m_wpA;
};


void TestLoopRef()
{
    weak_ptr<class A> wp1;

    {
        auto pA = std::make_shared<class A>();
        auto pB = std::make_shared<class B>();

        pA->m_spB = pB;
        pB->m_spA = pA;

        wp1 = pA;
    }//内存泄漏

    cout << "wp1 reference number: " << wp1.use_count() << "\n";

    weak_ptr<class WeakA> wp2;
    {
        auto pA = std::make_shared<class WeakA>();
        auto pB = std::make_shared<class WeakB>();

        pA->m_wpB = pB;
        pB->m_wpA = pA;

        wp2 = pA;
    }//无内存泄漏

    cout << "wp2 reference number: " << wp2.use_count() << "\n";
}

void TestWeakptr()
{
    //default consstructor
    weak_ptr<string> wp;

    {
        shared_ptr<string> p = make_shared<string>("hello world!\n");
        //weak_ptr对象也绑定到shared_ptr所指向的对象。
        wp = p;
        cout << "use_count: " <<wp.use_count() << endl;
    }
    //wp是弱类型的智能指针，不影响所指向对象的生命周期，
    //这里p已经析构，其所指的对象也析构了，因此输出是0
    cout << "use_count: " << wp.use_count() << endl;

    TestLoopRef();

}
}
namespace TestTuple {
void TestTuple()
{
    int size;
    //创建一个 tuple 对象存储 10 和 'x'
    std::tuple<int, char> mytuple(10, 'x');
    //计算 mytuple 存储元素的个数
    size = std::tuple_size<decltype(mytuple)>::value;
    cout << "size:" << size << endl;
    //输出 mytuple 中存储的元素
    std::cout << std::get<0>(mytuple) << " " << std::get<1>(mytuple) << std::endl;
    //修改指定的元素
    std::get<0>(mytuple) = 100;
    std::cout << std::get<0>(mytuple) << std::endl;
    //使用 makde_tuple() 创建一个 tuple 对象
    auto bar = std::make_tuple("test", 3.1, 14);
    //拆解 bar 对象，分别赋值给 mystr、mydou、myint
    const char* mystr = nullptr;
    double mydou;
    int myint;
    //使用 tie() 时，如果不想接受某个元素的值，实参可以用 std::ignore 代替
    std::tie(mystr, mydou, myint) = bar;
    //std::tie(std::ignore, std::ignore, myint) = bar;  //只接收第 3 个整形值
    cout << mydou << ", " << myint << endl;
    //将 mytuple 和 bar 中的元素整合到 1 个 tuple 对象中
    auto mycat = std::tuple_cat(mytuple, bar);
    size = std::tuple_size<decltype(mycat)>::value;
    std::cout << size << std::endl;
}
}
namespace TestFunctionBind {
using CallbackType = std::function<void(string)>;

class CPrint{
public:
    CPrint()
    {
        mCallbacks[66] = std::bind(&CPrint::PrintHello, this, std::placeholders::_1);
        mCallbacks[99] = std::bind(&CPrint::PrintWorld, this, std::placeholders::_1);
    }

    void run()
    {
//        auto it = mCallbacks.begin();
//        for(; it != mCallbacks.end(); it++)
//        {
//            it->second(" ");
//        }
        mCallbacks[66](" ");
        mCallbacks[99](" !!!\n");
    }

private:
    void PrintHello(string &msg)
    {
        cout << "Hello" << msg;
    }

    void PrintWorld(string &msg)
    {
        cout << "World" << msg;
    }

private:
    map<int, CallbackType> mCallbacks;
};

void TestFunctionBind()
{
    CPrint().run();
}
}
namespace TestRandom {
void TestRandom(){
    using namespace std::chrono;
    //随机非负数
    default_random_engine e;
    for(int i=0; i<10; ++i)
        cout<<e()<<endl;
    cout << endl;

    //使用种子 s 充值 e 的状态
//    time_t seed = time(nullptr);
    system_clock::duration d = system_clock::now().time_since_epoch();
    seconds sec = duration_cast<seconds>(d);
    int64_t seed = sec.count();
    cout << "seed:" << seed << endl << endl;
    e.seed((uint32_t)seed);
    for(int i=0; i<10; ++i)
        cout<<e()<<endl;
    cout << endl;

    //特定范围的非负数,0~9
    uniform_int_distribution<unsigned> uintR(0, 9);
    for(int i=0; i<10; ++i)
        cout<<uintR(e)<<endl;
    cout << endl;

    //随机浮点数,范围0.0~1.0
    uniform_real_distribution<double> doubleR(0.0, 1.0);
    for(int i=0; i<10; ++i)
        cout<<doubleR(e)<<endl;
    cout << endl;

    //随机布尔值
    bernoulli_distribution boolR;
    for(int i=0; i<10; ++i)
        cout<<boolR(e)<<endl;
    cout << endl;
}
}

int main()
{
    TestRandom::TestRandom();
//    TestFunctionBind::TestFunctionBind();
//    TestTuple::TestTuple();
//    TestWeakptr::TestWeakptr();
//    TestUniqueptr::TestUniqueptr();
//    TestDecltype::TestDecltype();
//    TConditionVariable::TestConditionVariable();
//    TestForeach::TestForeach();
//    TestEnum::TestEnum();
//    TestFunctional::TestFunctional();
//    TestRegex::TestRegex();
//    TestLambda::TestLambda();
//    TestInitializerList::TestInitializerList();
//    TestUsing::TestUsing();
//    TestSharedptr::TestSharedptr();
//    TestAtomic::TestAtomic();
//    TestFuture::TestFuture();

    return 0;
}

```
