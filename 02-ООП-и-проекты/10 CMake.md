---
title: CMake
tags: [cpp, cmake, сборка, CMakeLists, библиотеки]
source: cppm-30_CMake
---
# CMake

> [!abstract] О чём заметка
> CMake — кроссплатформенный инструмент, который по описанию проекта в файле `CMakeLists.txt` генерирует файлы сборки для конкретного компилятора и ОС. Вы один раз описываете, из каких файлов состоит программа и что из них получить, а CMake «рассказывает» сборщику, как это собрать. Разберём идею, синтаксис, минимальный проект, библиотеки и работу с Visual Studio.

## Зачем нужен CMake

В отличие от C# или Java, **C++ не кроссплатформенный на уровне исполняемого файла**: программа, скомпилированная под одну архитектуру процессора и ОС, на другой не запустится — её нужно собирать заново с другими параметрами компиляции. Решение — сторонний инструмент, абстрагирующий процесс сборки от ОС и процессора: вы описываете проект на языке инструмента, а он сам выполняет сборку теми средствами, которые нужны в текущей ситуации.

Таких инструментов несколько: GNU Make, Autotools, SCons, Premake, Ninja, Meson, **CMake**. Вручную же сборка многофайлового проекта (см. [[08 Многофайловые проекты и библиотеки]]) и библиотек ([[09 Сборка и подключение библиотек]]) превращается в длинные команды компилятора.

> [!note] CMake не собирает сам
> **CMake — генератор, а не сборщик.** Он читает `CMakeLists.txt` и создаёт файлы для реального сборщика (Visual Studio, Make, Ninja и т. д.), а те запускают компилятор и линковщик (см. [[01 Компиляция и линковка]]).

```text
CMakeLists.txt  ──►  CMake (генератор)  ──►  файлы для сборки  ──►  конкретный сборщик  ──►  исполняемый файл
                                             проектов (.sln,        (MSBuild, make,           и/или библиотека
                                             Makefile, build.ninja)  ninja)
```

Файл `CMakeLists.txt` пишет программист. Обычно он лежит в корне проекта; если в решении несколько проектов — по одному в корне каждого проекта и ещё один в корне решения.

## Файл CMakeLists.txt: синтаксис

Файл содержит сведения о том, из каких исходников состоит проект, как его собирать (исполняемый файл или библиотека), информацию о самом проекте и т. д. Это **специальный язык, не C++**. Команды выглядят как вызовы функций:

```text
<команда>(<аргументы>)
```

**Аргументы разделяются пробелами**, поэтому с пробелами нужно быть внимательным (имена файлов с пробелами берут в кавычки). Команды в CMake регистронезависимы, но принято писать строчными.

### Минимальный проект: три команды

```cmake
cmake_minimum_required(VERSION 3.22.0)   # 1. минимальная версия CMake
project(cmake_demo)                       # 2. имя проекта
add_executable(hello_cmake source.cpp)   # 3. из каких файлов собрать программу
```

| Команда | Назначение |
|---|---|
| `cmake_minimum_required(VERSION <версия>)` | задаёт минимальную версию CMake; если у пользователя версия ниже — обработка остановится с ошибкой. Visual Studio 2022 поддерживает 3.22.2, поэтому указывайте версию не выше |
| `project(<название> [<другие аргументы>])` | имя проекта; дополнительно можно указать версию и языки |
| `add_executable(<название> [<другие аргументы>] <исходные файлы>)` | создаёт исполняемую программу; `<название>` уникально в проекте и определяет имя итогового файла, исходники перечисляются с путями |

Исходный файл `source.cpp`:

```cpp
#include <iostream>

int main()
{
    std::cout << "Hello world!" << std::endl;
}
```

Заголовочные файлы в `add_executable` обычно не перечисляют: они подключаются через `#include`. В слайдах их иногда добавляют в `add_library` — для отображения в IDE.

## Сборка из командной строки

Проверено с CMake 3.28 и g++:

