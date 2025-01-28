# Cборка библиотек

## Быстрый старт

Ниже представлен пример работы с библиотеками на Windows:

```cmd
:: Из корневой папки:
git clean -xdf
git pull upstream main & git push origin main
:: Сборка библиотек в режиме Debug поверх runtime в режиме Release:
build.cmd clr+libs -rc Release
:: Операция выше обычно выполняется раз в день или при значительных изменений в коде.

:: При использовании Visual Studio откройте System.Collections.Concurrent.sln.
build.cmd -vs System.Collections.Concurrent

:: Переключитесь на нужную библиотеку (в данном случае System.Collections.Concurrent):
cd src\libraries\System.Collections.Concurrent

:: Переключитесь на папку тестов:
cd tests

:: Inner loop build / test
:: (При использовании Visual Studio можно запустить тесты c папки)
pushd ..\src & dotnet build & popd & dotnet build /t:test
```

Инструкции для Unix-систем в основном такие же:

```bash
#!/usr/bin/env bash

# Из корневой папки:
git clean -xdf
git pull upstream main; git push origin main
# Сборка библиотек в режиме Debug поверх runtime в режиме Release:
./build.sh clr+libs -rc Release
# Операция выше обычно выполняется раз в день или при значительных изменений в коде.

# Переключитесь на нужную библиотеку (в данном случае System.Collections.Concurrent)
cd src/libraries/System.Collections.Concurrent

# Переключитесь на папку тестов:
cd tests

# Inner loop build / test:
pushd ../src; dotnet build; popd; dotnet build /t:test
```

Для работы c библиотеками и внесения изменений достаточно использовать инструкции выше. Больше информации об использованных командах представлено ниже.

## Общая сборка

