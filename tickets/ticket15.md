# Жизнь объектов

## Семантика копирования

В C++ по умолчанию при присваивании объекта / 
передаче объекта в функцию или возврате из функции ожидается, что объект будет скопирован и 
изменения, произошедшие с копией не мутируют сам объект.

При этом существует способ создать ссылку на объект, то есть добавить новое имя для ровно того же объекта в памяти.
Для этого импользуется синтаксис `Type &refer = ....;`, функции также могут принимать аргументы по ссылке. 
При возвращении ссылки из функции нужно быть очень осторожным - локальные переменные умирают при завершении функции, 
возникает dangling reference - ссылка на умерший объект. Обращение по ней - UB. [stackowerflow](https://stackoverflow.com/questions/46011510/what-is-a-dangling-reference)


```c++
void foo(std::vector<int> a) {
    a.push_back(1);
    std::cout << a.size(); 
} 

void bar(std::vector<int>& a) {
    a.push_back(1);
    std::cout << a.size();                                  
} 


int main() {
    std::vector<int> a{0};
    foo(a); // 2
    std::cout << a.size(); // 1 
    bar(a); // 2
    std::cout << a.size(); // 2  
    return 0;
}
```

Пример с dangling reference

```c++
#include <iostream>
#include <vector>

std::vector<int>& foo() {
    std::vector<int> vec{1, 2, 3};
    return vec;
}

int main() {
    std::vector<int> vec = foo();
    std::cout << vec.size() << "\n"; // UB
}
```

## Storage duration

Storage duration - характеристика объектов, опиисание их времени жизни - момента, когда они создаются и умирают.
[cppreference](https://en.cppreference.com/w/cpp/language/storage_duration) - полное описание.
Мы рассматриваем automatic, static и dynamic.

### Automatic storage duration

Наиболее распространённая storage duration - мы создаём объект при проходе через объявление переменной - её значение 
(обычно оно кладётся на стэк, но стандарт это никак не оговаривает). Объект умирает, когда эта переменная 
становится невидимой навсегда (shadowing не считается, т.к. в какой-то момент переменные снова становятся видимыми).
У полей структуры с automatic storage duration - такой же storage duration. 

```c++
int main() {
    std::vector<int> v;                    // (1) - created

    for (int i = 0; i < 10; i++) {
        std::vector<int> v;                // (2) - created (10 times)
        if (i % 2 == 0) {
             break;                        // (2) - deleted
        }
    }                                      // (2) - deleted
    v.push_back(1);                      
    return 0;                              // (1) deleted
}


```

### Static storage duration

Переменные инициализируются в какой-то момент и живут до окончания всей программы. Основной пример -
глобальные переменные. Можно создать и локальные объекты с таким же storage duration - для этого
их нужно пометить `static Type var;`. Они инциализируются только при первом прохождении через строчку с их
объявленем, а затем существуют до конца выполнения программы. При их инициализации доступны вске видимые объекты, 
существующие в этот момент (в частности - аргументы функции, в которой они созданы).

```c++
std::vector foo(1000, 0);                 // (1) - created before main

int count(int start) {
    static int current = start;           // (2) - Created on first call; 
    return current++;
}

int bad_counter(int b){
    static int a{};                       // (3) Value-initialzation on first call
    a = b;                                // assigment on every call
    return a++;                                
}

int main() {
    foo.push_back(0);
    std::cout << count(5) << std::endl;           // 5
    std::cout << count(5) << std::endl;           // 6
    std::cout << count(0) << std::endl;           // 7
    std::cout << count(101) << std::endl;         // 8
    std::cout << bad_counter(5) << std::endl;     // 5
    std::cout << bad_counter(5) << std::endl;     // 5
    std::cout << bad_counter(0) << std::endl;     // 0
    std::cout << bad_counter(101) << std::endl;   // 101
}

```

Обычно такие переменные помещаются в область глобальных переменных, поэтому на практике имеет смысл создавать
большие объекты именно таким способом, чтобы не тратить память на стэке.

Здесь можно встретить все проблемы, связанные с SIOF [билет 33](https://github.com/khbminus/CppTickets/blob/master/tickets/ticket33.md).
Про порядокс (или его отсутсвие инициализации таких переменных можно почитать [тут](https://en.cppreference.com/w/cpp/language/initialization#Non-local_variables))
и в билете про инициализацию


### Dynamic storage duration

Программа сама полностью управляет временем жизни. При вызове оператора `new Type;` - создаётся
объект, значение этого выражения - указатель на него. Для того, чтобыы уничтожить объект используется
`delete ptr;`.

При попытке удалить объект 2 раза или удалить объект, созданый не при помощи опретаора `new` - 
возникает UB.
              
```c++

struct Foo {
    std::vector<int> vec(100);
}

int main() {
    Foo *ptr = new Foo;
    int bar;
    std::cout << ptr->vec.size() << std::cout;
    delete ptr; // To avoid memory leak
    //delete ptr; // Double free - UB; 
    //delete &bar; // UB;
}

```

Неосвобождённая память живёт до окончания программы - дальше современные ОС её освобождают. Такая ситуация называется утечкой памяти, 
она может вызвать отложенные проблемы.

Есть целый зоопарк new/delete; 

Можно выделять целые массивы при помощи `new Type[n]` - в таком случае освобождать память следует при помощи `delete[] ptr` - 
иначе UB (В том числе при попытке сделать `delete ptr;`). 

Для new работает много инициализаций:

* default - `new Type`
* default - `new Type()`
* default - `new Type{}`
* direct - `new Type(10)`
* direct list - `new Type{10}`

* Инициализация массивов - `new Foo[n]{val1, val2, val3}` - неинициализированные инициализируются по умолчанию.




## Время жизни временных объектов

Временные объекты умирают по завершении вычмсления выражения где они возникли, но при этом если создать
константную ссылку на временный объект, его время жизни продлится, чтобы соответсвовать времени жизни этой ссылки.

Важно, что эффект теряется, если мы инициализируем новую ссылку на временный объект старой.


```c++
int main() {
    const std::vector<int> &first = std::vectorP{0, 0, 0};
    std::cout << first.size() << std::endl; // NO UB
    const std::vector<int> &second = std::vectorP{0, 0, 0}; // would have been dangling if lifetime of second had been wider than lifetime of first;     
}
```

Это может иметь значение, если 


```c++
int const& func(int const& x) {
    return x;
} 


int main() {
    const std::vector<int> &first = func(1);
    std::cout << first; // UB
}
```
т.к. время жизни объекта продлевается только до времени жизни x. x - исчезает после завершения вычисления функции.

Подобное можно встретить и в STL - функция std::min - принимает и возвращает ссылки

```c++
int main() {
    const auto &val = std::min(0, 1); // Dangling reference
}
```
[ну или проблемы range-based for связанная с тем, что его рассахаривание - несколько выражений](https://github.com/Nekrolm/ubbook/blob/master/lifetime/for_loop.md) 
