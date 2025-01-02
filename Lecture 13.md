# Лекция 13 — 25.02.2022. Указатели на функции/методы. Функторы. Лямбды

# Header

```cpp
#include <exception>
#include <functional>
#include <iostream>
#include <string>
#include <vector>
```

<aside>
💡 Основная цель — научиться передавать кастомный код в другую часть программы, чтобы он там исполнился

</aside>

Функция **ZipMap** принимает на вход два вектора типа `int`, применяет к элементам вектора бинарную операцию и возвращает результирующий вектор типа `int`

# ZipMap

```cpp

std::vector<int> ZipMap(const std::vector<int>& a, const std::vector<int>& b) {
    if (a.size() != b.size()) {
        throw std::invalid_argument("a.size != b.size");
    }

    std::vector<int> result;

    for (size_t i = 0; i < a.size(); ++i) {
        result.push_back(a[i] + b[i]);
    }
    
    return result;
}

int main() {
    std::vector<int> a = {1, 2, 3};
		std::vector<int> b = {10, 20, 30};
		Print(ZipMap(a, b)); // функция Print просто выводит вектор
    return 0;
}
```

Вывод будет следующим:

```cpp
11 22 33
```

Сейчас в функции **ZipMap** мы применяем одну операцию сложения для элементов векторов

Мы хотим написать более универсальный код, который позволит применять произвольную операцию (функцию) к элементам векторов

# Указатель на функции

Напишем функцию Sum, которая суммирует два `int`-a

```cpp
int Sum(int a, int b) {
    return a + b;
}
```

Тогда в `int main()` создадим указатель на эту функцию

```cpp
int main() {
    int (*func)(int, int) = Sum; // Функции неявно переводятся в указатели на фукнции
		
		// Можно еще явно написать: int (*func)(int, int) = &Sum;
		
		// int - тип возвращаемого значения

		//func — имя указателя
		//* — знак указателя на функцию
		//Скобки нужны, чтобы компилятор правильно распарсил строку
		
		// (int, int) - часть сигнатуры фукнции Sum, которая определяет типы параметров
		
		
		

		// применим фунцию Sum через указатель
		std::cout << func(1, 2) << std::endl;
    
    std::vector<int> a = {1, 2, 3};
    std::vector<int> b = {10, 20, 30};
    
    Print(ZipMap(a, b)); // функция Print просто выводит вектор
    return 0;
}
```

Вывод:

```cpp
3
11 22 33
```

## Используем алиас типа

Чтобы упростить запись `int (*func)(int, int)`, будем использовать `using`(алиас типа)

```cpp
using Operation = int(int, int); // тип возращаемого значения и параметры функции
```

Тогда объявление указателя на функцию **`Sum`** будет иметь вид 

```cpp
Operation* func = Sum;

// и можно дальше с ним работать
std::cout << func(1, 2);
```

Чтобы передать в **ZipMap** кастомную функцию, изменим ее сигнатуру (итоговый код)

Также можем написать функцию **Sub** вычитания двух `int`-ов

```cpp
using Operation = int(int, int);

int Sum(int a, int b) {
    return a + b;
}

int Sub(int a, int b) {
    return a - b;
}

std::vector<int> ZipMap(const std::vector<int>& a, const std::vector<int>& b, Operation* operation) {
		if (a.size() != b.size()) {
        throw std::invalid_argument("a.size != b.size");
    }

    std::vector<int> result;

    for (size_t i = 0; i < a.size(); ++i) {
        result.pushback(operation(a[i], b[i]));
    }
    
    return result;
}

int main() {
	  std::vector<int> a = {1, 2, 3};
    std::vector<int> b = {10, 20, 30};
    
    Print(ZipMap(a, b, Sum)); // после Sum скобки не нужны, так как это не вызов функции, передаем как объект
    
		Print(ZipMap(a, b, Sub));
		return 0;
}
```

Вывод:

```cpp
11 22 33
-9 -18 -27
```

## Передача метода класса/структуры в функцию

Пусть теперь есть класс/структура **Foo**

```cpp
struct Foo {
    static int Do(int a, int b) const{
        return a * b;
    }
};
```

Если метод статический, то можем вызвать **ZipMap** и передать метод ****`Foo::Do` ****в качестве значения указателя на функцию, так как статический метод — тоже функция, которая живет внутри namespace по имени класса

```cpp
int main() {
	  std::vector<int> a = {1, 2, 3};
    std::vector<int> b = {10, 20, 30};
    
		Print(ZipMap(a, b, Sum)); 
		Print(ZipMap(a, b, Sub));
    Print(ZipMap(a, b, Foo::Do));
		return 0;
}
```

Вывод:

```cpp
11 22 33
-9 -18 -27
10 40 90
```

Что делать, если не статический метод?

```cpp
struct Foo {
    int Do(int a, int b) const {
        return a * b + x;
    }

    int x = 100;
};
```

Теперь нельзя вызвать **ZipMap** как в прошлый раз без создания объекта типа **Foo**

Создадим объект типа **Foo** и указатель на метод `Foo::Do`

