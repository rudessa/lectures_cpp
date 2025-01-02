# Лекция 11. Copy elision, move семантика — 18.02.2022

<aside>
💡 Основная идея - способы избежать копирования
</aside>

Попробуем написать свой собственный класс **String**

```cpp
#include <iostream>

class String {
public:
		String() {

				// Логгирование для того, чтобы было понятно происходящее

				std::cout << "String()" << std::endl;

		}

		String(const char* str) {

				// Логгирование для того, чтобы было понятно происходящее

				std::cout << "String(const char* str)" << std::endl;

				size_ == std::strlen(str); // функция std::strlen - бежит по строке, находит терминатор C-строки и на основе этого выводит длинну
				data_ = new char[size_];
				std::memcpy(data_, str, size_);

		}

		String(const String& other) {

				// Логгирование для того, чтобы было понятно происходящее
				std::cout << "String(const String& other)" << std::endl;

				data_ = new char[other.size_];
				size_ = other.size_;
				std::memcpy(data_, other.data_, size_); // std::memcpy - стандартная функция для копирования. Сигнатура (куда, откуда, размер)
		}

		~String() {

				// Логгирование для того, чтобы было понятно происходящее

				std::cout << "~String()" << std::endl;

				delete[] data_;
		}

		// Для удобства дебага определим оператор вывода в поток
		friend std::ostream& operator << (std::ostream& stream, const String& str) {

				// Нужно указать размер, потому что в С-строках в конце специальный символ ( терминатор ) строки - \0. В нашем массиве **data_** такого символа в конце строки нет
				// Поэтому если просто будем печатать **char*** , то не встретив такой символ в конце, вывод продолжится до тех пор пока не встретится ячейка памяти, в которой будет записан этот терминар.
				// Очевидно, что вывод может вообще не закончиться ( в пределах области памяти процесса ) или если закончится, то всё-равно будет выведен мусор

				return stream << std::string(str.data_, str.size_) << std::endl; 
		}

private:
		char* data_ = null;
		size_t size_ = 0;
};

int main() {
		String str("Hello");

		std::cout << str << std::endl;

		return 0;
}
```

Вывод будет :

```cpp
String(const char* str)
Hello

~String()
```

 Добавим же теперь функцию `MakeString(cosnt char* str)` и изменим `main()`

```cpp
String MakeString(const char* str) {
		return String(str); // RVO
}

int main() {

		std::cout << MakeString("Hello") << std::endl;

		return 0;
}
```

Ожидаемый вывод

```cpp
String(const char* str)
String(const String& other) // При возврате значения из функции MakeString мы копируем результат
~String() // Деструктор строки, созданной внутри MakeString

Hello

~String() // Деструктор строки, которую вернула функция MakeString
```

Но на самом деле ничего не изменится

# Return value optimization

Так происходит, потому что существует механизм **copy elision**. В нём есть **return value optimization (RVO)**. Если в выражении **return** создаётся временный объект того же типа, что и возвращается функцией ( за исключением модификаторов константности и волатильности, при их наличии механизм RVO также применяется ), то гарантируется, что компилятор избежит лишнее копирование. Это произойдёт за счёт того, что return-объект сразу создастся в стек-фрейме вызывающей функции, а не внутренней. В нашем случае объект создастся в стек-фрейме `main()` , а не в стек-фрейме `MakeString()`.

Если сделать объект не временным, а вполне себе настоящим

```cpp
String MakeString(const char* str) {
		String string(str);
		return string; // NRVO
}
```

То вывод также не изменится, потому что теперь применится механизм **named RVO** (NRVO). Гарантий 100% применения NRVO к подобным конструкциям нет, но в данном случае, например, он сработал.

<aside>
💡 Большинство современных компиляторов умеют в NRVO. То, насколько агрессивно будет применяться NRVO или другие подобные оптимизации компилятором, зависит также от настроек компиляции. Так при дебаге NRVO может не применяться, а в релизе применяться

</aside>

Аналогичным образом сработает и код

```cpp
void PrintString(String string) {
		std::cout << string << std::endl;
}

int main() {

		PrintString(String("Hello"));
		
		return 0;

}
```

# Move семантика

Если код будет таким :

```cpp
int main() {

		String str("Hello");
		PrintString(str);
		
		return 0;

}
```

Тут уже получим вариант без RVO

```cpp
String(const char* str)
String(const String& other) // Передаём в PrintString копию по значению
Hello

~String() // Деструктор строки, созданной внутри PrintString
~String() // Деструктор str из main()
```

