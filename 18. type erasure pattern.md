## std::function

Пусть `void(int,int)` - это тип функции, принимающей два инта и возвращающей ничего, тогда мы можем похранить ее в `std::function` следующим образом.

```cpp
void print(int a, int b) {
    std::cout << a << " " << b << endl;
}

std::function<void(int, int)> func = print;

func(1, 2); // output: 1 2
```

Похранить функции с другой сигнатурой мы тоже, конечно, можем.

С помощью `function` мы так же можем хранить и лямбды (и любой другой функциональный объект).

```cpp
void f(bool flag) {
    std::function<void(int,int)> func;
    if (flag) {
        func = [](int a, int b){}; 
    } else {
        // note: все лямбды имеют разный тип
        func = [](int a, int b){};
    }
}
```

Примечательно, что классу `function` достаточно знать только тип функции и только на этапе объявления объекта.

Этот класс реализует паттерн `type erasure`. Тот же самый паттерн встречается и в других классах STL.

## std::optional

Класс, который хранит опциональное значение (либо шаблонный тип, либо `std::nullopt_t`).

```cpp
std::optional<int> convert(std::function<void(int)> &f) {
    // ....
    if (!fail) {
        return result;
    }
    return {};
}

int main() {
    auto val = convert(...);
    if (val.has_value()) {
        std::cout << "OK";
    } else {
        std::cout << "Fail";
    }
}
```

## std::any

Тип `any` хранит в себе объект любого типа. Так одна и та же переменная типа `any` может сначала хранить `int`, затем `float`, а затем строку.

Требуется каст для обратного преобразования.

```cpp
std::any a = 42;
int v = std::any_cast<int&>(a);

a = std::string("hello");
std::string s = std::any_cast<std::string&>(a);

// a = mytype(); и так далее
```

Если в качестве шаблонного параметра `any_cast` был передан любой тип, отличный от типа текущего хранимого объекта, будет выброшено исключение `bad_any_cast`.

Если экземпляр `any` разрушается деструктором, то он корректно удаляет хранимый объект.

## std::variant

Шаблонный класс, который представляет собой типобезопасный `union`, который помнит, какой тип он хранит. В отличие от `union` , `variant` позволяет хранить не только POD-типы (тривиальные типы или тривиальные классы).

```cpp
std::variant<int, float, char> v;

v = 3.14f;
v = 42;

std::cout << std::get<int>(v);
```

Для получения значений из `variant` используется функция `get`. Она выбросит исключение `bad_variant_access`, если попытаться взять не тот тип.

Говоря про доступ к варианту, нельзя не упомянуть `visit`, принимающий функцию, которая должна уметь принимать любой тип из данного `variant`.

```cpp
std::variant<int, float, char> v;
v = 42;

std::visit([](auto& arg) {
    using Type = std::decay_t<decltype(arg)>;

    if constexpr (std::is_same_v<Type, int>) {
        std::cout << "int: " << arg;

    } else if constexpr (std::is_same_v<Type, float>) {
        std::cout << "float: " << arg;

    } else if constexpr (std::is_same_v<Type, char>) {
        std::cout << "char: " << arg;
    }
}, v);
```

---

Заимствования:

[cpp-notes/19_lambdas_type_erasure.md at master · lejabque/cpp-notes (github.com)](https://github.com/lejabque/cpp-notes/blob/master/src/19_lambdas_type_erasure.md)
