# Сборка библиотек для WebAssembly

## Требования

Требования к сборке библиотек для вашей операционной системы описаны в главе [Инструкция по работе с репозиторием](../../README.md#Build_Requirements).

## Сборка

На данный момент для сборки WebAssembly не требуются другие зависимости. Emscripten автоматически загружается и устанавливается в процессе сборки. 
Необходимая конфигурация для сборки представлена [в этой главе](../../README.md#Configurations). Этот раздел описывает процесс сборки runtime или библиотек.

При повторной сборке с помощью `build.sh` после изменения кода необходимо убедиться, что подмножества `mono.wasmruntime` и `libs.pretest` включены только для изменений в Mono. В противном случае этот каталог не будет обновлен (подробности ниже).

__Примечание: не используйте разные конфигурации для runtime и библиотек.__

На данный момент невозможно использовать разные конфигурации для runtime и библиотек. Поэтому нельзя указывать режим Release `-runtimeConfiguration` и Debug `-libraryConfiguration` (или `-configuration`) или наоборот. То же самое касается однопоточных и многопоточных конфигураций.

Необходимо использовать флаг `-configuration` только с режимом `Debug` или `Release`. Нельзя указывать  `-runtimeConfiguration` и `-libraryConfiguration`.

Проблема прослеживается по ссылке https://github.com/dotnet/runtime/issues/42553.


## Cборка System.Private.CoreLib и runtime Mono

При работе с Mono необходимо собрать runtime и [System.Private.CoreLib](../../../design/coreclr/botr/corelib.md) с помощью следующей команды:

```bash
./build.sh mono -os browser -c Debug|Release
```

Чтобы собрать только `System.Private.CoreLib` без runtime, используйте `Mono.CoreLib`:

```bash
./build.sh mono.corelib -os browser -c Debug|Release
```

Чтобы собрать только runtime без `System.Private.CoreLib`, используйте подмножество Mono.Runtime:

```bash
./build.sh mono.runtime -os browser -c Debug|Release
```

Сборка Mono/System.Private.CoreLib и управляемых библиотек::

```bash
./build.sh mono+libs -os browser -c Debug|Release
```

## Сборка файлов WebAssembly

Файлы WebAssembly собираются после сборки исходников библиотек и становятся доступными в папке артефактов. Если вы работаете с кодовой базой и хотите скомпилировать только эти модули, то сборка подмножества `Mono.WasmRuntime` позволит это сделать:

```bash
./build.sh mono.wasmruntime -os browser -c Debug|Release
```

## Обновление встроенного пакета runtime

Если вы не запускаете подмножество `Libs`, то вы можете использовать подмножество `Libs.PreTest`, чтобы скопировать обновленные бинарные файлы runtime/corelib в пакет runtime, который используется для запуска тестов:

```bash
./build.sh libs.pretest -os browser -c Debug|Release
```

## Сборка только нативных компонентов

Сборка библиотек включает в себя часть нативного код. Это включает шимы (shims) для `libc`, `openssl`, `gssapi` и `zlib`. Система сборки использует CMake для генерации Makefile с использованием clang. Сборка также использует git для генерации информации о версиях.

```bash
./build.sh libs.native -os browser -c Debug|Release
```

## Сборка отдельных библиотек

Отдельные проекты и библиотеки можно собрать, указав конфигурацию сборки.

**Примеры**

- Чтобы собрать все проекты для данной библиотеки (например, `System.Net.Http`), включая тесты:

```bash
./build.sh -os browser -c Release --projects <full-repository-path>/src/libraries/System.Net.Http/System.Net.Http.sln
```

- Чтобы собрать только исходный проект данной библиотеки (например, System.Net.Http):

```bash
 ./build.sh -os browser -c Release --projects <full-repository-path>/src/libraries/System.Net.Http/src/System.Net.Http.csproj
```

Больше информации и примеров представлено [в этой главе](./README.md#building-individual-libraries).

## Примечания

Сборка в режиме `Debug` устанавливает следующие переменные окружения по умолчанию:

- Отладка и логирование, которые будут записывать информацию о garbage collection в консоль:

```
MONO_LOG_LEVEL=debug
MONO_LOG_MASK=gc
```

__Пример__:
```
L: GC_MAJOR_SWEEP: major size: 752K in use: 39K
L: GC_MAJOR: (user request) time 3.00ms, stw 3.00ms los size: 0K in use: 0K
```

- Вывод `System.Diagnostics.Debug` перенаправляется в `stderr`, который будет отображаться в консоли:

```
    // Установка этой переменной окружения позволяет Diagnostic.Debug записывать 
    // в stderr. В среде браузера этот вывод будет отправлен в консоль. На 
    // данный момент это единственный способ вывести отладочные логи из
    // corlib assemblies.
    monoeg_g_setenv ("DOTNET_DebugWriteToStdErr", "1", 0);
```

## Обновление версии Emscripten в образе Docker

Сначала обновите версию Emscripten в [Webassembly Dockerfile](https://github.com/dotnet/dotnet-buildtools-prereqs-docker/blob/master/src/ubuntu/18.04/webassembly/Dockerfile#L19).

```
ENV EMSCRIPTEN_VERSION=1.39.16
```

Отправьте Pull Request c обновленной версией, дождитесь успешного прохождения всех проверок и слияния запроса. Файл [master.json ](https://github.com/dotnet/versions/blob/master/build-info/docker/image-info.dotnet-dotnet-buildtools-prereqs-docker-master.json#L1126) будет обновлен с новым образом Docker.

```
{
  "platforms": [
    {
      "dockerfile": "src/ubuntu/18.04/webassembly/Dockerfile",
      "simpleTags": [
        "ubuntu-18.04-webassembly-20210707133424-12f133e"
      ],
      "digest": "sha256:1f2d920a70bd8d55bbb329e87c3bd732ef930d64ff288dab4af0aa700c25cfaf",
      "osType": "Linux",
      "osVersion": "Ubuntu 18.04",
      "architecture": "amd64",
      "created": "2020-05-29T22:16:52.5716294Z",
      "commitUrl": "https://github.com/dotnet/dotnet-buildtools-prereqs-docker/blob/6a6da637580ec557fd3708f86291f3ead2422697/src/ubuntu/18.04/webassembly/Dockerfile"
    }
  ]
},
```

Скопируйте тег образа Docker и замените его в файле [platform-matrix.yml](https://github.com/dotnet/runtime/blob/main/eng/pipelines/common/platform-matrix.yml#L172)

```
container:
    image: ubuntu-18.04-webassembly-20210707133424-12f133e
    registry: mcr
```

Создайте PR с новым образом.

# Тестирование библиотек

Больше информации о запуске тестов библиотек см. в [Тестирование библиотек](../../../../src/mono/browser/README.md#libraries-tests).
