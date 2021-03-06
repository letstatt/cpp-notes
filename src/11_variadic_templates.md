## Parameter pack

В следующую функцию можно передать ноль или более различных аргументов любого типа.

```cpp
template<typename... Args>
void f(Args&&... args) {
    // ...
}
```

Как теперь можно работать с аргументом `args`?

Можно развернуть его в список аргументов другой функции, как если бы мы перечислили аргументы через запятую.

```cpp
template<typename... Args>
void f(Args&&... args) {
    std::cout << sum(args...);
}
```

Теперь если мы вызовем `f(1, 1.f, '1')`, то в консоль выведется результат `sum(1, 1.f, '1')`.

Parameter pack можно использовать и для более хитрых вещей, например, в следующем примере:

```cpp
template<typename... Args>
void f(Args&&... args) {
    std::cout << sum((args * 2)...);
}
```

Конструкция `((args * 2)...)` развернется в `(a1 * 2, a2 * 2, a3 * 2)`.

Раскрывать parameter pack можно с помощью, например, рекурсии:

```cpp
int sum(int t) {
    return t;
}

template <typename... Tail>
int sum(int t, Tail... tail) {
    return t + sum(tail...);
}

template<typename... Args>
void f(Args... args) {
    std::cout << sum(args...);
}

f(1, 2, 3); // out: 6
```

С помощью variadic templates, `type_traits` и `SFINAE` можно делать умопомрачительные вещи (в том числе в compile-time), но мы опустим этот момент.

## Fold expressions (C++17)

Синтаксис:

1. ( pack_name op ... )
2. ( ... op pack_name )
3. ( pack_name op ... op init )
4. (init op ... op pack )

где pack_name - имя parameter pack, op - оператор, init - начальное значение.

Во что эти конструкции разворачиваются:

1. ( E op ... ) -> ( E1 op (... op ( En-1 op En )))
2. ( ... op E) -> ((( E1 op E2 ) op ...) op En )
3. ( E op ... op I ) -> ( E1 op (... op ( En op I )))
4. ( I op ... op E ) -> ((( I op E1 ) op ...) op En )

### Битовое И переменного числа аргументов

```cpp
template <typename... Args>
bool all(Args... args) {
    return (... && args);
}

static_assert(all(true, true, false) == false);
static_assert(all() == true); // ?
```

Для раскрытия parameter pack длины 0 некоторые операторы имеют значения по умолчанию:

* Логическое И -> `true`
* Логическое ИЛИ -> `false`
* Оператор запятой `,` -> `void()` (?)

### Вывод в консоль переменного числа аргументов

```cpp
template<typename... Args>
void print(Args&&... args) {
    (std::cout << ... << args) << '\n';
}
```
