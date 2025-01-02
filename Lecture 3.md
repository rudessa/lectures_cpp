# Лекция 3. Функции, константность, ссылки и указатели (18.01.2022)

## Функции

### Общее понятие функции

Пусть у нас есть код, который выглядит не очень красиво.

```cpp
#include <iostream>

int main() {
    std::cout << "Hello, " << 1 << std::endl;
    std::cout << "Hello, " << 2 << std::endl;
    std::cout << "Hello, " << 3 << std::endl;
    return 0;
}
```

Его можно улучшить с помощью функции.

```cpp
#include <iostream>

void PrintHello(int value) {
    std::cout << "Hello, " << value << std::endl;
}

int main() {
    PrintHello(1);
    PrintHello(2);
    PrintHello(3);
    return 0;
}
```

1. Сначала пишется тип, который функция будет возвращать (`void` — это значит, что функция возвращать ничего не будет).
2. Потом пишется имя функции (`PrintHello`).
3. Далее в круглых скобочках пишутся аргументы функции (`int value`).
4. После этого идёт тело функции, которое пишется в фигурных скобочках.

ℹ️ Аргументы функции отличаются от параметров функции тем, что аргументы пишутся при создании функции, а параметры — при вызове функции.
То есть `int value` — аргумент функции, а `1`, `2`, или `3` — параметры функции.

### Функции с возвращаемым значением

Теперь напишем функцию, которая будет иметь возвращаемое значение.

Например, реализуем алгоритм подсчёта факториала:

```cpp
#include <iostream>

unsigned int CalcFactorial(unsigned int value) {
    if (value == 0) {  // Можно писать так: 0 == value. Это позволит избежать проблем с тем, что можно перепутать == и =
        return 1;
    }
    return CalcFactorial(value - 1) * value;
}

int main() {
    std::cout << CalcFactorial(5);  // 120
    return 0;
}
```

💡 В жизни факториал так не считают. Используют формулу Стирлинга.


### Функции с одинаковыми именами, но разной сигнатурой

Вернёмся к функции `PrintHello`. Сделаем вторую функцию `PrinrHello`, только теперь она будет принимать не `int`, а `std::string`.

```cpp
#include <iostream>

void PrintHello(int value) {
    std::cout << "Hello, " << value << std::endl;

}

void PrintHello(std::string value) {
    std::cout << "Hello, " << value << std::endl;

}

int main() {
    PrintHello(1);  // Hello, 1
    PrintHello(2); // Hello, 2
    PrintHello(3);  // Hello, 3
    PrintHello("world!"); // Hello, world!
    return 0;
}
```

Это работает, потому что в C++ функция описывается не только своим именем, но и сигнатурой (в сигнатуру функции входят: имя, типы её параметров, тип возвращаемого значения).

### Forward declaration

Теперь посмотрим на пример forward declaration.

Сначала мы описываем сигнатуру функции, а вместо тела функции ставим `;`.

```cpp
#include <iostream>

void PrintHello(int value);

unsigned int CalcFactorial(unsigned int value) {
    PrintHello(value);
    if (value == 0) {
        return 1;
    }
    return CalcFactorial(value - 1) * value;
}

void PrintHello(int value) {
    std::cout << "Hello, " << value << std::endl;
}

int main() {
    CalcFactorial(3);
    return 0;
}
```

### Разница между объявлением и определением

Рассмотрим разницу между двумя понятиями: объявление и определение.

- Объявление — это когда мы просто говорим компилятору, что такая сущность будет существовать (например, строчка `void PrintHello(int value);` из предыдущей программы называется объявлением функции `PrintHello`).
- Определение — это когда мы не только говорим, что у нас будет такая сущность, но и ещё и описываем её (например,
`void PrintHello(int value) {
    std::cout << "Hello, " << value << std::endl;
}`
называется определением функции `PrintHello`)

ℹ️ Чаще всего в файлах `.h` описаны **объявления** функций, классов, констант.
А в `.cpp` файлах описаны уже **определения**.


### Принципы декомпозиции кода на функции

Надо следовать двум принципам:

1. **Принцип единственности ответственности** — функция должна делать только одну вещь и делать её хорошо.
2. **Вызовы других функций внутри функции должны находиться на одном и том же уровне абстракции** — то есть низкоуровневый код не должен смешиваться с высокоуровневым.

## Константность

Константные переменные — это такие переменные, которые нельзя изменять.

При изменении такой переменной возникнет ошибка.

Рассмотрим код, который содержит константы.

