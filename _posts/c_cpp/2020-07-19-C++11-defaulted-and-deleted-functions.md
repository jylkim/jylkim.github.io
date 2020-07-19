---
layout: single
title:  "[C++11] Defaulted and deleted functions"
toc: true
category: [C/C++]
tags: [C++, Mordern C++, C++11, function]
---

함수 정의에서 ```default``` 와 ```delete``` 키워드는 자동 생성되는 특수 멤버 함수 (special member functions) 의 명시적인 제어가 필요할 때 사용합니다. ```delete``` 의 경우 특수 멤버 함수 뿐만 아니라 모든 종류의 함수에서도 사용이 가능하며, 원치 않는 함수 호출로 인해 발싱할 수 있는 문제를 차단할 수 있습니다.

# How it works

## Special member functions

C++ 에서 class 를 생성간 정의가 되어 있지 않아도 컴파일러가 알아서 생성해주는 멤버 함수들이 있습니다. 이를 특수 멤버 함수라 부르는데요, 

* 기본 생성자 (default constructor)
* 소멸자 (destructor)
* 복사 생성자 (copy constructor)
* 복사 연산자 (copy assignment)

가 해당됩니다. 여기에 C++11 부터 move semantic 이 생기면서 특수 멤버 함수로 두 함수가 추가되었습니다.

* 이동 생성자 (move constructor)
* 이동 연산자 (move assignment)

## Rule of five 

컴파일러가 알아서 해준다는 것은 개발자 입장에서 편의가 제공된다고 생각될 수 있지만, 다른 시각으로는 개발자가 의도한 방향이 아닌 곳으로 프로그램이 튈 수 있다는 것도 의미합니다. C++ 의 경우 컴파일러가 개발자가 작성한 코드에 개입을 할 수 있어 프로그램이 개발자가 예상치 못한 방향으로 흘러갈 가능성이 존재합니다.

위 특수 멤버 함수들도 마찬가지인데, 함수가 알아서 동작하기 때문에 구현의 수고를 덜 수 있다고 생각할 수 있지만 컴파일러가 암시적으로 생성한 코드를 개발자가 인지를 하지 않을 가능성이 높고, 이로 인해 프로그램이 엉뚱한 방향으로 흘러가 어디서부터 디버깅을 시작해야 할 지 추측조차 어려운 상태가 됩니다.

많은 경우 포인터의 소유권 관련 동작을 개발자가 명시적으로 정의하지 않아 문제가 발생합니다. 컴파일러의 암시적인 기본 구현은 인스턴스 제거나 deep copy 를 하지 않는데, 이후 코드가 인스턴스 제거나 deep copy 로 가정하고 작성이 되었다면 미정의 동작이 발생하는 것이지요.

```cpp
class Foo {
public:
    Foo() : m_bar(new Bar()) {}
    ~Foo() { delete m_bar; }
    // Copy constructor 를 구현하지 않았기에 컴파일러의 기본 구현으로 대체
private:
    Bar* m_bar;
};

Foo* foo1 = new Foo;
Foo* foo2 = new Foo(*foo1); // copy constructor 를 구현하지 않았으며 컴파일러가 자동으로 shallow copy 를 적용
delete foo1;
delete foo2; //m_bar 인스턴스의 double deletion 발생
```

이러한 문제 때문에 rule of five (C++11 이전은 rule of three) 라는 규칙을 만들었습니다. 소멸자, 복사 생성자, 복사 연산자, 이동 생성자, 이동 연산자는 반드시 개발자가 정의 및 구현을 명시적으로 해야 한다는 것이죠.

그리고 이번에 다룰 ```default``` 와 ```delete``` rule of five 처럼 함수 구현을 명시적으로 해야 하는 규칙들을 코드에 적용하는 데 도움을 주는 키워드입니다.

## Defaulted functions

특수 멤버 함수의 구현을 반드시 하자! 라고 다짐을 했지만 꼭 이를 따라야 하나 싶은 상황이 있습니다. 예로 들어 멤버 변수가 없는 class 의 기본 생성자나 멤버 변수가 primitive type 으로만 구성된 class 의 복사 생성자 같은 경우 개발자의 구현은 컴파일러의 기본 구현과 다를 게 없습니다. 되려 해당 생성자들을 일일히 구현하자니 시간이 아깝고요.