```cpp
Foo foo; // создали объект
int (Foo::*method)(int, int) = &Foo::Do; // обязательно по ссылке!

// синтаксис, похожий, как у указателя на функцию
// method - имя указателя

// чтобы теперь вызвать метод
std::cout << foo.*method(10, 20) << std::endl;
         //то же что и foo.Do(10, 20) как бы разыменовали *method
```

Но все равно придется писать разные указатели для разных типов. Например, если есть два класса/структуры с методами похожими методами, то для каждого нужно писать собственный указатель, так как в сигнатуре указателя мы указываем, откуда берется метод

## std::mem_fn()

Чтобы решить проблему передачи метода, используем функцию `std::mem_fn()`

```cpp
Foo foo;
auto method = std::mem_fn(&Foo::Do); 
// создает обертку вокруг указателя на метод Foo:Do с такой же сигнатурой
// method - объект, похожий на функцию, с тремя параметрами: объект типа Foo и двумя параметрами метода Foo::Do

// вызвать метод
std::cout << method(foo, 10, 20) << std::endl; // передача foo в качестве первого аргумента чем-то похоже на то, как в Python передаем self
```

И все равно проблема осталась

## std::bind()

Используем `using namespace std::placeholders;`

```cpp
 using namespace std::placeholders;

int main() {

	Foo foo;
	auto method = std::mem_fn(&Foo::Do); 
	std::cout << method(foo, 10, 20) << std::endl;

	auto method2 = std::bind(&Foo::Do, &foo, _1, _2);
	// создает объект, который хранит указатель на Foo:Do, указатель на foo
	// _1, _2 - глобальные переменные, в которые вставляются параметры метода Foo::Do - placeholder-ы
	// то есть _1 = первый параметр Foo::Do, _2 = второй параметр Foo::Do (копируются по значению)

	// foo можно передать и по значению, но тогда он скопируется

	// вызов method2
	std::cout << method2(100, 200) << std::endl;
	return 0;

}
```

Вывод:

```cpp
2100
```

Поменяем местами placeholder-ы 

Так как операция в `Foo::Do` симметричная, то результат не изменится. Поменяем ее

```cpp
struct Foo {
    int Do(int a, int b) const {
        return a - b + x;
    }

    int x = 100;
};
```

```cpp
...
auto method2 = std::bind(&Foo::Do, foo, _2, _1);
std::cout << method2(100, 200) << std::endl;
// 100 -> второй параметр Foo::Do, 200 -> первый параметр Foo::Do
```

Вывод:

```cpp
2100
```

Уберем один placeholder

```cpp
...
auto method2 = std::bind(&Foo::Do, foo, 1000, _1); // теперь это объект с одним параметром
std::cout << method2(10000) << std::endl;
// 1000 -> первый параметр Foo::Do, 10.000 -> второй параметр Foo::Do
```

Вывод:

```cpp
-8900
```

Или 

```cpp
...
auto method2 = std::bind(&Foo::Do, foo, _1, 1000); // теперь это объект с одним параметром
std::cout << method2(10000) << std::endl;
// 10.000 -> первый параметр Foo::Do, 1000 -> второй параметр Foo::Do
```

Вывод:

```cpp
9100
```

Штука довольно хрупкая, так как нужно помнить, в каком порядке указали placeholder-ы, перечислить их по порядку

### Функторы. Объект, похожий на функцию

`std::mem_fn()` и `std::bind()` возвращают объекты, похожие на функцию. Что это такое?

На самом деле, это специальный класс с перегруженным оператором `**()`** с сигнатурой как у метода, указатель на который был передан первым аргументом

И поэтому указываем тип `auto` чтобы много не думать, что это за тип 

```cpp
//например, напишем класс Division
class Division {
public:
    int operator()(int a, int b) const {
        return a / b;
    }
};

int main() {

   Division div;
   std::cout << div(100, 25) << std::endl;
   return 0;
}
```

Вывод:

```cpp
4
```

Если уберем какие-то placeholder-ы, то это отразится и на сигнатуре оператора `()`

```cpp
auto method2 = std::bind(&Foo::Do, foo, _1, 1000); <=> int operator(int a) const {
																													return a * 1000 // * -- бинарная операция
																												}
```

## std::function<>

`std::function` лежит в `#include <functional>` . Используют с С++ 11

У него один шаблонный параметр — тип функции. Например, `int(int, int)`

У него есть перегрузки для всего, что похоже на функцию. Можем передать и указатель на функцию, и статический метод, результат работы `std::bind`, `std::mem_fn`, наш класс с перегруженным оператором `**()**`

```cpp
std::function<int(int, int)> op = Sum; // хранит в себе все похожее на функцию с такой сигнатурой
std::function<int(int, int)> op = method2; // method2 - std::bind с двумя placeholders
std::function<Operation> op = div; // div - объект класса Division, Operation - альяc = int(int, int)

// если вызвать пустой std::function, то выбросит исключение bad_function call
```

Теперь как передавать `std::function` в **ZipMap** (можно передавать по значению, так как он небольшой)