```bash
cmake -S . -B build          # сгенерировать файлы сборки в каталоге build (-S — где CMakeLists.txt, -B — куда складывать)
cmake --build build          # собрать проект тем сборщиком, который выбрал CMake
./build/hello_cmake          # Вывод: Hello world!
```

Сборка **«вне исходников»** (out-of-source) — файлы сборки лежат в отдельной папке `build`, а исходники остаются чистыми; поэтому каталог `build` можно удалить в любой момент и не коммитить в Git (см. [[01 Знакомство с Git]]). Выбрать другой генератор можно флагом `-G` (например, `-G Ninja`), а режим сборки — `-DCMAKE_BUILD_TYPE=Release`.

## CMake в Visual Studio

Visual Studio 2022 умеет работать с CMake «из коробки» (в версии для Windows).

**Проверка установки.** «Средства» → «Получить средства и компоненты» → вкладка «Рабочие нагрузки» → «Разработка классических приложений на C++» → пункт **«Средства CMake C++ для Windows»** должен быть отмечен. Если нет — отметить и нажать «Изменить».

**Создание проекта:**

1. Создать пустую папку (например, `CMakeDemo`) и в ней пустой файл `CMakeLists.txt`.
2. В стартовом окне Visual Studio выбрать «Открыть локальную папку» и указать эту папку. Именно **наличие `CMakeLists.txt` подсказывает Visual Studio, что перед ней проект CMake**; интерфейс при этом выглядит иначе: внизу вывод обработки CMake, справа — «Обозреватель решений — представление папок».
3. Visual Studio сама создаст каталог `out` с подкаталогом `build` для результатов сборки.
4. Добавить файл `source.cpp` (правый клик по папке → «Добавить» → «Добавить новый элемент» → «Файл C++ (.cpp)») и записать код выше.
5. Вписать в `CMakeLists.txt` три команды из раздела выше и сохранить (Ctrl+S) — при сохранении запускается обработка файла и создаются промежуточные файлы.
6. «Сборка» → «Собрать всё». Программа `hello_cmake.exe` появится в `out\build\x64-Debug`.
7. Запуск: стрелочка справа от кнопки запуска → выбрать `hello_cmake.exe` → кнопка запуска или «Отладка» → «Начать отладку». Отладка работает как в обычных проектах: точки останова и т. д.

Дополнительно: в панели «Обозревателя решений» можно переключить представление с «папок» на **«Просмотр целевых объектов CMake»** (CMake Targets View): там видна структура проекта из `CMakeLists.txt`, можно добавлять файлы и цели.

## Библиотеки в CMake

Для проекта из исходников в одной папке:

```text
CMakeDemo/
├── CMakeLists.txt
├── MathPower.h / MathPower.cpp                 # статическая библиотека
├── DynamicMathPower.h / DynamicMathPower.cpp   # динамическая библиотека
└── MathPowerClient.cpp                         # приложение-клиент
```

```cmake
cmake_minimum_required(VERSION 3.22.0)
project(MathPower)

add_library(MathPowerStaticLib  STATIC MathPower.h MathPower.cpp)
add_library(MathPowerDynamicLib SHARED DynamicMathPower.h DynamicMathPower.cpp)

add_executable(MathPowerExe MathPowerClient.cpp)
target_link_libraries(MathPowerExe MathPowerStaticLib MathPowerDynamicLib)
```

- `add_library(<имя> STATIC|SHARED <файлы>)` — как `add_executable`, но создаёт библиотеку; `STATIC` — статическую (`.lib`/`.a`), `SHARED` — динамическую (`.dll`/`.so`).
- `target_link_libraries(<цель> <библиотеки>...)` — связывает приложение с библиотеками; библиотека должна быть собрана раньше, CMake учитывает это сам по зависимостям.

Чтобы в Windows не писать `__declspec(dllexport/dllimport)` (см. [[09 Сборка и подключение библиотек]]), перед `add_library` добавляют:

```cmake
set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)   # экспортировать из DLL все символы
```

