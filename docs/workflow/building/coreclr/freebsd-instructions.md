# Cборка CoreCLR на FreeBSD

Инструкция ниже демонстрирует процесс сборки CoreCLR на FreeBSD.

Как указано в главе [Требования для FreeBSD](../../../requirements/freebsd-requirements), есть три способа сборки CoreCLR на FreeBSD:

* Сборка с помощью Docker
* Кросс-компиляция на Linux
* Сборка на FreeBSD

## Сборка с помощью Docker

Сборка с помощью Docker на FreeBSD похожа на работу с Docker на Linux. Поскольку этот процесс включает в себя сценарии кросс-сборки, инструкции по работе с Docker на FreeBSD можно найти в главе [Кросс-сборка](../coreclr/cross-building/).

## Кросс-компиляция на Linux

Установите все зависимости из главы [Требования для Linux](../../../requirements/linux-requirements), а также зависимости, перечисленные в главе [Требования для FreeBSD](../../requirements/freebsd-requirements/).

Далее следуйте инструкции для Linux из главы [Кросс-сборка](../coreclr/cross-building.md). В этой главе представлены подробные инструкции по кросс-компиляции на Linux, включая раздел, посвященный сборке для FreeBSD.

## Сборка на FreeBSD

Установите все зависимости из главы [Требования для FreeBSD](../../requirements/freebsd-requirements/). Инструкции ниже могут устареть для вашего случая, поэтому следите за обновлениями в этой документации.

### Среда

Инструкции ниже предполагают, что вы используете `pkg` (аналог *apt-get* или *yum* для Linux). Установка зависимостей из других источников через древо портов может сработать, но не тестировалась.

