---
title: Тестирование и Unit-тесты
tags: [cpp, тестирование, unit-тесты, assert, catch2, googletest]
source: Тестирование_и_Unit-тесты
---
# Тестирование и Unit-тесты

> [!abstract] О чём заметка
> **Unit-тестирование (модульное тестирование)** — проверка мелких частей программы (функций, классов) по отдельности. Заметка объясняет, зачем это нужно, как выглядит хороший тест, как написать собственный мини-фреймворк на `assert` и исключениях и как устроены тесты в профессиональных фреймворках (Microsoft Unit Testing Framework, Catch2). Тестирование ищет **логические ошибки**, которых не видит компилятор (см. [[06 Исключения и обработка ошибок]]).

> [!warning] Фреймворки в этой среде не установлены
> Примеры с Catch2 и Microsoft CppUnitTest ниже приведены **как в слайдах** и помечены «требует установки фреймворка» — они не компилировались. Все остальные примеры (assert, самодельные раннеры) скомпилированы и запущены с `g++ -std=c++17 -Wall`.

## Что такое unit-тестирование

**Unit-тест** проверяет отдельный компонент программы **изолированно**, чтобы убедиться, что он работает как задумано. Зачем:

- **защита от регрессии** — после правок сразу видно, что сломалось то, что раньше работало;
- **документация** — тест показывает, как использовать код и какой результат ожидается;
- **безопасный рефакторинг** — код можно переписывать, опираясь на тесты.

Не путайте: **тестирование** — обязательный этап разработки, ищет ошибки; **отладка** — поиск причины уже известной ошибки.

### Характеристики хорошего теста

| Свойство | Смысл |
|---|---|
| Скорость выполнения | тысячи тестов должны проходить за секунды |
| Изолируемость | тест не зависит от других тестов, диска, сети, порядка запуска |
| Повторяемость | при том же коде всегда один и тот же результат |
| Самопроверка | тест сам говорит «прошёл/упал», без чтения вывода глазами |
| Скорость написания | писать тесты должно быть просто |

### Тестовые дублёры

Чтобы изолировать компонент от его зависимостей (база данных, сеть), реальные объекты подменяют **тестовыми дублёрами (test doubles)**:

| Вид | Что это |
|---|---|
| Пустышка (dummy) | заполнитель параметра, не содержит поведения, реально никогда не вызывается |
| Подделка (fake) | упрощённая рабочая реализация (например, хранилище в памяти вместо БД) |
| Заглушка (stub) | отдаёт заранее заготовленные ответы, почти без логики |
| Мок (mock) | самый мощный: сложная логика ответов, проверка числа и параметров вызовов, исключения при неожиданных вызовах |

### Лучшие практики

- **Избегайте зависимостей от инфраструктуры** (файлы, сеть, БД) — они делают тесты медленными и хрупкими; оставьте их интеграционным тестам.
- **Называйте тесты по трём частям:** имя тестируемого метода, сценарий, ожидаемое поведение — например `Mean_EmptyArray_Throws`.
- **Шаблон AAA — Arrange, Act, Assert:** подготовить объект, выполнить действие, проверить результат.
- **Минимум входных данных:** самые простые значения, достаточные для проверки именно этого поведения.
- **Без управляющих конструкций** (`if`, `for`, `while`, `switch`, ручная склейка строк) в тесте — иначе тест сам может содержать ошибку.
- **Одно действие на тест:** отдельный тест на каждое поведение или параметризованные тесты.

```cpp
void test_mean_negativeNumbers_returnsAverage()
{
    ExtArray<int> v{-1, -3};          // Arrange
    double r = v.mean();              // Act
    AssertEq(r, -2.0);                // Assert
}
```

## Тестируемый класс

На протяжении заметки тестируется шаблонный класс `ExtArray<T>` — вектор с методами статистики: среднее (`mean`), медиана (`median`) и мода (`mode`, возвращает пару «значение, сколько раз встретилось»). Слайды показывали только скриншоты и тесты, поэтому реализация написана здесь заново (см. [[05 Шаблоны и функторы]], [[06 STL часть 1]]):

