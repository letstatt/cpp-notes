## std::atomic

Атомарные переменные обеспечивают атомарные взаимодействия с объектом (только я сейчас читаю/пишу в данную переменную, а остальные потоки меня ждут и т.д).

Фишка в том, что на каких-то архитектурах атомарный доступ может быть реализован отдельными инструкциями, а не просто захватом мьютекса или входом в критическую секцию.

Мьютекс - вещь тяжелая, она тянет за собой вызовы к ядру (перепланирование, усыпление потока). Некоторые компиляторы/операционные системы могут соптимизировать блокировку следующим образом: сначала ждать какое-то время в спинлоке, и только затем захватывать мьютекс - при малом ожидании к ядру можно и не обращаться.

Рассмотрим следующий не очень оптимальный код, но жить так можно:

```cpp
// overkill
std::mutex m;
int a = 1;

// ...

m.lock();
a += 100;
m.unlock();
```

Этот же код с использованием атомарных переменных:

```cpp
std::atomic<int> a(1);

// ...

a.fetch_add(100); // равносильно a += 100;
```

Правда, атомарные переменные по стандарту вовсе не обязаны быть `lock-free` (без мьютексов и других блокировок). Проверить, что атомарная переменная является неблокирующей можно с помощью `atomic<T>::is_lock_free()`.

Единственный атомарный тип с гарантированным по стандарту неблокирующим поведением - `atomic_flag`.

```cpp
// constructor leaves it in uninitialized state until C++20
std::atomic_flag f;

// set to false
f.clear();

// set to true and return previous value
f.test_and_set();

// return value
f.test()
```

В языке определены следующие специализации атомиков (некоторые опущены):

```
atomic_bool   - std::atomic<bool>
atomic_char   - std::atomic<char>
atomic_short  - std::atomic<short>
atomic_int    - std::atomic<int>
atomic_long   - std::atomic<long>
atomic_llong  - std::atomic<long long>
atomic_size_t - std::atomic<std::size_t>
```

В отличие от `atomic_flag` у них побольше методов.

```cpp
std::atomic_int a(1337);

// replace value
a.store(445);

// get value
a.load();

// replace value and return previous
a.exchange(42);

// a += 42 and return previous
a.fetch_add(42);

// a += 1 and return previous
a++;

// a += 1 and return new value
++a;

// and so on
```

Другие рассматривать не будем, потому что очень сложно.

На самом деле у тех операций, что мы выписали, есть дополнительный второй параметр, который называется `memory_order`.

## std::memory_order

`memory_order` - это про порядок операций и синхронизацию памяти между потоками.

Внимание: синхронизация процесса выполнения и синхронизация памяти - это, внезапно, разные вещи!

Известно, что компилятор может переупорядочивать наш код, чтобы он работал быстрее, ровно этим же занимается процессор.

Когда мы работаем с многопоточным кодом разбрасываться порядком операций и синхронизацией уже нельзя.

Рассмотрим три типа `memory_order` - `relaxed`, `release/acquire` и `sequential consistency`.

### std::memory_order_relaxed

Самый простой для понимания флаг синхронизации памяти — `relaxed`. Он гарантирует только свойство атомарности операций, при этом не может участвовать в процессе синхронизации данных между потоками.

Свойства:

* Модификация переменной "появится" в другом потоке не сразу
* Поток `thread2` "увидит" значения **одной и той же** переменной в том же порядке, в котором происходили её модификации в потоке `thread1`
* Порядок модификаций разных переменных в потоке `thread1` не сохранится в потоке `thread2`

Можно использовать `relaxed` модификатор в качестве счетчика или в качестве флага остановки.

Пример неправильного использования `relaxed`:

```cpp
std::string data;
std::atomic<bool> ready{ false };

void thread1() {
    data = "very important bytes";
    ready.store(true, std::memory_order_relaxed);
}

void thread2() {
    while (!ready.load(std::memory_order_relaxed));
    std::cout << "data is ready: " << data << "\n"; // potentially memory corruption is here
}
```

Тут нет гарантий, что поток `thread2` увидит изменения `data` ранее, чем изменение флага `ready`, так как синхронизацию памяти флаг `relaxed` не обеспечивает.

### std::memory_order_seq_cst

Флаг синхронизации памяти "единая последовательность" (sequential consistency, `seq_cst`) дает самые строгие свойства:

* Порядок модификаций разных атомарных переменных в потоке `thread1` сохранится в потоке `thread2`
* Все потоки будут видеть один и тот же порядок модификации всех атомарных переменных. Сами модификации могут происходить в разных потоках
* Все модификации памяти (не только модификации над атомиками) в потоке `thread1`, выполняющим `store` на атомарной переменной, будут видны после выполнения `load` этой же переменной в потоке `thread2` (свойство, как у мьютекса)

Таким образом можно представить `seq_cst` операции, как барьеры памяти, в которых состояние памяти синхронизируется между всеми потоками программы.

Этот флаг синхронизации памяти в C++ используется по умолчанию, так как с ним меньше всего проблем с точки зрения корректности выполнения программы, но `seq_cst` является дорогой операцией для процессоров.

### std::memory_order_acquire & std::memory_order_release

Флаг синхронизации памяти `acquire/release` является более тонким способом синхронизировать данные между парой потоков. Два ключевых слова: `memory_order_acquire` и `memory_order_release` работают только в паре над одним атомарным объектом. Рассмотрим их свойства:

* Модификация атомарной переменной с `release` будет видна видна в другом потоке, выполняющем чтение этой же атомарной переменной с `acquire`
* Все модификации памяти в потоке `thread1`, выполняющим запись атомарной переменной с `release`, будут видны после выполнения чтения той же переменной с `acquire` в потоке `thread2` (свойство, как у мьютекса)
* Процессор и компилятор не могут перенести операции записи в память раньше `release` операции в потоке `thread1`, и нельзя перемещать выше операции чтения из памяти, которые были позже `acquire` операции в потоке `thread2`

Используя `release`, мы даем инструкцию, что данные в этом потоке готовы для чтения из другого потока. Используя `acquire`, мы даем инструкцию "подгрузить" все данные, которые подготовил для нас первый поток. Но если мы делаем `release` и `acquire` на разных атомарных переменных, то получим UB вместо синхронизации памяти.

Рассмотрим mutex на основе спинлока:

```cpp
class mutex {
public:
    void lock() {
        bool expected = false;
        while(!_locked.compare_exchange_weak(expected, true, std::memory_order_acquire)) {
            expected = false;
        }
    }
 
    void unlock() {
        _locked.store(false, std::memory_order_release);
    }
 
private:
    std::atomic<bool> _locked;
};
```

Обратите внимание, что мьютекс не только обеспечивает эксклюзивный доступ к блоку кода, который он защищает. **Он также делает доступным те изменения памяти, которые были сделаны до вызова** `unlock()` **в коде, который будет работать после вызова** `lock()`. Это важное свойство. Иногда может сложиться ошибочное мнение, что мьютекс в конкретном месте не нужен.


---

Заимствования:

[std::atomic. Модель памяти C++ в примерах / Хабр (habr.com)](https://habr.com/ru/post/517918/)