Этот документ объясняет, как работать с библиотеками. Для работы с проектами библиотек или запуска тестов библиотек необходимо сначала собрать runtime.
Обычно нужно собирать CoreCLR в конфигурации Release, а библиотеки — в конфигурации Debug.
Больше информации в главе [Инструкция по работе с репозиторием](../../README.md#Configurations).

Следующая команда соберет Release-версию CoreCLR (и CoreLib), а также библиотеки и установщик в режиме Debug:

Для Linux:

```bash
./build.sh -rc Release
```

Для Windows:

```cmd
./build.cmd -rc Release
```

Ниже представлена подробная информация о сборке и тестировании runtime и библиотек.

### Подробная информация

Вышеуказанные команды собирают библиотеки в режиме _Debug_ (по умолчанию), используя ранее собранный runtime в конфигурации _Release_.

Сборка библиотек состоит из двух логических компонентов:

1. Нативная сборка, которая производит "_шимы_" ("_shims_") (нужны для интерфейса между ОС и управляемым кодом);
2. Управляемая сборка, которая производит код MSIL и пакеты NuGet.

Указанные выше команды соберут оба компонента.

Настройки сборки (BuildTargetFramework, TargetOS, Configuration и Architecture) имеют установки по умолчанию в зависимости от того, где выполняется сборка (т.е. какая ОС или архитектура используется). Есть несколько сокращений для отдельных параметров, которые можно передать скриптам сборки:

-   `-framework|-f` указывает фреймворк для сборки. Возможные значение включают `net10.0` (последняя версия .NET ) и `net48` (последняя версия .NET Framework). (настройка msbuild `BuildTargetFramework`);
-   `-os` указывает OC для сборки. По умолчанию используется ОС, на которой вы работаете, но возможные значения включают `windows`, `unix`, `linux` или `osx`. (настройка msbuild `TargetOS`);
-   `-configuration|-c Debug|Release` управляет уровнем оптимизации, который компиляторы используют для сборки. По умолчанию используется `Debug`(настройка msbuild `Configuration`);
-   `-arch` определяет архитектуру для сборки. По умолчанию используется `x64`, но возможные значения включают `x64`, `x86`, `arm` и `arm64`. (настройка msbuild `TargetArchitecture`)

Подробную информацию о настройках билда см. в [этой главе](../../../coding-guidelines/project-guidelines.md#build-pivots).

При вызове скрипта `build` без каких-либо действий по умолчанию выполняется цепочка действий `-restore -build`.

По умолчанию скрипт `build` собирает только продуктовые библиотеки и не собирает тесты. Чтобы включить в сборку тесты нужно использовать подмножество `libs.tests`. Для запуска тестов используйте флаг `-test` вместо `-build`. Например, `build.cmd/sh libs.tests -test`.

Чтобы указать только библиотеки, используйте `libs`.

**Примеры**

-   Сборка в режиме _Release_ для архитектуры x64 (подразумеваются восстановление и сборка, так как никаких действий не передается):

```bash
./build.sh libs -c Release -arch x64
```

-   Сборка src assemblies, сборка и запуск тестов. Обратите внимание, что запуск всех тестов занимает много времени:

```bash
./build.sh libs -test
```

-   Очистка всей папки артефактов

```bash
./build.sh -clean
```

Замените `./build.sh` на `build.cmd` для Windows.

### Сборка нативных компонентов и санитайзеров

Нативные компоненты библиотек могут быть собраны с использованием нативных санитайзеров, таких как AddressSanitizer, чтобы помочь выявить проблемы с работой памяти. Чтобы собрать проект с нативными санитайзерами используйте флаг `-fsanitize`, например:

```bash
build.sh -s libs -fsanitize address
```

При сборке репозитория с любыми нативными санитайзерами нужно собирать все нативные компоненты в репозитории с одним и тем же набором санитайзеров.

### Как собрать только нативные компоненты

Сборка библиотек частично содержит нативный код, что включает в себя шимы для libc, openssl, gssapi и zlib. Скрипт сборки использует CMake для генерации Makefile с использованием clang. Сборка также использует git для генерации некоторой информации о версиях.

**Примеры**

-   Сборка в режиме отладки для x64 архитектур:

```bash
./src/native/libs/build-native.sh debug x64
```

-   Сборка и обновление binplace (напр. для testhost) - необходимо при итерациях нативных компонентов:

```bash
dotnet.sh build src/native/libraries/build-native.proj
```

-   Следующий пример показывает, как выполнить кросс-компиляцию для arm - архитектур:

```bash
./src/native/libs/build-native.sh debug arm cross verbose
```

Замените `./build.sh` на `build.cmd` для Windows.

## Сборка отдельных библиотек

Аналогично сборке всего репозитория с помощью `build.cmd` или `build.sh` из корневой папки можно собрать отдельные проекты, опираясь на структуру репозитория и передавая соответствующие директории в скрипт сборки. Также поддерживается и сокращения для библиотек, поэтому корневую папку `src` можно не указывать. При работе с отдельной директорией скрипт рекурсивно находит и собирает все проекты, которые там находятся. Примеры находятся ниже.

**Примеры**

-   Сборка всех проектов для указанной библиотеки (например, System.Collections), а также запуск тестов:

```bash
 ./build.sh -projects src/libraries/*/System.Collections.sln
```

-   Сборка тестов для проекта библиотеки:

```bash
 ./build.sh -projects src/libraries/System.Collections/tests/*.csproj
```

-   Поддерживаются также и все вышеперечисленные параметры, такие как фреймворк или конфигурация. Обратите внимание, что эти параметры должны быть указаны после папки, например:

```bash
 ./build.sh -projects src/libraries/*/System.Collections.sln -f net472 -c Release
```

Поскольку `dotnet build` работает на Unix и Windows, эта команда будет использоваться в командах ниже.

В папке `src` находятся другие папки, которые представляет разные ассамблеи в Библиотеках. Больше информации см. в главе [Руководство по проекту](../../../coding-guidelines/project-guidelines.md).

Например папка `src\libraries\System.Diagnostics.DiagnosticSource` содержит исходный код для ассамблея _System.Diagnostics.DiagnosticSource.dll_.

Вы можете собрать DLL для _System.Diagnostics.DiagnosticSource.dll_ из папки `src\libraries\System.Diagnostics.DiagnosticsSource\src`, используя команду `dotnet build`. Собранный DLL находится в папке `artifacts\bin\System.Diagnostics.DiagnosticSource`, а также в папке `artifacts\bin\runtime\[$(BuildTargetFramework)-$(TargetOS)-$(Configuration)-$(TargetArchitecture)]`.

Тесты для _System.Diagnostics.DiagnosticSource.dll_ могут быть собраны из папки
`src\libraries\System.Diagnostics.DiagnosticSource\tests`, используя команду `dotnet build`.

Некоторые библиотеки также могут иметь папки `ref` и/или `pkg`. Их можно собрать аналогичным образом, введя команду `dotnet build` в соответствующих директорияъ.

Для библиотек, которые имеют несколько целевых фреймворков, необходимые фреймворки должны быть перечислены в настройках <TargetFrameworks>. При сборке csproj для _BuildTargetFramework_ будет выбрана и установлена наиболее совместимый фреймворк. Для получения дополнительной информации см. в главе [Руководство по проекту](../../../coding-guidelines/project-guidelines.md).

**Примеры**

-   Сборка проекта на Linux:

```bash
dotnet build System.Net.NetworkInformation.csproj /p:TargetOS=linux
```

-Сборка Release-версии библиотеки:

```bash
dotnet build -c Release System.Net.NetworkInformation.csproj
```

### Итерация изменений System.Private.CoreLib

После полной сборки и после внесения изменений в `System.Private.CoreLib` для тестирования вам потребуется обновленная версия `System.Private.CoreLib` в `testhost`. Для этого нужно собрать подмножество `libs.pretest`, которое выполняет настройку `testhost`, включая копирование `System.Private.CoreLib`.

После этого запустите сборку runtime:

```cmd
build.cmd clr -rc Release
```

Итерация изменений `System.Private.CoreLib` выполняется следующей командой:

```cmd
build.cmd clr.corelib+clr.nativecorelib+libs.pretest -rc Release
```

При этом `System.Private.CoreLib` будет собран в режиме Release. Затем будет выполнена кроссгенерация и `testhost` будет обновлен до последней версии corelib.

Тот же процесс используется для Mono runtime и подмножетства `mono.corelib+libs.pretest`.

### Сборка Mono

По умолчанию библиотеки собираются с использованием СoreCLR – `System.Private.CoreLib.dll`. Для сборки Mono нужно передать аргумент `/p:RuntimeFlavor=Mono`.

```cmd
.\build.cmd libs /p:RuntimeFlavor=Mono
```

### Сборка для других ОС-ей

По умолчанию сборка из корневой директории будет собирать только библиотеки для той ОС, на которой вы работаете. Вы можете собрать для другой ОС, используя соответствующий флаг `./build.sh libs -os [value]`.

Обратите внимание, что обычно собрать нативные компоненты для другой ОС невозможно, но можно собрать управляемые компоненты. Для этого используйте отдельный проект и скрипт сборки, передав `/p:BuildNative=false`.

### Сборка в режиме Release или Debug

По умолчанию сборка из корневой папки или внутри проекта соберет библиотеки в режиме _Debug_. Можно собрать библиотеки в режиме _Release_, используя `./build.sh libs -c Release`.

### Сборка для других архитектур

Вы можете собирать 32 или 64-битные бинарные файлы, а также работать и с другими архитектурами, указав в корне `./build.sh libs -arch [value]` или в проекте `/p:TargetArchitecture=[value]` после команды `dotnet build`.

## Работа в Visual Studio

Если вы работаете на Windows и используете Visual Studio, вы можете открыть отдельные проекты с библиотеками. Visual Studio можно использовать для сборки, отладки и запуска тестов.

## Отладка

Visual Studio 2022 версии 17.5 и выше можно использовать для проверки подписей перед загрузкой библиотек отладки. Дополнительную информацию можно найти по ссылке: https://aka.ms/vs/unsigned-dotnet-debugger-lib.

## Запуск тестов

Для получения подробной информации о запуске тестов в Visual Studio, [перейдите в эту главу](../../testing/visualstudio.md).

Больше информации о запуске тестов см. в [этой главе](../../testing/libraries/testing.md) document.

## Сборка пакетов

Для сборки этого пакета достаточно выполнить команду `dotnet pack` из проекта `src` после успешной сборки _.NETCoreApp_ из корневой папки:

```cmd
build libs
dotnet.cmd pack src\libraries\System.Text.Json\src\
```

Для команд `dotnet build` или `dotnet publish` также можно указать желаемую конфигурацию с помощью флага `-c`, например:

```cmd
dotnet.cmd pack src\libraries\System.Text.Json\src\ -c Release
```
