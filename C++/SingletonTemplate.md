---
title: SingletonTemplate
date: 2022-01-24 19:30:20
tags: C/C++
categories: C/C++
description: C/C++单实例模板。
---

```c++
#include <mutex>
#include <iostream>

template<typename T>
class singleton
{
public:
	static inline T* instance()
	{
		if (instance_ == 0)
		{
			lock_.lock();
			if (instance_ == 0)
			{
				instance_ = new T;
				static CGarbo cGarbo;
			}
			lock_.unlock();
		}
		return instance_;
	}
	
protected:
	singleton() {}
	virtual ~singleton()
	{
		if (!free_)
		{
			lock_.lock();
			if (instance_ != 0)
			{
				delete instance_;
				instance_ = 0;
			}
			lock_.unlock();
		}
	}

private:
	static inline void free()
	{
		free_ = true;
		lock_.lock();
		if (instance_ != 0)
		{
			delete instance_;
			instance_ = 0;
		}
		lock_.unlock();
	}

private:
	class CGarbo
	{
	public:
		~CGarbo() {
			singleton<T>::free();
		}
	};


private:
	singleton(const singleton&) = delete;// {}
	singleton& operator=(const singleton&) = delete;// {}

private:
	static T* instance_;
	static std::mutex lock_;
	static bool free_;
};

template<typename T>
T* singleton<T>::instance_ = 0;

template<typename T>
std::mutex singleton<T>::lock_;

template<typename T>
bool singleton<T>::free_ = false;
```

**Example-1:**

```c++
class MyClass
{
public:
	void Print(){
		std::cout << "MyClass::Print()" << std::endl;
	}
};

typedef singleton<MyClass> MyClassSingleton;

int main()
{
	MyClassSingleton::instance()->Print();
	
	//MyClassSingleton::free();
	
	return 0;
}
```

**Example-2:**

```c++
class MyClass : public singleton<MyClass>
{
public:
	void Print(){
		std::cout << "MyClass::Print()" << std::endl;
	}
};

int main()
{
	MyClass::instance()->Print();
	
	//MyClass::free();
	
	return 0;
}
```

