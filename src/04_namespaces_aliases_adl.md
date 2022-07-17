## Квалифицированный поиск

```cpp
#include <iostream>
 
int main() {
    struct std {};
    std::cout << "fail"; // Error: unqualified lookup
    ::std::cout << "ok"; // OK, qualified
}
```

```cpp
struct A {
    static int n;
};
 
int main() {
    int A;
    A::n = 42; // OK: ignoring the variable
    A b;       // Error: unqualified lookup of A finds the variable A
}
```

Квалифицированный поиск может быть использован для доступа к членам класса, которые были скрыты вложенным объявлением или содержатся в наследуемом классе. Вызов квалифицированного метода класса не учитывает виртуальное объявление.

```cpp
struct B {virtual void foo();};
 
struct D : B {void foo() override;};
 
int main() {
    D x;
    B& b = x;
    b.foo();    // calls D::foo (virtual dispatch)
    b.B::foo(); // calls B::foo (static dispatch)
}
```

## Псевдонимы пространств имен

Синтаксис:

**namespace** *alias_name* = *ns_name*
**namespace** *alias_name* = ::*ns_name*
**namespace** *alias_name* = *nested_name*::*ns_name*

Псевдоним *alias_name* будет виден в том блоке, в котором он объявлен.

## Using-директива

Синтаксис:

**using namespace** *ns_name*

С точки зрения неквалифицированного поиска имен любой символ, объявленный в *ns_name*, после using-директивы и до конца блока ее объявления будет виден, как если бы он был объявлен в ближайшем пространстве имен, которое одновременно содержит как using-директиву, так и *ns-name*.

## Using-объявление

Синтаксис:

**using** *ns_name*::*member_name*

Делает символ *member-name* из пространства имен *ns-name* доступным для неквалифицированного поиска, как если бы он был объявлен в том же классе, блоке или пространстве имен, в котором using-объявление появляется.

## Анонимные пространства имен

Синтаксис:

**namespace** { ```namespace-body``` }

Пример кода:

```cpp
namespace {
    int i; // defines ::(unique)::i
}

void f() {
    i++;   // increments ::(unique)::i
}
 
namespace A {
    namespace {
        int i;        // A::(unique)::i
        int j;        // A::(unique)::j
    }
    void g() { i++; } // A::(unique)::i++
}
 
using namespace A; // introduces all names from A into global namespace

void h() {
    i++;    // error: ::(unique)::i and ::A::(unique)::i are both in scope
    A::i++; // ok, increments ::A::(unique)::i
    j++;    // ok, increments ::A::(unique)::j
}

```

Символы внутри анонимных пространств имен (так же как и любые символы, объявленные внутри именованных пространств внутри других анонимных) имеют внутреннее связывание, как если бы им приписали `static`.

## ADL (Argument-dependent lookup)

Позволяет не указывать явно пространство имен функций при вызове, если какие-либо их аргументы объявлены в том же пространстве имен.

```cpp
namespace MyNamespace {
    class MyClass {};
    void doSomething(MyClass) {}
}

MyNamespace::MyClass obj; // global object


int main() {
    doSomething(obj); // works fine - MyNamespace::doSomething() is called.
}
```

Такой поиск запускается только для имён, у которых имя неквалифицированное.

Важно знать, что у поиска ADL и обычного нет приоритетов, они запускаются оба и кандидаты из обоих уходят в overload resolution. Обычно они либо оба найдут одно и то же, либо обычный не найдет ничего и возьмется из ADL.

Но есть такой tricky пример кода:

```cpp
using std::swap;
swap(obj1, obj2);
```

Если существует пространство имен `A` и если существуют `A::obj1`, `A::obj2` и `A::swap()`, то произойдет вызов `A::swap()` вместо вызова `std::swap()`, чего разработчик иногда может не ожидать. *todo: с чем связано?*

---

Заимствования:

[Argument-dependent lookup - cppreference.com](https://en.cppreference.com/w/cpp/language/adl)