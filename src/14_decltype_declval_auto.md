## decltype

Иногда возвращаемый тип не хочется писать руками:

```cpp
int f();

??? g() {
    return f();
}
```

В C++11 появилась конструкция, позволяющая по выражению узнать его тип:

```cpp
int main() {
    decltype(2 + 2) a = 42; // int a = 42;
}

decltype(f()) g() {
    return f();
}
```

`decltype` сделан так, чтобы его было удобно использовать для возврата значений:

```cpp
int foo();
int& bar();
int&& baz();

// decltype(foo()) -> int
// decltype(bar()) -> int&
// decltype(baz()) -> int&&
```

То есть `decltype(expr)` возвращает следующие типы:

* type для `prvalue`
* type& для `lvalue`
* type&& для `xvalue`

Также ключевое слово `decltype` применимо для переменных и членов класса.

```cpp
struct x {
    int a;
};

int main() {
    decltype(x::a) y; // int y

    int a;
    decltype(a) b = 42; // int b = 42
    decltype((a)) c = a; // int & c = a
}
```

Последнее работает так, потому что `(a)` - это выражение, и оно имеет тип int&, а `a` - это имя переменной.

## declval

Иногда хочется узнать тип чего-то, что зависит от шаблонных аргументов функции, но просто сделать это с помощью `decltype` не получится, так как тогда компилятор встречает обращение к параметру функции, когда еще не дошел до его объявления.

Для этого есть синтаксическая конструкция `declval`:

```cpp
int foo(int);
float foo(float);

// compile error
template <typename T>
decltype(foo(t)) f(T&& t) {
    return foo(std::forward<T>(t));
}

// nice.
template <typename T>
decltype(foo(declval<T>())) f(T&& t) {
    return foo(std::forward<T>(t));
}
```

Сигнатура `declval` могла бы выглядеть как-то так:

```cpp
template <typename T>
T declval();
```

Для `declval` не нужно тело функции, так как `decltype` не генерирует машинный код и считается на этапе компиляции.

В языке есть несколько мест с похожей логикой - например, `sizeof`. Такие места называются *unevaluated contexts*.

При использовании сигнатуры, как выше, могут возникать проблемы с неполными типами (просто не скомпилируется). Это происходит из-за того, что если функция возвращает структуру, то в точке, где вызывается функция, эта структура должно быть complete типом. Чтобы обойти это, делают возвращаемый тип rvalue-ссылкой:

```cpp
template <typename T>
T&& declval();
```

### Trailing return types

Чтобы не писать `declval`, сделали возможной следующую конструкцию:

```cpp
template <typename... Args>
auto f(Args&&... args) -> decltype(foo(std::forward<Args>(args)...)) {
    return foo(std::forward<Args>(args)...);
}
```

То есть компилятор, натыкаясь на `decltype` уже знает аргументы, которые передаются в функцию, и может на них опираться. *Очень удобно, конечно (сарказм).*

## auto

Можно заметить, что в `return` и `decltype` повторяется одно и то же выражение. Во избежание копипасты добавили возможность писать `decltype(auto)`.

```cpp
int main() {
    decltype(auto) b = 2 + 2; // int b = 2 + 2;
}

template <typename... Args>
decltype(auto) f(Args&&... args) {
    return foo(std::forward<Args>(args)...);
}
```

Возникает вопрос, а зачем нам вообще `decltype`, можно ли заменить его на просто `auto`? Для этого стоит сказать о том, как работает `auto`.

Правило вывода типов у `auto` почти полностью совпадает с тем, как выводятся шаблонные параметры. Поэтому `auto` отбрасывает ссылки и cv-квалификаторы.

```cpp
int& bar();

int main() {
    auto c = bar(); // int c = bar()
    auto& c = bar(); // int& c = bar()
}
```

Поэтому обычный `auto` в возвращаемом типе отбрасывает ссылки с cv-квалификаторами, поэтому чаще всего нам нужен именно `decltype(auto)`.

Еще стоит сказать, что если у функции несколько инструкций `return`, который выводятся в разные типы, то использовать `decltype` и `auto` нельзя:

```cpp
// compile error
auto f(bool flag) {
    if (flag) {
        return 1;
    } else {
        return 1u;
    }
}
```

---

Заимствования:

[cpp-notes/18_decltype_auto_nullptr.md at master · lejabque/cpp-notes (github.com)](https://github.com/lejabque/cpp-notes/blob/master/src/18_decltype_auto_nullptr.md)

