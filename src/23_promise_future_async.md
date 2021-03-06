## Примеры использования promise и future

```cpp
// Create a promise
std::promise<int> promise;

// And get its future
std::future<int> future = promise.get_future();

// You can also get a shared future this way, by the way! (Choose one please)
std::shared_future<int> shared_future = promise.get_future();

// Now suppose we passed promise to a separate thread.
// And in the main thread we call...
int val = future.get(); // This will block!

// Until, that is, we set the future's value via the promise
promise.set_value(10); // In the separate thread

// So now in the main thread, if we try to access val...
std::cout << val << std::endl;

// Output: 10
```

Из заметки про атомики здесь заметна семантика `release`/`acquire`. Так и будет - все изменения, сделанные потоком, записавшим в `promise` значение, будут видны потоку, который прочитал `future`.

```cpp
#include <iostream>
#include <thread>
#include <future>
 
void initiazer(std::promise<int> *promObj) {
    std::cout << "Inside Thread" << std::endl;
    promObj->set_value(35);
}
 
int main() {
    std::promise<int> promiseObj;
    std::future<int> futureObj = promiseObj.get_future();
    std::thread th(initiazer, &promiseObj);
    std::cout << futureObj.get() << std::endl;
    th.join();
    return 0;
}
```

## std::async

Наипростейший способ использовать `async` - это просто передать callback-функцию как аргумент и дать системе сделать все за тебя.

```cpp
auto future = std::async(some_function, arg_1, arg_2);
```

Стоит отметить, что `async` поддерживает параллелизм, но конструктор по умолчанию может не запускать переданные функции в отдельном потоке. Для того, чтобы функция точно выполнилась в отдельном потоке, необходимо явно сказать `async` об этом.

Also, since Linux threads run sequentially by default, it's especially important to force the functions to run in separate threads. We'll see how to do that later.

### Политики запуска std::async

Есть три способа запустить асинхронную задачу:

* `std::launch::async` - гарантирует запуск в отдельном потоке
* `std::launch::deferred` - функция будет вызвана только при вызове `get()`
* `std::launch::async | std::launch::deferred` - поведение по умолчанию, отдать на откуп системе.

Запуск задачи будет выглядеть так:

```cpp
auto future = std::async(std::launch::async, some_function, arg_1, arg_2);
```

Еще примеры:

```cpp
// Pass in function pointer
auto future = std::async(std::launch::async, some_function, arg_1, arg_2);

// Pass in function reference
auto future = std::async(std::launch::async, &some_function, arg_1, arg_2);

// Pass in function object
struct SomeFunctionObject {
    void operator() (int arg_1) {}
};

auto future = std::async(std::launch::async, SomeFunctionObject(), arg_1);

// Lambda function
auto future = std::async(std::launch::async, [](){});
```


---

Заимствования:

[coding-notes/07 C++ - Threading and Concurrency.md at master · methylDragon/coding-notes (github.com)](https://github.com/methylDragon/coding-notes/blob/master/C%2B%2B/07%20C%2B%2B%20-%20Threading%20and%20Concurrency.md)