```cpp
#include <iostream>

void PrintResult(int value);

unsigned int CalcFactorial(unsigned int value) {
    PrintResult(value);
    if (value == 0) {
        return 1;
    }
    return CalcFactorial(value - 1) * value;
}

int ReadInputValue() {
    int value = 0;
    std::cin >> value;
    return value;
}

void PrintResult(int value) {
    const std::string result_message_prefix = "Hello, ";
    std::cout << result_message_prefix << value << std::endl;
}

int main() {
    const int input_value = ReadInputValue();
    int factorial = CalcFactorial(input_value);
    PrintResult(factorial);
    return 0;
}
```

## Ссылки

### Передача аргументов в функцию

Можно воспринимать параметры функции, как локальные переменные, которые объявлены в начале функции и проинициализированы тем, что мы туда передали.


ℹ️ Если мы просто передаём (без модификаторов) какой-то аргумент в функцию, то он будет передаваться в функцию по значению, то есть он скопируется.
То же самое происходит, когда мы возвращаем что-то в `return` — значение просто скопируется наружу.

### Общее понятие ссылки

Ссылка — это, по сути, другое имя для какого-то объекта (говоря другими словами, ссылка — константный указатель).

**Ссылка ссылается на ту же область памяти, что и переменная.**

```cpp
#include <iostream>

int main() {
    int value = 10;
    int& value_ref = value;
    return 0;
}
```

1. Сначала пишем тип, на который должна ссылаться ссылка.
2. Потом используем `&` (амперсанд — это модификатор типа, который превращает тип в ссылку)
3. Далее указываем имя ссылки.
4. Пишем знак присвоить.
5. Потом инициализируем ссылку тем, на что она должна ссылаться.

⚠️ Есть два вида записи:
`int &value_ref = value;
int& value_ref = value;`
В задачах данного курса принят второй вариант записи, чтобы подчеркнуть, что `int` и `int&` — это разные типы.

### Изменение значения переменной и ссылки на эту переменную

Так как ссылка — это просто другое имя какого либо объекта, то при изменении этого объекта меняется и ссылка, а при изменении ссылки — меняется объект.

```cpp
#include <iostream>

int main() {
    int value = 10;
    int *value_ptr = &value;

    std::cout << value << std::endl;  // 10
    std::cout << *value_ptr << std::endl;  // 10

    *value_ptr = 20;

    std::cout << value << std::endl;  // 20
    std::cout << *value_ptr << std::endl;  // 20

    value = 30;

    std::cout << value << std::endl;  // 30
    std::cout << *value_ptr << std::endl;  // 30
    return 0;
}
```

### Константные ссылки

Если ссылка константная, то можем её использовать только для чтение, но не для записи.

```cpp
#include <iostream>

int main() {
    int value = 10;
    const int &value_ref = value;
    return 0;
}
```

### Передача по ссылке в функцию

```cpp
#include <iostream>

void PrintResult(const std::string &value) {
    const std::string result_message_prefix = "Hello, ";
    std::cout << result_message_prefix << value << std::endl;
}

int main() {
    PrintResult("World!");  // Hello, World!
    return 0;
}
```

ℹ️ Если мы передаём параметр по ссылке, то копирования уже не происходит.

### Возврат нескольких значений через ссылки

```cpp
#include <iostream>

bool CalcFactorial(int value, int &result) {
    if (value < 0) {
        return false;
    }

    if (value == 0) {
        result = 1;
        return true;
    }
    CalcFactorial(value - 1, result);

    result = result * value;
    return true;
}

int main() {
    int factorial = 0;
    if (CalcFactorial(5, factorial)) {
        std::cout << "OK: " << factorial << std::endl;
    } else {
        std::cout << "Error!" << std::endl;
    }
    return 0;
}
```

Функция `CalcFactorial` принимает в качестве второго параметра неконстантную ссылку на на `factorial`.
Внутри функции `CalcFactorial` мы считаем новое значение.
В `result` присваиваем то, что у нас получилось.

### Продление времени жизни временных объектов с помощью константных ссылок

```cpp
#include <iostream>

void PrintResult(int value) {
    const std::string result_message_prefix = "Hello, ";
    std::cout << result_message_prefix << value << std::endl;
}

int main() {
	// Создали временный объект. Время жизни такого объекта — до конца выражения, в котором этот объект используется. После этого объект удалится:
    10 + 20;
	// Если же мы сделаем константную ссылку на этот объект, то время его жизни будет продлено до конца текущего scope:
    const int &ref = 10 + 20;
    return 0;
}
```