Минимально необходимый объем оперативной памяти для сборки — 1 ГБ. Сборка завершается неудачей на виртуальных машинах с 512 МБ  ([Issue 4069](https://github.com/dotnet/runtime/issues/4069)).

#### Настройка инструментария

Установите следующие пакеты для подготовки инструментария:

- bash
- cmake
- llvm37 (включая LLVM 3.7, Clang 3.7 и LLDB 3.7)
- libunwind
- gettext
- icu
- ninja (опционально)
- lttng-ust
- python27

Чтобы установить необходимые пакеты используйте команду:

```sh
janhenke@freebsd-frankfurt:~ % sudo pkg install bash cmake libunwind gettext llvm37 icu
```

Команда выше установит Clang и LLVM 3.7. Для получения информации о сборке CoreCLR с другими версиями смотрите раздел о версиях Clang/LLVM ниже.

### Отладка CoreCLR (опционально)

Примечание: операция ниже не требуется для сборки самого CoreCLR, но требуется, если вы планируете изменять или отлаживать исходный код CoreCLR. Операция должна быть выполнена *перед запуском скрипта сборки*.

Для отладки CoreCLR необходимо установить [LLDB](http://lldb.llvm.org/) - отладчик LLVM.

Для работы с clang 3.7 выполните следующую команду из корневой папки coreclr:

```sh
LLDB_LIB_DIR=/usr/local/llvm37/lib LLDB_INCLUDE_DIR=/usr/local/llvm37/include ./build.sh clang3.7 debug
```

Для запуска тестов используйте:

```sh
./src/pal/tests/palsuite/runpaltests.sh $PWD/artifacts/obj/FreeBSD.x64.Debug $PWD/artifacts/paltestout
```

### Настройка Git

Инструкция предполагает, что вы склонировали репозитории *corefx* и *coreclr* в директории `~/git/corefx` и `~/git/coreclr` на FreeBSD. (`D:\git\corefx` и `D:\git\coreclr` на Windows). Если ваша настройка отличается, необходимо перепроверять команды, которые вы выполняете. Инструкции ниже показывают нужную директорию как на FreeBSD, так и на Windows.

### Сборка Runtime

Чтобы собрать runtime on FreeBSD запустите build.sh из корневой папки репозитория coreclr:

```sh
janhenke@freebsd-frankfurt:~/git/coreclr % ./build.sh
```

Примечание:  Для системы FreeBSD 10.1-RELEASE версия Clang/LLVM — 3.4, минимальная версия для компиляции CoreCLR runtime — 3.5. См. примечание о версиях Clang/LLVM ниже.

Если сборка завершается с ошибками, связанными с компонентами LLVM, предполагаемая версия Clang (3.5) может быть неподходящей для вашей системы. Перепишите версию, используя синтаксис ниже. В этом примере используется LLVM 3.6:

```sh
janhenke@freebsd-frankfurt:~/git/coreclr % ./build.sh clang3.6
```


После завершения сборки должны появиться файлы в папке `artifacts/Product/FreeBSD.x64.Debug`. Наиболее важными являются:

* `corerun`: Хост командной строки. Эта программа запускает CoreCLR runtime и передает ей программу, которую вы хотите запустить.
* `libcoreclr.so`: Сам CoreCLR runtime.
* `libcoreclrpal.so`: Библиотека абстракций платформы (platform abstraction library) для CoreCLR runtime. Присутствует временно, в будущем библиотека будет объединена с `libcoreclr.so`

Рекомендуется создать отдельную папку и скопировать в нее файлы runtime и corerun:

```sh
janhenke@freebsd-frankfurt:~/git/coreclr % mkdir -p ~/coreclr-demo/runtime
janhenke@freebsd-frankfurt:~/git/coreclr % cp artifacts/Product/FreeBSD.x64.Debug/corerun ~/coreclr-demo/runtime
janhenke@freebsd-frankfurt:~/git/coreclr % cp artifacts/Product/FreeBSD.x64.Debug/libcoreclr*.so ~/coreclr-demo/runtime
```

### Сборка нативных компонентов фреймворка

```sh
janhenke@freebsd-frankfurt:~/git/corefx$ ./build-native.sh
janhenke@freebsd-frankfurt:~/git/corefx$ cp artifacts/FreeBSD.x64.Debug/Native/*.so ~/coreclr-demo/runtime
```

### Сборка управляемых компонентов фреймворка

На данный момент нет поддержки сборки кода на FreeBSD, поэтому вам понадобится машина на Windows с клонами как репозитория CoreCLR, так и CoreFX.

Необходимо собрать `System.Private.CoreLib.dll` из репозитория coreclr и остальную часть фреймворка из репозитория corefx. Для сборки `System.Private.CoreLib` (из обычного окна командной строки) выполните:

```
D:\git\coreclr> build.cmd freebsdmscorlib
```

Результаты команды доступен в папке `bin\Product\FreeBSD.x64.Debug\System.Private.CoreLib.dll`.  Скопируйте этот файл в папку runtime на вашей машине с FreeBSD (например, `~/coreclr-demo/runtime`).

Для остальной части фреймворка вам нужно будет передать специальные параметры в скрипт build.cmd при сборке из репозитория CoreFX.

```
D:\git\corefx> build-managed.cmd -os=Linux -target-os=Linux -SkipTests
```

Примечание: необходимо использовать Linux для сборки, так как CoreFX еще не поддерживает FreeBSD.

Также можно добавить `/t:rebuild` к скрипту build.cmd, чтобы принудительно удалить ранее собранные сборки.

Для Hello World-тестирования , нужно скопировать `bin\Linux.AnyCPU.Debug\System.Console\System.Console.dll` и `bin\Linux.AnyCPU.Debug\System.Diagnostics.Debug\System.Diagnostics.Debug.dll` в папку runtime на FreeBSD. (напр. `~/coreclr-demo/runtime`).

После выполнения всех шагов выше папка runtime на FreeBSD должна выглядеть следующим образом:

```
janhenke@freebsd-frankfurt:~/git/coreclr % ls ~/coreclr-demo/runtime/
System.Console.dll  System.Diagnostics.Debug.dll  corerun  libcoreclr.so  libcoreclrpal.so  System.Private.CoreLib.dll
```

### Загрузка зависимостей

Остальные сборки, необходимые для запуска, в настоящее время являются просто заглушками, которые ссылаются на `System.Private.CoreLib`. Эти зависимости можно загрузить через NuGet (на данный момент требует Mono).

Создайте папку для необходимых пакетов:

```sh
janhenke@freebsd-frankfurt:~/git/coreclr % mkdir ~/coreclr-demo/packages
janhenke@freebsd-frankfurt:~/git/coreclr % cd ~/coreclr-demo/packages
```

### Установка Mono

Если в вашей системе еще не установлен Mono, используйте pkg для установки:

```sh
janhenke@freebsd-frankfurt:~/coreclr-demo/packages % sudo pkg install mono
```

### Загрузка клиента NuGet

Скачайте NuGet, если клиент еще не установлен:

```sh
janhenke@freebsd-frankfurt:~/coreclr-demo/packages % curl -L -O https://nuget.org/nuget.exe
```
### Загрузка пакетов NuGet

С Mono и NuGet вы можете использовать NuGet для получения необходимых зависимостей.

Создайте файл `packages.config` с содержимым сниппита ниже. Здесь представлены необходимые зависимости для данного приложения. У разных приложений могут быть и другие зависимости, а также требоваться другой `packages.config` - смотрите [Issue #4053](https://github.com/dotnet/runtime/issues/4053).

```xml
<?xml version="1.0" encoding="utf-8"?>
<packages>
  <package id="System.Console" version="4.0.0-beta-22703" />
  <package id="System.Diagnostics.Contracts" version="4.0.0-beta-22703" />
  <package id="System.Diagnostics.Debug" version="4.0.10-beta-22703" />
  <package id="System.Diagnostics.Tools" version="4.0.0-beta-22703" />
  <package id="System.Globalization" version="4.0.10-beta-22703" />
  <package id="System.IO" version="4.0.10-beta-22703" />
  <package id="System.IO.FileSystem.Primitives" version="4.0.0-beta-22703" />
  <package id="System.Reflection" version="4.0.10-beta-22703" />
  <package id="System.Resources.ResourceManager" version="4.0.0-beta-22703" />
  <package id="System.Runtime" version="4.0.20-beta-22703" />
  <package id="System.Runtime.Extensions" version="4.0.10-beta-22703" />
  <package id="System.Runtime.Handles" version="4.0.0-beta-22703" />
  <package id="System.Runtime.InteropServices" version="4.0.20-beta-22703" />
  <package id="System.Text.Encoding" version="4.0.10-beta-22703" />
  <package id="System.Text.Encoding.Extensions" version="4.0.10-beta-22703" />
  <package id="System.Threading" version="4.0.10-beta-22703" />
  <package id="System.Threading.Tasks" version="4.0.10-beta-22703" />
</packages>

```

Восстановите файл `packages.config`:

```sh
janhenke@freebsd-frankfurt:~/coreclr-demo/packages % mono nuget.exe restore -Source https://www.myget.org/F/dotnet-corefx/ -PackagesDirectory .
```

ПРИМЕЧАНИЕ: Сертификаты CA должны быть установлены по умолчанию. Если у вас возникли проблемы с загрузкой пакетов, см. [Issue #4089](https://github.com/dotnet/runtime/issues/4089#issuecomment-88203778). Команда для FreeBSD:

```sh
janhenke@freebsd-frankfurt:~/coreclr-demo/packages % mozroots --import --sync
```

Наконец, вам нужно скопировать сборки в папку среды выполнения. Однако не копируйте System.Console.dll или System.Diagnostics.Debug, так как версия из NuGet является версией для Windows. Самый простой способ сделать это — использовать команду find:

```sh
janhenke@freebsd-frankfurt:~/coreclr-demo/packages % find . -wholename '*/aspnetcore50/*.dll' -exec cp -n {} ~/coreclr-demo/runtime \;
```

### Сборка приложения

Ниже показано как запустить тестовое Hello World-приложение. За основу взято приложение из corefxlab - оно рисует пингвина Tux (маскота Linux). Можно использовать и свое приложение для запуска.

```sh
janhenke@freebsd-frankfurt:~/coreclr-demo/packages % cd ~/coreclr-demo/runtime
janhenke@freebsd-frankfurt:~/coreclr-demo/runtime % curl -O https://raw.githubusercontent.com/dotnet/corefxlab/master/demos/CoreClrConsoleApplications/HelloWorld/HelloWorld.cs
```

Соберите приложение при помощи `mcs` (компилятора Mono C#). Поскольку нужно скомпилировать приложение с учетом *.NET Core*, вам нужно передать ссылки на ваши восстановленные контрактные сборки (contract assemblies) через NuGet:

```sh
janhenke@freebsd-frankfurt:~/coreclr-demo/runtime % mcs /nostdlib /noconfig /r:../packages/System.Console.4.0.0-beta-22703/lib/contract/System.Console.dll /r:../packages/System.Runtime.4.0.20-beta-22703/lib/contract/System.Runtime.dll HelloWorld.cs
```

### Запуск приложения

Все готово для запуска приложения Hello World! Для  запустите corerun, передав путь папки к exe-файлу и любые другие аргументы. Приложение из corefxlab отобразит чертенка Beastie, если использовать флаг "freebsd":

```sh
janhenke@freebsd-frankfurt:~/coreclr-demo/runtime % ./corerun HelloWorld.exe freebsd
```

Если все работает правильно, вас встретит маскот систем FreeBSD.

Со временем процесс станет проще. Поэтому следите за обновлениями в документации.


### Запуск тестового набора

Если вы внесли изменения в PAL-код  CoreCLR, вам может понадобится запустить PAL-тесты PAL, чтобы проверить работоспособность ваших изменений. Запустить эти тесты можно после чистой сборки, без каких-либо других зависимостей.

Из директории проекта coreclr запустите команду:

```sh
janhenke@freebsd-frankfurt:~/coreclr % ./src/pal/tests/palsuite/runpaltests.sh  ~/coreclr/artifacts/obj/FreeBSD.x64.Debug ~/coreclr/artifacts/paltestout
```

Эта команда запустит все тесты, которые связаны с PAL.

### Примечание о версиях Clang/LLVM

Минимальная версия для сборки CoreCLR — Clang 3.5 или выше.

Релизы FreeBSD 10.X поставляются с Clang 3.4

Если вам необходимо собирть CoreCLR с поддержкой отладки LLDB, выберите llvm37 или llvm-devel.

Для установки clang 3.5: `sudo pkg install clang35`

Для установки clang 3.6: `sudo pkg install clang36`

Для установки clang 3.7: `sudo pkg install llvm37`

Для установки clang development snapshot: `sudo pkg install llvm-devel`

clang35 и clang36 загружают пакеты llvm35 и llvm36 в качестве зависимостей.

llvm37 и llvm-devel включают в себя clang и lldb. Поскольку clang включен в llvm 3.7 и выше, пакетов clang37 не существует.

После установки желаемой версии LLVM вам необходимо уКазать версию в скрипте build.sh.

Например, если нужно установить llvm37, нужно добавить clangX.X к команде сборки как показано ниже:
```sh
janhenke@freebsd-frankfurt:~/git/coreclr % ./build.sh clang3.7
```