```cpp
// extarray.h
#pragma once
#include <vector>
#include <algorithm>
#include <map>
#include <numeric>
#include <stdexcept>
#include <utility>

template <typename T>
class ExtArray : public std::vector<T> {
public:
    using std::vector<T>::vector;               // наследуем конструкторы, в т.ч. от списка инициализации

    double mean() const {
        if (this->empty()) throw std::invalid_argument("empty array");
        return std::accumulate(this->begin(), this->end(), 0.0) / this->size();
    }
    double median() const {
        if (this->empty()) throw std::invalid_argument("empty array");
        std::vector<T> s(this->begin(), this->end());
        std::sort(s.begin(), s.end());
        size_t n = s.size();
        return n % 2 ? s[n / 2] : (s[n / 2 - 1] + s[n / 2]) / 2.0;
    }
    std::pair<T, int> mode() const {            // (значение, число повторов)
        if (this->empty()) throw std::invalid_argument("empty array");
        std::map<T, int> cnt;
        for (const T& x : *this) ++cnt[x];
        auto best = std::max_element(cnt.begin(), cnt.end(),
            [](const auto& a, const auto& b) { return a.second < b.second; });
        return {best->first, best->second};
    }
};
```

Ожидаемые значения: для `{4, 2, 7, 3, -5, 0, -3, 1}` среднее $9/8 = 1{,}125$; для `{4.5, 0.7, 10.8, -3.5}` медиана $(0{,}7 + 4{,}5)/2 = 2{,}6$; для `{4, 2, 7, 2, 5, -5, 0, 5, 5}` мода — `5` (3 раза).

## Ручное unit-тестирование

Главное преимущество ручного подхода — полная свобода: тест можно сделать именно таким, какой нужен.

### Базовый вариант на assert

Простейший фреймворк строится на макросе `assert` из `<cassert>`:

```cpp
#include <cassert>
#include <iostream>
#include "extarray.h"

void test_mean()   { ExtArray<int> v{4, 2, 7, 3, -5, 0, -3, 1}; assert(v.mean() == 1.125); std::cout << "Mean test OK\n"; }
void test_median() { ExtArray<double> v{4.5, 0.7, 10.8, 2.6, -3.5}; assert(v.median() == 2.6); std::cout << "Median test OK\n"; }
void test_mode()   { ExtArray<int> v{4, 2, 7, 2, 5, -5, 0, 5, 5}; assert(v.mode() == std::pair(5, 3)); std::cout << "Mode test OK\n"; }

int main() { test_mean(); test_median(); test_mode(); }
// Вывод:
// Mean test OK
// Median test OK
// Mode test OK
```

Если условие в `assert` истинно, программа идёт дальше; если ложно — **аварийно завершается** с сообщением. Если заменить ожидание на неверное (`== 1.5`), получим:

```text
m1f: manual1f.cpp:4: void test_mean(): Assertion `v.mean() == 1.5' failed.
Aborted
```

> [!warning] Запятые в assert
> `assert` — макрос, и запятая внутри угловых скобок (`std::pair<int,int>(5, 3)`) принимается за разделитель аргументов макроса: ошибка «macro "assert" passed 2 arguments, but takes just 1». Спасает запись `std::pair(5, 3)` (в круглых скобках; C++17) или дополнительные скобки вокруг всего выражения. Кроме того, при `-DNDEBUG` (типично для Release-сборки) все `assert` вырезаются — проверки исчезают.

**Недостатки базового варианта:**
1. первый же провал завершает программу — остальные тесты не выполняются;
2. не видно, какие значения сравнивались;
3. сообщения идут в стандартный вывод и смешиваются с другими данными;
4. каждый тест нужно запускать отдельно.

### Улучшенный вариант: исключения и раннер

Решение — заменить `assert` на шаблонную функцию, которая **бросает исключение** с описанием, и обернуть каждый тест в `try/catch`:

```cpp
#include <iostream>
#include <sstream>
#include <stdexcept>
#include <string>
#include "extarray.h"

template <typename T, typename U>
void AssertEq(const T& a, const U& b) {
    if (a != b) {
        std::ostringstream err;
        err << "Assertion failed: " << a << " != " << b;     // показываем оба значения
        throw std::runtime_error(err.str());
    }
}

template <typename TestFunc>
void RunOneTest(TestFunc f, const std::string& name) {
    try { f(); std::cerr << name << " OK" << std::endl; }
    catch (const std::exception& e) { std::cerr << name << " FAIL: " << e.what() << std::endl; }
}

