# Преобразования/касты указателей и ссылок [00:15]
Напоминание примера:
```
struct Queue { // base class (базовый класс)
private:
    IntList data;
public:
/*
    Queue();
    Queue(const IntArray &init_values);
    void push(int x);
    void pop();
    int front() const;
    bool empty() const;
*/
};
struct SizedQueue : public Queue { // derived class (производный класс)
private:
    size_t len;
public:
/*
    void push(int x) { len++; /*Queue::*/push(x); }
    void pop() { len--; /*Queue::*/pop(x); }
    size_t length() const { return len; }
*/
};
```

## Upcast [00:05]
Есть неявное приведение типа "вверх по иерархии" (upcast) или к базовому классу (base-cast):
```
SizedQueue *psq = ...;
Queue *pq = psq;
pq->front(); pq->empty(); // ok
pq->push(...); pq->pop(); // oops

SizedQueue sq = ...;
Queue &q = sq;
q.front(); q.empty(); // ok
q.push(...); q.pop(); // oops
```

## Нарезка (slicing) [00:05]
* Из-за ссылок в конструктор копирования (даже по умолчанию) можно передать наследника:
  ```
  Queue(const Queue &other) { ... }
  ```
* Тогда во фразе `SizedQueue a; Queue b = a` в `b` окажется копия `a`, которая забыла
  свою длину.
* Вроде ничего страшного, но нарезка (slicing) происходит молча:
  ```
  void foo(slow_queue q);
  fast_queue q;
  foo(q); // Ой, нарезка.
  ```

## Downcast [00:05]
Можно и в обратную сторону приводить:
```
Queue *pq = ...;
if (we_are_sure_it_is_sized_queue) {
    SizedQueue *psq = (Queue*)pq;
}
```

Но есть проблема, C-style cast очень мощный:
```
struct Point { int x, y; }
Point *pp = (Point*)pq;  // WTF, UB
```

Поэтому лучше в Си использовать `static_cast<Point*>(pq)`.
Почти везде, где бы в Си был `(Point*)`.
Есть ещё другие `_cast`, но про них потом.

# Простые виртуальные функции [00:20]
## Синтаксис [00:10]
```
struct Queue {
    virtual void push(int x) { ... }
    virtual int pop() { ... }
    int front() const { ... }
};
struct SizedQueue {
    void push(int x) { ... } // virtual автоматически следует.
};
SizedQueue *psq = ...;
Queue *pq = psq;
pq->push(); // Ok!
```
По смыслу: помечаем в `Queue` метод как "можно будет в наследнике
перезаписать, и надо всегда вызывать правильную версию".

Почему "виртуальное" - не знаю.

С C++11 можно ещё `void push(int x) override`, тогда проверит, что в базовом
классе есть виртуальный метод с такими параметрами.

## Статическое и динамическое связывание [00:10]
* Статическое связывание - имя заменяется на код на этапе компиляции.
  Так обычно происходит в C++.
* Динамическое связывание - только на этапе выполнения программы.
  Как раз виртуальные функции.
  А в Python/Java так всегда и везде, там все функции "виртуальные".

Статическое реализуется на этапе компиляции.
А вот для динамического надо что-то хранить в процессе выполнения.

Компилятор для каждого типа заводит таблицу с функциями: "имя - адрес".
Дальше в начало каждого __объекта__ ставит указатель `vptr` на эту таблицу.
Теперь можно смотреть по этому указателю и получать адрес нужной функции.

Это сильно медленнее (два лишних обращения к памяти) и нельзя
встраивать код, зато работает.

Вопрос на засыпку: почему, если пометить как `virtual`, уже в наследнике,
работать не будет?

# Параметрический полиморфизм [00:35]
## Задача [00:10]
Новый пример: пишем графический редактор.
В нём есть список фигуры, у каждой фигуры есть операции.
Фигуры бывают разные.

Хочется, чтобы код вроде "напечатай все фигуры" или "найди все фигуры,
в которые мы кликнули" работал независимо от классов фигур.
Много операций у фигур общие, создадим класс, описывающий
только общий интерфейс:
```
struct Figure {
    Figure(int id, int center_x, int center_y);
    virtual void print() {}
    virtual bool is_inside(int x, int y) const {}
    virtual void zoom(int factor) {}
    virtual void move(int new_x, int new_y) {}
protected:
    int center_x, center_y;
};
```
А ещё хочется все фигуры сложить в один массив.

```
struct Circle : Figure {
    Circle(...) { ... }
    void print() override { ... }
private:
    int radius;
};
```

## Массив объектов [00:05]
Попробуем `Figure objects[100]; objects[10] = Circle(...);`

Происходит слайсинг, мы теряем всю информацию у кружочке.

Оно и логично: фигура может занимать очень много места,
причём все разное, а элемент массива должен занимать фиксированное.

Единственный выход - массив указателей, а элементы хранить отдельно.
Либо свой массив для каждого типа элементов, либо в куче.

## Виртуальный деструктор [00:10]
```
Figure* objects[2] = {
    new Circle(10, 10, 3),
    new Rectangle(...)
};
delete objects[1];
delete objects[0];
```

Будет упс: `delete` не знает точный тип `Figure` и вызовет не тот деструктор.
Это UB.

Поэтому деструктор всегда делаем виртуальным, если такая штука предполагается
(хорошее правило - если есть виртуальная функция, то деструктор тоже виртуальный):
`virtual ~Circle() {}`
Остальные деструкторы вниз тоже автоматически виртуальны.

## Чисто виртуальные функции, абстрактные классы [00:10]
Здесь у `Figure` нет своей логики.
Поэтому надо как-то гарантировать, что:

1. Наследник перезапишет все методы и они не останутся пустыми.
2. Нельзя создать экземпляр `Figure f`, он ничего делать не умеет.

Пишем так: `virtual void print() = 0;`
Это прям `0`, исторический синтаксис, даже не `nullptr`.

Теперь `Figure` стал __абстрактным классом__ .
`Figure f;` больше написать нельзя, `new Figure;` тоже,
а вот `new Circle` можно, потому что в нём все методы реализованы.

К сожалению, нельзя сказать "этот класс точно не абстрактный".

Получили интерфейсы, как были в Python.

# Вывод [00:01]
Обычное наследование — это инструмент, иногда полезный (особенно со следующей лекции),
но просто его использовать для экономии места не стоит, можно огрести с нарезкой.

А вот виртуальные функции позволяют делать динамическое связывание и параметрический
полиморфизм, что полезно.