### Ошибка при возврате по ссылке через return

Напишем **неправильный** код, который будет возвращать ссылку на переменную.

```cpp
#include <iostream>

int& ReadInputValue() {
    int value = 0;
    std::cin >> value;
    return value;
}

int main() {
    int& v = ReadInputValue();
    std::cout << v;
    return 0;
}

```

Он не будет работать, так как при `return` мы будем возвращать ссылку на переменную `value`. А переменная `value` локальная, то есть определена только в функции `ReadInputValue` $\Longrightarrow$ undefined behavior.

## Указатели

### Общее понятие указателя

Указатель — это переменная, которая хранит в себе адрес какой-то ячейки памяти.

```cpp
#include <iostream>

int main() {
    int value = 10;
    int* value_ptr = &value;
    return 0;
}
```

1. Сначала пишем тип, на который должен указывать указатель.
2. Потом используем `*` (звёздочка — это модификатор типа, который превращает тип в указатель).
3. Далее указываем имя указателя.
4. Пишем знак присвоить.
5. Затем пишем знак `&` — оператор взятия адреса.
**Это не ссылка! Тут этот символ означает совершенно другое!**
Строчка `value_ptr = &value` означает, что в `value_ptr` будет лежать адрес ячейки памяти `value`.
6. Потом инициализируем указатель тем, на что он должен указывать.
Можно инициализировать указатель дефолтным значением `nullptr` — указатель в никуда. Математически он равен 0.

### Получение значения по указателю

```cpp
#include <iostream>

int main() {
    int value = 10;
    int *value_ptr = &value;
    std::cout << value_ptr << std::endl;  // 0x7ffd780be1ac — адрес ячейки памяти
    std::cout << *value_ptr << std::endl;  // 10 — значение value
    return 0;
}
```

Для получения значения по адресу нужно использовать `*` — оператор разыменования.

### Изменение значения переменной и указателя на эту переменную

Это работает аналогично ссылкам.
Чтобы изменить значение указателя, надо обратиться к нему через `*`.

```cpp
#include <iostream>

int main() {
    int value = 10;
    int *value_ptr = &value;

    std::cout << value << std::endl;  // 10
    std::cout << *value_ptr << std::endl;  // 10

    *value_ptr = 20;

    std::cout << value << std::endl;  // 20
    std::cout << *value_ptr << std::endl;  // 20

    value = 30;

    std::cout << value << std::endl;  // 30
    std::cout << *value_ptr << std::endl;  // 30
    return 0;
}
```

### Передача по указателю между переменными

```cpp
#include <iostream>

int main() {
    int value = 10;
    int value2 = 11;
    int *value_ptr = &value;
    int *value2_ptr = &value2;

    value_ptr = value2_ptr;

    std::cout << *value_ptr << std::endl;  // 11
    std::cout << *value2_ptr << std::endl;  // 11
    return 0;
}
```

То есть указатель может указывать на разные объект в течение времени своей жизни.

### Константность указателей

*Мнемоническое правило: читаем справа налево, так можно запомнить.*

- Константный указатель на `int`.
То есть такому указателю нельзя присвоить другой адрес.
Это почти то же самое, что и ссылка.
    
    ```cpp
    int const *value_ptr = &value;
    ```
    
- Указатель на `const int` .
То есть такой указатель указывает на `const int`.
Мы можем поменять сам указатель, но мы не можем через него поменять значение, которое лежит по этому указателю.
    
    ```cpp
    const int *value_ptr = &value;
    ```
    
- Константный указатель на `const int`.
Ни его самого, ни его значение, хранящееся по этому указателю, менять нельзя.
Можно только читать данные.
    
    ```cpp
    const int const *value_ptr = &value;
    ```
    

### Передача по указателю в функцию

```cpp
#include <iostream>

void PrintHello(const std::string *str) {
    std::cout << *str << std::endl;
}

int main() {
    std::string str = "Hello";
    PrintHello(&str);
    return 0;
}
```

### Арифметика указателей

```cpp
#include <iostream>

int main() {
    int value = 10;
    int value2 = 11;
    int *value_ptr = &value;
    int *value2_ptr = &value2;

    value_ptr += 1;
    value_ptr - value2_ptr;
    return 0;
}
```

- Сложение — добавление числа `n` к указателю — сдвиг памяти на `n * sizeof(тип_на_который_указывает_указатель)`.
- Вычитание указателей — возвращает количество элементов между двумя указателями (возвращает тип `size_t`).
