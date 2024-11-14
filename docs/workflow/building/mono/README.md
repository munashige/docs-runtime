# Cборка Mono

## Требования к сборке

| Windows  | Linux    | macOS    | FreeBSD  |
| :------: | :------: | :------: | :------: |
| [Требования](../../requirements/windows-requirements.md) | [Требования](../../requirements/linux-requirements.md) | [Требования](../../requirements/macos-requirements.md) | [Требования](../../requirements/freebsd-requirements.md) |

Прежде чем продолжить, используйте одну из ссылок выше, которая соответствует вашей системе. Для корректной сборки необходимо установить все перечисленные пакеты и требования.

## Общие сведения

Чтобы начать работу, нужно собрать runtime Mono и библиотеки. Выполните команду из корня репозитория:

```bash
./build.sh mono+libs
```
на Windows:

```cmd
build.cmd mono+libs
```
Обратите внимание, что по умолчанию собирается конфигурация *Debug*. Она генерирует выходные данные 'debug', которые включают утверждения (asserts), меньше оптимизаций кода и облегчают отладку. Для проверки производительности или для ускоренного выполнения тестов можно собрать версию 'release', добавив флаг `-configuration release` (или `-c release`).


После сборки Mono и библиотек, если вам необходимо работать только с Mono, нужно использовать следующую команду:

```bash
./build.sh mono
```
на Windows:
```cmd
build.cmd mono
```
Бинарные файлы будут доступны в папке `artifacts\bin\mono\<OS>.<arch>.<flavor>` после сборки.

Если вам нужно запустить тесты библиотек или запустить HelloWorld-пример с вашим изменениями в Mono, соберите Mono с помощью этой команды:

```bash
./build.sh mono+libs.pretest
```
на Windows:
```cmd
build.cmd mono+libs.pretest
```

Если вы хотите пропустить восстановление пакетов nuget и внести изменения только в Mono, вам нужно использовать эту команду:
```bash
./build.sh mono --build
```
на Windows:
```cmd
build.cmd mono --build
```

### Полезные аргументы сборки
Ниже представлен список полезных аргументов сборки:

`/p:MonoEnableLLVM=true` - Собирает Mono с LLVM

`/p:MonoEnableLLVM=true /p:MonoLLVMDir=path/to/llvm` - Собирает Mono с LLVM в указанной папке

`/p:MonoEnableLLVM=true /p:MonoLLVMDir=path/to/llvm /p:MonoLLVMUseCxx11Abi=true` - Собирает Mono с LLVM
в указанной папке (LLVM собирается вместе с C++11 ABI)

Для `build.sh`

`/p:DisableCrossgen=true` - Пропускает сборку установщика, если он не нужен (ускоряет операцию)

`/p:KeepNativeSymbols=true` - Сохраняет символы в бинарном файле вместо того, чтобы выносить их в отдельный файл. Это помогает при отладке Mono с помощью lldb.

Сборка имеет и другие аргументы. Используйте `build -?` для ознакомления.

### WebAssembly

См. главу [Сборка WebAssembly](../../../../src/mono/browser/README.md).

### Android

См. главу [Тестирование Android](../../testing/libraries/testing-android.md)

### iOS

См. главу [Тестирование iOS](../../testing/libraries/testing-apple.md)

## NuGet пакеты

Сгенерируйте пакеты NuGet при помощи команды:

```bash
./build.sh packs -runtimeFlavor mono (with optional release configuration)
```
на Windows:
```cmd
build.cmd packs -runtimeFlavor mono (with optional release configuration)
```

В папке `artifacts\packages\<configuration>\Shipping` будут созданы следующие пакеты:

- `Microsoft.NETCore.Runtime.Mono.<version>-dev.<number>.1.nupkg`
- `runtime.<OS>.Microsoft.NETCore.Runtime.Mono.<version>-dev.<number>.1.nupkg`
- `transport.Microsoft.NETCore.Runtime.Mono.<version>-dev.<number>.1.nupkg`
- `transport.runtime.<OS>.Microsoft.NETCore.Runtime.Mono.<version>-dev.<number>.1.nupkg`

## Пример "Hello World"

Этот пример доступен в папке `src/mono/sample/HelloWorld`.
Для запуска выполните следующую команду из этой папки:
```cd ../..
make run
```

## Примечание

Тестовые бинарные файлы недоступны для Mono на данный момент.

Сборка помещает логи в папку `artifacts\log`. Информация в логах может помочь, если сборка возвращает ошибку.

Сборка помещает все выходные данные в папку `artifacts\obj\mono`. Если эта папка удалена, запущенный скрипт начнет полную сборку Mono.