void test_mean()   { ExtArray<int> v{4, 2, 7, 3, -5, 0, -3, 1}; AssertEq(v.mean(), 1.125); }
void test_median() { ExtArray<double> v{4.5, 0.7, 10.8, -3.5}; AssertEq(v.median(), 2.6); }
void test_median_bad() { ExtArray<double> v{4.5, 0.7, 10.8, -3.5}; AssertEq(v.median(), 3.0); } // намеренно неверный
void test_empty()  { ExtArray<int> v; v.mean(); }            // код сам бросает исключение
void test_mode()   { ExtArray<int> v{4, 2, 7, 2, 5, -5, 0, 5, 5}; AssertEq(v.mode().first, 5); }

void RunAllTests() {
    RunOneTest(test_mean, "test_mean");
    RunOneTest(test_median, "test_median");
    RunOneTest(test_median_bad, "test_median_bad");
    RunOneTest(test_empty, "test_empty");
    RunOneTest(test_mode, "test_mode");
}
int main() { RunAllTests(); }
```

Вывод (в `stderr`; `catch (const std::exception&)` ловит и ошибки сравнения, и исключения самого тестируемого кода):

```text
test_mean OK
test_median OK
test_median_bad FAIL: Assertion failed: 2.6 != 3
test_empty FAIL: empty array
test_mode OK
```

Теперь упавший тест не мешает остальным, а в сообщении видны сравниваемые значения. Это уже по сути минимальный фреймворк. Про исключения: [[06 Исключения и обработка ошибок]].

> [!tip] Сравнение дробных чисел
> Сравнение `double` через `==` работает в примерах только потому, что значения точно представимы (`1.125`) или получаются одинаковой цепочкой операций. В общем случае сравнивайте с допуском: `std::abs(a - b) < 1e-9` (в Catch2 для этого есть `Approx`).

### Мини-раннер в стиле Catch2

Можно пойти дальше и сымитировать макросы настоящих фреймворков: `TEST_CASE` автоматически регистрирует тест, `CHECK` фиксирует провал и продолжает, `REQUIRE` — прерывает тест. Работающий пример (макросы + реестр тестов):

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <functional>
#include <cmath>

struct Test { std::string name; std::function<void()> fn; };
inline std::vector<Test>& registry() { static std::vector<Test> r; return r; }
struct Registrar {
    Registrar(std::string n, std::function<void()> f) { registry().push_back({std::move(n), std::move(f)}); }
};
static int g_failed_checks = 0;

#define CONCAT_(a, b) a##b
#define CONCAT(a, b) CONCAT_(a, b)
#define TEST_CASE(name) \
    static void CONCAT(test_fn_, __LINE__)(); \
    static Registrar CONCAT(reg_, __LINE__)(name, CONCAT(test_fn_, __LINE__)); \
    static void CONCAT(test_fn_, __LINE__)()
#define CHECK(expr) do { if (!(expr)) { ++g_failed_checks; \
    std::cout << "  CHECK failed: " #expr " (" << __FILE__ << ":" << __LINE__ << ")\n"; } } while (0)
#define REQUIRE(expr) do { if (!(expr)) { ++g_failed_checks; \
    std::cout << "  REQUIRE failed: " #expr "\n"; throw 1; } } while (0)

#include "extarray.h"

TEST_CASE("Mean value") {
    ExtArray<int> a{4, 2, 7, 3, -5, 0, -3, 1};
    CHECK(a.mean() == 1.125);
    ExtArray<double> b{4.5, 0.7, 10.8, -3.5};
    CHECK(b.mean() == 3.125);
}
TEST_CASE("Median value") {
    ExtArray<double> b{4.5, 0.7, 10.8, -3.5};
    CHECK(std::abs(b.median() - 2.6) < 1e-9);
    CHECK(b.median() == 99.0);       // провал: тест продолжится
    REQUIRE(b.size() == 5);          // провал: тест прервётся
    CHECK(false);                    // сюда не дойдём
}
TEST_CASE("Mode value") {
    ExtArray<int> a{4, 2, 7, 2, 5, -5, 0, 5, 5};
    REQUIRE(a.mode().first == 5);
}

int main() {
    int failed = 0;
    for (auto& t : registry()) {
        int before = g_failed_checks;
        std::cout << "[ RUN  ] " << t.name << "\n";
        try { t.fn(); } catch (...) {}
        bool ok = g_failed_checks == before;
        std::cout << (ok ? "[  OK  ] " : "[ FAIL ] ") << t.name << "\n";
        if (!ok) ++failed;
    }
    std::cout << registry().size() - failed << " of " << registry().size() << " tests passed\n";
    return failed ? 1 : 0;      // ненулевой код выхода — сигнал для CI
}
```