Эта конфигурация проверена на Linux с клиентом из заметки про библиотеки: результат `build/` содержит `libMathPowerStaticLib.a`, `libMathPowerDynamicLib.so` и `MathPowerExe`; `./build/MathPowerExe` выводит `5 6 42`. CMake сам прописывает путь к `.so` в исполняемый файл, поэтому запуск из `build` работает без `LD_LIBRARY_PATH`.

> [!note] Почему клиент находит заголовки
> В примере все файлы лежат в одной папке, и `#include "MathPower.h"` находится рядом с файлом. Если библиотека в другом каталоге, нужно указать путь поиска заголовков (см. ниже).

### Проект в нескольких каталогах (дополнение)

Для больших проектов — отдельный `CMakeLists.txt` в каждой папке и корневой, который их объединяет. Современный стиль привязывает пути заголовков к цели:

```text
project/
├── CMakeLists.txt
├── mathlib/    CMakeLists.txt  MathPower.h  MathPower.cpp  DynamicMathPower.h  DynamicMathPower.cpp
└── src/        CMakeLists.txt  main.cpp
```

```cmake
# CMakeLists.txt (корень)
cmake_minimum_required(VERSION 3.22)
project(MathPower LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)              # версия стандарта C++
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_subdirectory(mathlib)               # подключить CMakeLists.txt из подпапки
add_subdirectory(src)
```

```cmake
# mathlib/CMakeLists.txt
add_library(mathpower_static  STATIC MathPower.cpp)
target_include_directories(mathpower_static PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})   # заголовки видны и клиентам

add_library(mathpower_dynamic SHARED DynamicMathPower.cpp)
target_include_directories(mathpower_dynamic PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
set_target_properties(mathpower_dynamic PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS ON)
```

```cmake
# src/CMakeLists.txt
add_executable(app main.cpp)
target_link_libraries(app PRIVATE mathpower_static mathpower_dynamic)
```

`PUBLIC` у `target_include_directories` означает, что каталог нужен и самой библиотеке, и всем, кто с ней линкуется — клиенту не нужно ничего настраивать вручную (в Visual Studio это был шаг с «Дополнительными каталогами включаемых файлов»). Сборка: `cmake -S . -B build && cmake --build build && ./build/src/app` выводит `5 6 42`.

## Частые ошибки

- Файл назван не `CMakeLists.txt` (регистр важен, например `CMakeList.txt` — не сработает).
- В `cmake_minimum_required` указана версия выше установленной — обработка останавливается с ошибкой.
- Забыт исходник в `add_executable`/`add_library` — линковщик сообщает `undefined reference`.
- Имя цели повторяется: `<название>` должно быть уникальным в проекте.
- Лишний пробел или опечатка в аргументах: разделитель — пробел.
- Изменили `CMakeLists.txt`, но не сохранили (в VS регенерация выполняется при сохранении) или не повторили `cmake -S . -B build`.
- Коммит каталогов `out/` и `build/` в репозиторий.

## Кратко (шпаргалка)

- CMake генерирует файлы сборки по `CMakeLists.txt`; сам не компилирует.
- Минимум: `cmake_minimum_required`, `project`, `add_executable`.
- Библиотеки: `add_library(имя STATIC|SHARED файлы)`, связывание — `target_link_libraries(цель библиотеки)`.
- `set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)` — экспорт всех символов DLL без `__declspec`.
- Командная строка: `cmake -S . -B build`, `cmake --build build`.
- Visual Studio: «Открыть локальную папку» с `CMakeLists.txt`, «Сборка → Собрать всё», результат — в `out\build\x64-Debug`.

## Связанные темы
- [[09 Сборка и подключение библиотек]] — что такое статические и динамические библиотеки, которые описываются командами `add_library`.
- [[08 Многофайловые проекты и библиотеки]] — исходные и заголовочные файлы, из которых состоит проект.
- [[01 Компиляция и линковка]] — что на самом деле делают компилятор и линковщик «под капотом» CMake.
- [[08 Тестирование и Unit-тесты]] — тестовые проекты обычно тоже подключаются через CMake.