Начиная с С++11 появилась **move семантика**. Её основная идея заключается в следующем : пусть нам нужно скопировать объект, при этом копируемый объект нам больше не нужен. Тогда вместо **deep-copy** всех внутренних полей ( как в конструкторе копирования ) мы можем просто взять и “украсть” внутренние поля копируемого объекта. Очевидно, что обычно это дёшево. Например в нашем случае достаточно скопировать указатели.

## Move конструктор

Такая семантика может быть реализована с помощью **move конструктора**. Такой конструктор принимает специальную **RValue** ссылку ( обозначается как `T&& other` )

```cpp
// move - конструктор. Принимает RValue ссылку
String(String&& other) {

		std::cout << "String(String&& other)" << std::endl;

		data_ = other.data_;
		size = other.size_;

		// Но чтобы при вызове деструктора other не потерялись данные, нужно предпринять какие-то шаги. Например :
		other.data_ = nullptr;
		other.size_ = 0; // особо ни на что не влияет, но зато красиво :)
}
```

Для демонстрации преимущества, напишем оператор +

```cpp
String operator+(const String& other) {

		String result;

		result.size = size_ + other.size_;
		result.data_ = new char[result.size_];

		std::memcpy(result.data_, data_, size_);
		std::memcpy(result.data_ + size_, other.data_, other.size_);
		
		return result;
}
```

Теперь при исполнении `main()`

```cpp
int main() {

		String str1("Hello");
		String str2(" world");

		PrintString(str1 + str2);
		
		return 0;

}
```

получим

```cpp
String(const char* str) // Создание str1
String(const char* str) // Создание str2
String() // Объект, созданный внутри оператора +
Hello world // Вывод строки. Заметим, что в PrintString также применился RVO

~String() // Деструктор строки, полученной при сложении
~String() // Деструктор str2
~String() // Деструктор str1
```

Заметим, что тут нет move-конструктора, так как компилятор слишком умный и применил RVO. Чтобы продемонстировать его вызов, воспользуемся функцией `std::move`

```cpp
int main() {

		String str1("Hello");
		String str2(" world");

		// валидно было бы написать и
		// String str = std::move(str1);
		String str(std::move(str1));
		
		return 0;

}
```

и получим вывод

```cpp
String(const char* str) // Создание str1
String(const char* str) // Создание str2
String str(String&& other) // Создание str

~String() // Деструктор str
~String() // Деструктор str2
~String() // Деструктор str1
```

<aside>
💡 Функция `std::move` на самом деле не выполняет никаких операций с памятью. Её можно воспринимать как некоторый **cast** к **RValue** ссылке (`T&& other`) для того, чтобы явно вызвать **move** конструктор. Очевидно, что вызвав его мы как бы заявляем о том, что дальше этот объект не будет использоваться.

</aside>

## Move assign

Кроме того, **move** семантика может быть реализована с помощью присваивания, принимающего **RValue** ссылку. Сравним обычное присваивание и **RValue** присваивани

```cpp
// Обычный оператор присваивания
String& operator=(const String& other) {
		std::cout << "operator=(const String& other)" << std::endl;

		if (this == &other) {
				return *this;
		}

		// Плохой вариант
		// Если не получится выделить память, тут может выброситься **bad_alloc** и тогда присваивание произойдёт нетранзакционно ( потому что старые данные уже удалены )
		// delete[] data_;
		// data_ = new char[other.size_];

		// Хороший вариант
		char* tmp = new char[other.size_]

		delete[] data_;
		data_ = tmp;
****
		size_ = other.size_;
		std::memcpy(data_, other.data_, size_); // std::memcpy - стандартная функция для копирования. Сигнатура (куда, откуда, размер)

		return *this;
}
```

Код

```cpp
int main() {

		String str1("Hello");
		String str2(" world");

		str2 = str1

		std::cout << str2 << std::endl;
		
		return 0;

}
```

выведет

```cpp
String(const char* str) // Создание str1
String(const char* str) // Создание str2
operator=(const String& other)
Hello

~String() // Деструктор str2
~String() // Деструктор str1
```

<aside>
💡 Порядок вызова конструкторов и деструкторов можно отследить добавив к выводу ссылку, например :

```cpp
		~String() {

				std::cout << "~String()" << this << std::endl;

				delete[] data_;
		}
```

</aside>

**RValue** присваивание