Вывод (код возврата `1`):

```text
[ RUN  ] Mean value
[  OK  ] Mean value
[ RUN  ] Median value
  CHECK failed: b.median() == 99.0 (manual3.cpp:32)
  REQUIRE failed: b.size() == 5
[ FAIL ] Median value
[ RUN  ] Mode value
[  OK  ] Mode value
2 of 3 tests passed
```

Идея `TEST_CASE` — зарегистрировать функцию в глобальном реестре до входа в `main` через конструктор статического объекта; именно так работают настоящие фреймворки. Макросы подробнее — в [[07 Препроцессор и макросы]].

## Автоматизация тестов в IDE

Автоматизированное тестирование позволяет создать и выполнить гораздо больше тестов, чем вручную, — растёт покрытие и находится больше ошибок. Современные IDE имеют встроенную поддержку фреймворков: Microsoft Visual Studio (с версии 17.0) включает Microsoft Unit Testing Framework, Google Test, Boost.Test и CTest.

Порядок работы со встроенным Microsoft Unit Testing Framework:

1. Создать проект с unit-тестами — он добавляется отдельным проектом в решение (solution).
2. Добавить в нём **ссылку на тестируемый проект**, чтобы видеть его функции и классы.
3. Подключить `#include` заголовков тестируемого кода.
4. Написать тесты: класс-заглушка и пример метода создаются автоматически; макросы `TEST_CLASS` и `TEST_METHOD` (часть Microsoft Native Test Framework) делают методы видимыми для Visual Studio. `TEST_METHOD` возвращает `void`, а результат проверяют статические методы класса `Assert`.
5. Запускать из окна **«Обозреватель тестов»**: можно все тесты или выбранные, в том числе в режиме отладки с точками останова; видны пройденные и упавшие.

Пример теста из слайдов (**требует установки фреймворка / Visual Studio; не компилировался**; в слайдах для медианы ожидалось `2.6` на четырёх элементах, что верно):

```cpp
namespace NativeUnitTest {
    TEST_CLASS(ExtendedArrayTest) {
    public:
        TEST_METHOD(Mean) {
            ExtArray<int> v1{4, 2, 7, 3, -5, 0, -3, 1};
            Assert::AreEqual(v1.mean(), 1.125);
        }
        TEST_METHOD(Median) {
            ExtArray<double> v2{4.5, 0.7, 10.8, -3.5};
            Assert::AreEqual(2.6, v2.median());
        }
        TEST_METHOD(Mode) {
            ExtArray<int> v1{4, 2, 7, 2, 5, -5, 0, 5, 5};
            Assert::AreEqual(5, v1.mode().first);
        }
    };
}
```

## Библиотека Catch2

**Catch2** — современный фреймворк модульного тестирования для C++ с открытым исходным кодом (лицензия Boost Software License). Распространён из-за простоты: тесты пишутся обычными выражениями `a == b`, а при падении фреймворк сам показывает значения обеих частей.

### Установка (по слайдам)

1. Скачать релиз библиотеки с GitHub (репозиторий `catchorg/Catch2`, раздел releases) — zip или tar.gz.
2. Распаковать в любую папку (это не каталог установки).
3. В командной строке перейти в распакованный каталог (при необходимости — с правами администратора).
4. Выполнить:

```bash
cmake -Bbuild -H. -DBUILD_TESTING=OFF -DCMAKE_INSTALL_PREFIX=<каталог установки>
cmake --build build/ --target install
```

Предварительно должен быть установлен CMake (см. [[10 CMake]]).

### Макросы проверки

Два семейства:

- **`REQUIRE(expr)`** — при провале **прерывает** текущий тест-кейс;
- **`CHECK(expr)`** — фиксирует провал, но тест **продолжается**.

Они должны находиться внутри `TEST_CASE`. Тест-кейс можно разбить на секции `SECTION`, логически группирующие проверки.

### Пример (требует установки Catch2; не компилировался)

В слайдах пример был с синтаксической ошибкой (пропущена закрывающая скобка у первого `TEST_CASE`) — ниже исправлено. Файл `test.cpp`:

```cpp
#include <catch2/catch_all.hpp>     // Catch2 v3; макрос CATCH_CONFIG_MAIN в v3 не нужен
#include "extended_array.h"

TEST_CASE("Mean value") {
    SECTION("int numbers") {
        ExtArray<int> v1{4, 2, 7, 3, -5, 0, -3, 1};
        REQUIRE(v1.mean() == 1.125);
    }
    SECTION("real numbers") {
        ExtArray<double> v2{4.5, 0.7, 10.8, -3.5};
        REQUIRE(v2.mean() == 3.125);
    }
}

TEST_CASE("Median value") {
    SECTION("real numbers") {
        ExtArray<double> v2{4.5, 0.7, 10.8, -3.5};
        REQUIRE(v2.median() == 2.6);
    }
}

TEST_CASE("Mode value") {
    SECTION("int numbers") {
        ExtArray<int> v1{4, 2, 7, 2, 5, -5, 0, 5, 5};
        REQUIRE(v1.mode().first == 5);
    }
}
```

> [!note] Каждая SECTION выполняется отдельно
> Catch2 запускает тест-кейс заново для каждой секции, поэтому переменные, созданные до `SECTION`, в каждой секции «свежие».

### Сборка через CMake

Установленный Catch2 экспортирует два target'а:

- `Catch2::Catch2` — если вы пишете свою `main()`;
- `Catch2::Catch2WithMain` — если подойдёт встроенная `main` фреймворка (и только он, не вместе с первым). Он добавляет пути включения и подключает нужные библиотеки.

```cmake
cmake_minimum_required(VERSION 3.5)
project(tests LANGUAGES CXX VERSION 0.0.1)

find_package(Catch2 3 REQUIRED)
add_executable(tests test.cpp)
target_link_libraries(tests PRIVATE Catch2::Catch2WithMain)
```

Запуск исполняемого файла выполняет все тест-кейсы; при успехе Catch2 печатает, что все тесты пройдены, а при ошибках — список проваленных проверок с выражением и фактическими значениями.

### Для сравнения: GoogleTest

Тот же тест на GoogleTest (**требует установки фреймворка; здесь не приведён в слайдах и не компилировался**) выглядит похоже: `TEST(Набор, Имя) { EXPECT_EQ(a, b); }` — аналог `CHECK`, `ASSERT_EQ` — аналог `REQUIRE`.

## Задачи для практики

- Напишите функцию `isPrime(int)` и набор тестов: 0, 1, 2, 17, 25, отрицательное. *Решение:* по одному тесту на сценарий в стиле AAA, либо цикл по таблице значений в раннере (в самих тестах циклов лучше избегать — используйте параметризованные тесты).
- Добавьте в `ExtArray` метод `max()` и тест на пустой массив, ожидающий исключение. *Решение:* в раннере — `try { v.max(); FAIL } catch (const std::invalid_argument&) { OK }`.
- Расширьте `AssertEq` до `AssertNear` с допуском для `double`.
- Добавьте в мини-раннер подсчёт времени каждого теста через `<chrono>`.

## Частые ошибки

- Один тест проверяет сразу много вещей — непонятно, что именно сломалось.
- Тесты зависят друг от друга или от внешних файлов/сети — становятся нестабильными.
- Сравнение `double` через `==` без допуска.
- `assert` в «боевых» проверках: в Release (`NDEBUG`) он исчезает.
- Тест, который никогда не падает (проверка тавтологии), — всегда проверяйте, что тест может упасть.
- Логика в самих тестах (циклы, `if`) — тест сам становится источником ошибок.

## Кратко (шпаргалка)

| Подход | Плюсы | Минусы |
|---|---|---|
| `assert` | ноль зависимостей, 1 строка | падает на первом же провале, нет значений |
| Исключения + раннер | продолжает после провала, показывает значения | свой код надо поддерживать |
| Фреймворк (Catch2, GoogleTest, MS Test) | авторегистрация, отчёты, секции, моки, интеграция с IDE/CI | нужна установка |

Правила: AAA; одно поведение — один тест; имя = метод + сценарий + ожидание; без зависимостей от инфраструктуры; `CHECK` — продолжить, `REQUIRE` — прервать.

## Ссылки
- Catch2: https://github.com/catchorg/Catch2

## Связанные темы
- [[06 Исключения и обработка ошибок]] — исключения лежат в основе самодельного раннера.
- [[05 Шаблоны и функторы]] — `AssertEq` и `RunOneTest` — шаблонные функции.
- [[10 CMake]] — подключение Catch2 и сборка тестовых проектов.
- [[07 Препроцессор и макросы]] — устройство макросов `TEST_CASE`, `CHECK`, `REQUIRE`, `assert`.
