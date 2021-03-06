## Исключения

Исключительная ситуация - следствие вызова операции с ошибкой, которая не может или не должна быть проигнорирована.

Например, деление на ноль вне типа `double`. К чему это должно привести? Непонятно, поэтому кидаем исключение.

Другой пример: `dynamic_cast` не смог привести тип к другой ссылке. Что нужно вернуть, чтобы принимающий знал, что операция завершилась неудачей? Нулевой ссылки не бывает, значит, кидаем исключение.

Для генерации исключения в C++ используется слово `throw`. Бросать можно практически все, что угодно (можно сказать, все, что можно сконструировать). Где будет выделена память под объект, который будем бросать - не специфицировано.

Для отлавливания исключений используем блоки `try` и `catch`.

```cpp
struct A {};
struct B : A {};

try {
    // бросает B
    f();
    
} catch (B b) {
    // так можно
    
} catch (const B& b) {
    // так можно
    
} catch (B&& b) {
    // ошибка компиляции

} catch (const A&) {
    // так можно

} catch (A a) {
    // так можно, но осторожно

} catch (...) {
    // так можно, ловит все

}
```

Поиск обработчика при появлении исключения в блоке `try` идет последовательно, в объявленном разработчиком порядке, поэтому рекомендуется писать обработчики от частного к общему.

Если обработчик не был найден, исключение покинет блок `try`, и стек продолжит разворачиваться.

Конструкцию `throw` можно писать по-разному:

```cpp
struct A {};
struct B : A {};

void f() {
    throw B();     // B
    throw new B(); // B*
}

try {
    // бросает B
    f();
    
} catch (B b) {
    throw b;
    // new B1 will be copied on catch(B b)
    // new B2 will be copied on throw
    // B2 will be thrown
    // B and B1 will be destroyed
    
} catch (B b) {
    throw;
    // new B1 will be copied on catch(B b)
    // B1 will be thrown
    // B will be destroyed
    
} catch (const B& b) {
    throw b;
    // new B1 will be copied on throw
    // B1 will be thrown
    // B will be destroyed
    
} catch (const B& b) {
    throw;
    // B will be thrown

} catch (const A&) {
    throw;
    // B will be thrown

} catch (A a) {
    throw;
    // B will be thrown (check on gotbolt, i'm not sure)

} catch (...) {
    throw;
    // caught object will be thrown
}

```

## noexcept

При использовании после объявления функции ключевое слово `noexcept` разворачивается в конструкцию `noexcept(true)`, означающую "эта функция никогда не бросает исключения".

Гарантия с нашей стороны позволяет компилятору генерировать более компактный код. Однако, если мы нарушим гарантию, и исключение будет брошено из `noexcept` функции, то мгновенно произойдет вызов `std::terminate` и по умолчанию программа аварийно завершится без запуска каких-либо деструкторов.

```cpp
void f() noexcept {}

void g() noexcept {
    throw "oops";
} // calls std::terminate()
```

В конструкцию `noexcept(expr)` после объявления функции можно писать любое статическое булево выражение. Таким образом гарантия исключений шаблонных функций может, например, зависеть от переданного в них типа:

```cpp
template <typename T>
void foo(T&& t) noexcept(std::is_trivially_destructible_v<T>);
```

В другом случае `noexcept(expr)` служит булевым оператором, проверяющим, является ли переданная в `unevaluated`-контексте функция `noexcept`-функцией.

```cpp
struct A {
    ~A() {
        // non-trivial
    }
};

static_assert(noexcept(foo<A>(std::declval<A>())) == 0);
```

Часто используется в таком виде:

```cpp
template <typename T>
void bar(T&& t) noexcept(noexcept(f<T>(std::declval<T>())));
```

## Исключения в деструкторах

Деструкторы, начиная со стандарта C++11, неявно помечены как `noexcept`, то есть им не разрешается бросать исключения вообще, иначе произойдет мгновенный вызов `std::terminate`.