이러한 명시적 구현의 필요성과 구현 비용의 모순 ```default``` 로 해결할 수 있습니다. Defaulted function 은 명시적으로 해당 특수 멤버 함수를 컴파일러의 기본 구현로 지정하겠다고 선언합니다.

```cpp
class Foo {
public:
    Foo() = default; // Foo class 에서 초기화할 게 없기 때문에 default function 으로 지정
    virtual void bar() = 0;
};
```

## Deleted functions

C++ 에서는 class 인스턴스의 복사를 권장하지 않는 것이 보통의 관례입니다. 위에서 언급했던 포인터 소유권 이슈로 인해 shallow copy 는 엄청난 양의 버그를 낼 수 있고, deep copy 는 성능의 문제가 있으니까요.

그런데 C++11 전까지는 복사를 허용하지 않는다고 언어적으로 명시할 수 있는 방법이 없었습니다. ```boost::noncopyable``` 이나 링킹 관계를 끊는 방법들이 패턴으로 전해져 오나 C++ 언어에서 제공하는 해결책은 아니기에 명쾌한 방법은 아닙니다. 되려 다중상속의 가능성이나 링킹 에러로 오는 디버깅의 불편함이 존재할 수 있는 상황이죠.

```cpp
// boost::noncopyable 상속을 통한 복사 방지
#include <boost/noncopyable.hpp>

class Foo : private boost::noncopyable { // class 상속관계 설계 간 다중상속의 위험성 존재
    ...
};

// 복사 생성자의 선언만 있고 정의를 없애 링킹이 불가능하도록 구현 
class Bar {
    ...
private:
    Bar(const Bar&); // Bar class 의 멤버 함수에서 복사 생성자 호출 시 링킹 에러 발생 
    ...
};
```

C++11 부터 deleted function 개념이 추가되면서 드디어 언어적으로 명시할 수 있게 되었습니다. ```delete``` 를 명시함으로 컴파일러에게 직접 이 함수를 사용할 수 없음을 알려주고, 컴파일 간 deleted function 의 호출 여부를 확인할 수 있게 되어 위의 꼼수(?) 가 가져올 수 있는 문제점들을 해결할 수 있게 되었지요.

```cpp
class Foo { // 복사를 막기위한 불필요한 class 상속이 없음
    ...
private:
    Foo(const Foo&) = delete; // Foo class 의 멤버 함수에서 복사 생성자 호출 시 컴파일 에러 발생
    ...
};
```

# Notes

* Constructor 에 default/delete 를 적용하면 컴파일러가 자동으로 assigment 에도 적용을 해줍니다.

* Defaulted function 은 컴파일러에게도 기분이 좋은 함수입니다. 굳이 함수의 구현체를 찾을 필요 없이 바로 컴파일러 기본 구현체를 적용할 수 있기 때문에 컴파일 성능의 향상을 기대할 수 있습니다.

* Deleted function 은 defaulted function 과 다르게 특수 멤버 함수 뿐만 아니라 일반 멤버 함수 또는 비멤버 함수에도 적용이 가능합니다. 예로 heap allocation 을 막을 수도 있고, 타입 캐스팅을 통한 의도치 않은 함수 호출도 예방할 수 있습니다.

```cpp
class Foo {
    ...
    void* operator new(std::size_t) = delete; // new Foo() 불가
    ...
};

...

void call_with_double(float) = delete; // 인자로 float 을 넣을 때 double 로 type casting 되는 것을 방지
void call_with_double(double value) {}

...

// 위 type casting 방지의 업그레이드 버전
template<typename T>
void call_with_double_only(T) = delete;
// double 을 제외한 어떤 타입의 입력을 불허
void call_with_double_only(double value) {} // double& 같은 타입의 입력을 하용하고 싶으면 함수를 추가로 선언
```

# Example

```cpp
class Foo {
public:
    Foo() = default;
    Foo(int, int) = default; // 특수 멤버 함수가 아니기에 컴파일 에러 발생
    ~Foo();

    int do_something() = default; // 특수 멤버 함수가 아니기에 컴파일 에러 발생

    Foo(const Foo&) = delete;
    Foo& operator=(const Foo&);
    ...
};

Foo::~Foo() = default; // out-of-line 으로도 default 사용 가능
Foo& Foo::operator=(const Foo&) = delete; // out-of-line 으로 delete 사용 불가로 컴파일 에러 발생

...

Foo a;
Foo b(a); // delete funciton 으로 인해 컴파일 에러 발생
```