```cpp
String& operator=(String&& other) {
		std::cout << "operator=(String&& other)" << std::endl;

		if (this == &other) {
				return *this;
		}

		delete[] data_;

		data_ = other.data_;
		size = other.size_;

		// Но чтобы при вызове деструктора other не потерялись данные, нужно предпринять какие-то шаги. Например :
		other.data_ = nullptr;
		other.size_ = 0; // особо ни на что не влияет, но зато красиво :)

		retrun *this;
}
```

Теперь

```cpp
int main() {

		String str1("Hello");
		String str2(" world");

		str2 = std::move(str1);

		std::cout << str2 << std::endl;
		std::cout << str1 << std::endl;
		
		return 0;

}
```

выведет

```bash
String(const char* str) // Создание str1
String(const char* str) // Создание str2
operator=(String&& other)
Hello

// отметим, что str1 не распечатался ( точнее распечаталось **ничего** ), потому что его данные были удалены

~String() // Деструктор str2
~String() // Деструктор str1
```

## Swаp идиома

Так как полей в **RValue** объекте может быть много, и надо очищать их и в **move**-конструкторе и в **move** операторе присваивания, то можно просто поменять местами поля. Тогда после обмена, при вызове деструктора **RValue** объекта очистятся старые данные

```cpp
String& operator=(String&& other) {
		std::cout << "operator=(String&& other)" << std::endl;

		// Отметим, что сравнивать с самим собой объект теперь не надо, потому что это заложено в функцию swap
		std::swap(data_, other.data_);
		std::swap(size_, other.size_);

		retrun *this;
}
```

<aside>
💡 Важный момент, что время вызова деструктора **RValue** объекта не особо определёно. Поэтому память будет очищена не сразу

</aside>

## Какие объекты могуть биндиться к RValue ?

<aside>
💡 Строго говоря нужно посмотреть статью про ссылки на cppreference.com

</aside>

Но вообще биндятся

1. Объекты, которые перестанут существовать после вызова строчки
2. Объекты, которые явно закастили с помощью `std::move`

## Хак ( паттерн )

Если же уберём **move** присваивание, то код ниже будет работать аналогично варианту с **move** присваиванием

```cpp
String& operator=(String other) {
		std::cout << "operator=(String other)" << std::endl;

		// Отметим, что сравнивать с самим собой объект теперь не надо, потому что это заложено в функцию swap
		std::swap(data_, other.data_);
		std::swap(size_, other.size_);

		retrun *this;
}
```

Это работает из-за того, что объект, к которому присваиваем - временный, и применяется идиома **copy-and-swap, к**огда сначала с помощью **move**-конструктора создаётся временный объект ( в котором поля - поля временного объекта ), а затем с помощью **swap** происходит обмен.

## Автоматическая генерация move

Достаточно часто компилятор сам генерирует **move**-конструктор. Чтобы это произошло нужно :

1. Не было пользовательских **copy**-конструкторов ( считается, что раз мы хотим кастомный **copy**, значит что-то тут не чисто и **move**-конструктор дефолтный мы не захотим )
2. Пользовательский оператор присваивания
3. Пользовательское **move**-присваивание
4. Пользовательский деструктор
    
    <aside>
    💡 Потому что **move** не всегда означает то, что описано выше. Например для примитивных типов **move** может сослаться на копирование
    
    </aside>
    
5. Если в классе есть поля, которые не могут быть **move**, то и весь класс не может быть **move** компилятором по умолчанию

<aside>
💡 Аналогично и для **move**-присваивания, но вместо пункта 3 пользовательский **move**-конструктор

</aside>

# Про `std::unique_ptr`

Код ниже не сработает 

```cpp
std::unique_ptr<String> ptr = std::make_unique<String>();
std::unique_ptr<String> ptr2 = ptr;
```

Потому что в `std::unique_ptr` удалён конструктор копирования. Если бы можно было копировать, то остались бы в ситуации, когда два разных `std::unique_ptr` указывают на одну и ту же область памяти.

Тогда при удалении какого-то из них вызвалось бы освобождение области памяти на которую они ссылаются, и при удалении второго, также произошла попытка освобождения памяти, которая была уже очищена первым `std::unique_ptr` ( или что ещё хуже, попытались бы работать с этой областью памяти )

Но в нём есть **move**-конструктор и **move**-присваивание. Поэтому код ниже сработает

```cpp
std::unique_ptr<String> ptr = std::make_unique<String>();
std::unique_ptr<String> ptr2 = std::move(ptr);
```