```cpp
std::vector<int> ZipMap(const std::vector<int>& a, const std::vector<int>& b, std::function<Operation> operation) {
    if (a.size() != b.size()) {
        throw std::invalid_argument("a.size != b.size");
    }

    std::vector<int> result;

    for (size_t i = 0; i < a.size(); ++i) {
        if (operation) { // проверка на пустоту операции. переопределен оператор bool. true - передали функцию, иначе false
            result.push_back(operation(a[i], b[i]));
        }
    }

    return result;
}

int main() {

	  Print(ZipMap(a, b, Sum)); 
		Print(ZipMap(a, b, Sub));
   return 0;
}

```

Можно передавать по константной ссылке. И даже неконстантные операторы все равно будут работать 

### lambda

Перепишем класс Division на lambda function

```cpp
auto div = [](int a, int b) { return a / b; }; // один и тот же код, что и  класс Division
             // Часть сигнатуры функции. Нет возвращаемого типа, так как выводится по умолчанию из return

// копилятор создаст уникальный тип для каждой lambda (даже если одинаковые)
// это будет класс с оператором (), принимающий два int
std::cout << div(100, 25) << std::end
```

Пусть у нас есть еще параметр `x`

```cpp
// лямбда функция захватывает по умолчанию локальные переменные внутри скобок, в которых она объявлена
int x = 100;
auto div = [x](int a, int b) { return a / b + x; }; 

// технически в классе Division создается по умолчанию const приватное поле типа, как у х, и конструктор

class Division {
public:
		Division(int x){
			this->x = x;
		}
    int operator()(int a, int b) const {
        return a / b;
    }

private:
		const int x;
};

// x скопируется в это приватное поле и будет доступно внутри тела lam
```

```cpp

auto div = [x_ = x](int a, int b) { return a / b + x_; }; // переименовали x на x_. в классе поле будет иметь имя x_

auto div = [&x](int a, int b) { return a / b + x; }; // передача по ссылке. в классе будет поле int& и конструктор Divison(int& _x) : x(_x) {}

auto div = [&foo, x](int a, int b) { return foo.Do(a, b) + x; }; // то же самое, что и std::bind, но более читаемо
// копирование x, передача по ссылке foo

auto div = [&](int a, int b) { return foo.Do(a, b) + x; };
// все локальные пременные, которые используем внутри тела lambda, (foo, x) передаются по ссылке

auto div = [=](int a, int b) mutable { return foo.Do(a, b) + x; };
// foo, x передаются по значению
// mutable создает класс с не константными приватными полями foo, x

auto div = [=, &foo](int a, int b) mutable { return foo.Do(a, b) + x; };
// все передается по значению, foo - по ссылке
```

Пусть хотим переписать структуру `Foo::Do` через lambda function

```cpp
struct Foo {
    int Div(int a, int b) const {
        return a / b;
    }
    int Do(int a, int b) const {
        auto div = [](int a, int b) { return Div(a, b) ;};
				return div(a, b);
    }

    int x = 100;
};

// такой код не сработает, так как нужно передавать this 
// можно передавать явно по ссылке, можно по значению (все равно захватывается по ссылке)

struct Foo {
    int Div(int a, int b) const {
        return a / b;
    }
    int Do(int a, int b) const {
        auto div = [&](int a, int b) { return this->Div(a, b) ;};
				// либо auto div = [&](int a, int b) { return this->Div(a, b) ;};
				// либо auto div = [this](int a, int b) { return this->Div(a, b) ;};
				// следующий способ запрещен с 20 стандарта
				// auto div = [=](int a, int b) { return this->Div(a, b) ;};

				return div(a, b);
    }

    int x = 100;
};
```

Если хотим, чтобы лямбда функция возвращала объект определенного типа, то (например, `double`)

```cpp
auto div = [](int a, int b) ->double { return Div(a, b) ;};
```

Если в лямбда функции параметры будут иметь тип `auto` — generic lambda

```cpp
auto div = [](auto a, auto b) ->double { return Div(a, b) ;};

// этот код равносилен шаблонному оператору () 
class Division {
public:
    template<typename T1, typename T2>
    auto operator()(T1 a, T2 b) {
        return a / b;
    }
};

```

## Шаблонная ZipMap

Шаблонным параметром будет бинарная операция и тип векторов

```cpp
template<typename Op, typename T>
auto ZipMap(const std::vector<T>& a, const std::vector<T>& b) {
    if (a.size() != b.size()) {
        throw std::invalid_argument("a.size != b.size");
    }

    Op op;
    std::vector<decltype(op(a[0], b[0]))> result; 
		// decltype нужно, чтобы определить тип вектора
		// при этом никакой оператор не будет реально вызываться

    for (size_t i = 0; i < a.size(); ++i) {
        result.push_back(op(a[i], b[i]));
    }

    return result;
}

class Division {
public:
    template<typename T1, typename T2>
    auto operator()(T1 a, T2 b) {
        return a / b;
    }
};

// тогда ZipMap можно будет вызвать 
Print(ZipMap<Division> (a, b));

int main() {
		std::vector<int> a = {1, 2, 3};
    std::vector<int> b = {10, 20, 30};
   // тогда ZipMap можно будет вызвать 
   Print(ZipMap<Division> (a, b));
   return 0;
}
```
