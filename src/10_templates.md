## Объявление шаблона

Объявление шаблонного класса или шаблонной функции в примере ниже:

```cpp
template <typename T>
struct vector {
    void push_back(T const &);
    T const& operator[](size_t index) const;
    
    template <typename U>
    void g(T, U);
};

template <typename T>
void f(T&& a, T&& b) {
    std::cout << a + b << std::endl;
}
```

Вместо T и U будет подставляться тот тип, который был указан в шаблонном параметре при использовании класса/функции. Также в большинстве случаев компилятор может вывести тип самостоятельно без явного его указания.

```cpp
vector<int> v;
vector<double> v2;

v.template g<float>(42, 42.0);
v.g(42, 42.0); // same

f(1, 3);     // 4
f(1.5, 2.5); // 4.0
```

## Специализации

Специализация - это выделенная реализация для каких-то указанных типов. К примеру из примера выше для `vector<std::string>` и `void f(bool&&, bool&&)` я бы хотел иметь другое тело функции. Это пример надуманный, конечно.

```cpp
template <>
struct vector<std::string> {
    /* ... */
};

template <>
void f<bool>(bool&& a, bool&& b) {
    std::cout << (a | b) << std::endl;
}
```

Специализацию можно вводить не целиком, а частично, оставляя свободными другие шаблонные параметры. В качестве примера рассмотрим простенькую реализацию `std::conditional`.

```cpp
template <bool Cond, typename IfTrue, typename IfFalse>
struct conditional;

template <typename IfTrue1, typename IfFalse1>
struct conditional<false, IfTrue1, IfFalse1> { // partial specialization
    typedef IfFalse1 type;
};

template <typename IfTrue1, typename IfFalse1>
struct conditional<true, IfTrue1, IfFalse1> { // partial specialization
    typedef IfTrue1 type;
};
```

Специализациям свойственен большее высокий приоритет при разрешениях перегрузок (overload resolution).

```cpp
template <typename T>
struct vector<T*> {
    /* ... */
}

vector<foo*> v; // выберется эта специализация
```

Пример неразрешимой перегрузки:

```cpp
template <typename U, typename V>
struct mytype {};

template <typename U, typename V>
struct mytype<U*, V> {};

template <typename U, typename V>
struct mytype<U, V*> {};

mytype<long*, double*> f;
```

Мы получим ошибку, так как есть два равноправных кандидата. Исправить это можно определением еще одной, более подходящей специализации:

```cpp
template <typename U, typename V>
struct mytype<U*, V*> {};
```

## Ошибка разделения на объявление и реализацию

Давайте представим, как мы могли бы писать код с шаблонными функциями, используя разделение на объявление и реализацию, как полагается.

```cpp
// util.h
template <typename T>
void f(T&, T&);

// util.cpp
template <typename T>
void f(T& a, T& b) {
    std::cout << a + b << std::endl;
}

// main.cpp
#include "util.h"
int main(){
    int a, b;
    f(a, b);
}
```

Но в данном примере, к сожалению, мы получим ошибку компиляции. Так происходит, потому что генерация и подстановка кода шаблонов (инстанцирование) происходит до линковки и после компиляции каждой отдельной единицы трансляции. Компилятор, обрабатывая `util.cpp`, не знает о том, что кто-то будет вызывать `f(int, int)` в других единицах трансляции.

На самом деле все шаблонные функции неявно являются `inline`, поэтому их реализацию можно сразу писать в `.h` файле.

## Инстанцирование

В стандарте прописано, что инстанцирование происходит только когда это необходимо. При этом компилятор может делать это в конце единицы трансляции.

В следующем примере приводится случай, который это показывает:

```cpp
template <typename T>
struct foo {
    T* a;
    void f(){
        T a;
    }
};

int main() {
    foo<void> a; // так скомпилируется
    a.f();       // а так нет, ошибка из-за void a
}
```

Генерация и подстановка по требованию, так сказать. С классами работает аналогично: полное тело класса не подставляется, если не требуется. Пример:

```cpp
template <typename T>
struct foo {
    T a;
};

int main() {
    foo<void>* a; // так скомпилируется
    a->a;         // а так нет, опять ошибка из-за void a
}
```

## Явное инстанцирование

Пусть мы не хотим, чтобы одни и те же лишние инстанцирования были в разных единицах трансляции. Чтобы этого избежать, можно делать так:

```cpp
template <typename T>
void foo(T) {}

template void foo<int>(int); // генерирует тело функции в этом месте
template void foo<float>(float);
template void foo<double>(double);
```

## Подавление инстанцирования

Пусть мы знаем, что функции уже где-то инстанцированы и мы не хотим лишних:

```cpp
extern template void foo<int>(int); 
extern template void foo<float>(float);
```

Выдаём тело наружу и говорим, что уже проинстанцировано. `main` не будет пытаться инстанцировать функцию, так как увидит `extern` и будет работать соответствующе.

Теперь зная про шаблоны и специализации можно творить всякую магию:

## Подсчет факториала в compile-time

Осторожно, при отрицательных значениях компилятор может надолго зависнуть.

```cpp
template <int T>
struct factorial {
    static const int result = T * factorial<T - 1>::result;
};


template <>
struct factorial<0> {
    static const int result = 1;
};

static_assert(factorial<0>::result == 1);
static_assert(factorial<3>::result == 6);
static_assert(factorial<5>::result == 120);
```


---

Заимствования:

[cpp-notes/11_templates.md at master · lejabque/cpp-notes (github.com)](https://github.com/lejabque/cpp-notes/blob/master/src/11_templates.md)
