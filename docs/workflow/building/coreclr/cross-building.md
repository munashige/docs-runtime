# Кросс-сборка для разных архитектур и операционных систем

Инструкции ниже демонстрируют как выполнять кросс-сборку на нескольких операционных системах и архитектурах. Стоит отметить, что поддерживаются только те комбинации для кросс-сборки, которые описаны в этой документации. Если будут обнаружены другие комбинации, этот документ будет обновлен соответствующим образом.

## Кросс-сборка на Windows

Этот раздел покрывает кросс-компиляцию на Windows. В настоящее время Windows позволяет выполнять кросс-компиляцию с x64 на практически любую другую архитектуру. 

### Кросс-компиляция для ARM64 на Windows

Для начала работы с кросс-компиляцией ARM64 на Windows, необходимо установить соответствующие инструменты и Windows SDK. Подробные инструкции можно найти в главе [Требования для Windows](../requirements/windows-requirements.md).

После установки требуемых зависимостей, достаточно использовать следующую команду и указать требуемую архитектуру для сборки:

```cmd
.\build.cmd -s clr -c Release -arch arm64
```

### Кросс-компиляция для х86 на Windows

Сборка для x86 не требует установки или настройки дополнительного программного обеспечения, так как все инструменты сборки для x64 также могут выполнять сборку для x86. Достаточно использовать следующий скрипт сборки::

```cmd
.\build.cmd -s clr -c Release -arch x86
```

## Кросс-сборка на macOS

Раздел покрывает кросс-компиляцию на macOS. На данный момент macOS позволяет выполнять кросс-компиляцию между x64 и ARM64.

Нативные инструменты, которые описаны в [Требованиях для macOS](/docs/workflow/requirements/macos-requirements.md), могут быть использованы для кросс-компиляции. Достаточно передать флаг `-cross` и требуемую архитектуру. Например, для сборки ARM64 на Intel x64 Mac:

```bash
./build.sh -s clr -c Release --cross -a arm64
```

## Кросс-сборка на Linux

Раздел посвящен кросс-компиляции на Linux. В настоящее время Linux позволяет выполнять кросс-компиляцию с x64 на ARM32 и ARM64, а также на другие операционные системы на базе Unix, такие как FreeBSD и Alpine.

### Генерация ROOTFS

Для начала работы с кросс-компиляцией на Linux необходимо сгенерировать ROOTFS, который соответствует конечной системе. Скрипт находится в `eng/common/cross/build-rootfs.sh` и отвечает за выполнение кросс-компиляции. Обратите внимание, что этот скрипт необходимо запускать с правами sudo, так как он требует создания некоторых символических ссылок в системе.

Например, чтобы сгенерировать ROOTFS для конечной системы Ubuntu 18 (Bionic) на ARM64:


```bash
sudo ./eng/common/cross/build-rootfs.sh arm64 bionic
```

Бинарные файлы *rootfs* находятся в папке `.tools/rootfs/<arch>`. В этом примере это папка `.tools/rootfs/arm64`. Обратите внимание, что указывать аргумент с кодовым именем Linux (напр. bionic) не обязательно. Если кодовое имя не указано, скрипт выберет значение по умолчанию.

Также можно указать вывод в другом место. Для этого нужно установить переменную окружения `ROOTFS_DIR` в папку, где вы хотите разместить свои бинарные файлы *rootfs*.


#### ROOTFS для FreeBSD

Генерация ROOTFS для кросс-компиляции FreeBSD такая же, как и для других дистрибутивов Linux на других архитектурах. Единственное отличие заключается в том, что нужно указать версию freebsd. Например, для кросс-компиляции x64 для FreeBSD 13:

```bash
sudo ./eng/common/cross/build-rootfs.sh x64 freebsd13
```

### Кросс-компиляция CoreCLR

После генерации _ROOTFS_, убедитесь, что вы установили переменную окружения `ROOTFS_DIR` в папку, где находятся ваши бинарные файлы. Затем запустите сборку в обычном режиме и передайте флаг `--cross`:

```bash
export ROOTFS_DIR=/path/to/runtime/.tools/rootfs/arm64
./build.sh --subset clr --configuration Release --arch arm64 --cross
```

Собранные бинарные файлы находятся в папке `artifacts/bin/coreclr/Linux.<arch>.<configuration>`. Для примера выше папка будет называться `artifacts/bin/coreclr/Linux.arm64.Release`.

#### CoreCLR для FreeBSD

Кросс-сборка для FreeBSD следует тому же процессу, что и для других архитектур. Необходимо использовать флаг `--cross`, а также добавить флаг `--os` и указать FreeBSD. Например:

```bash
export ROOTFS_DIR=/path/to/runtime/.tools/rootfs/x64
./build.sh --subset clr --configuration Release --cross --os freebsd
```

#### Кросс-компиляция CoreCLR для других конфигураций VFP

По умолчанию конфигурация компиляции ARM для CoreCLR — это _armv7-a_ с набором инструкций _thumb-2_, а также _VFPv3 floating point_ c 32 64-битными регистрами FPU.

CoreCLR JIT требует 16 64-битных или 32 32-битных регистров FPU.

В скриптах сборки предоставлен набор параметров конфигурации FPU для поддержки различных типов процессоров. Параметры конфигурации включают в себя: 

* _CLR\_ARM\_FPU\_TYPE_: Соответствует значению, которое передается в опции компилятора `-mfpu`. Cм. документацию вашего выбранного компилятора для получения списка опций.
* _CLR\_ARM\_FPU\_CAPABILITY_: Используется PAL-кодом для определения, в какие регистры FPU должны быть сохранены и восстановлены во время переключений контекста (поддерживаемые опции — 0x3 и 0x7):
  * Bit 0 не используется, всегда установлен на 1.
  * Bit 1 соответствует 16 64-битным регистрам FPU.
  * Bit 2 соответствует 32 64-битным регистрам FPU

Например для поддержки armv7 CPU с VFPv3-d16 нужно использовать следующие параметры компиляции:

```bash
./build.sh --subset clr --configuration Release --cross --arch arm --cmakeargs "-DCLR_ARM_FPU_CAPABILITY=0x3" --cmakeargs "-DCLR_ARM_FPU_TYPE=vfpv3-d16"
```

### Сборка инструментов кросс-таргетинга

Некоторые операции процесса сборки требуют наличия нативных компонентов для архитектуры текущей машины, независимо от архитектуры конечной системы. Эти инструменты называются инструментами для кросс-таргетинга или "_кросс-инструментами_". В настоящее время существует две категории таких инструментов:

* Crossgen2 JIT
* Библиотеки диагностики (Diagnostic Libraries)

Инструменты Crossgen2 JIT используются для запуска Crossgen2 в библиотеках , которые собраны в процессе libraries built during the current build, such as during the `clr.nativecorelib` stage. Under normal circumstances, you should have no need to worry about this, since these tools are automatically built when using the `.\build.cmd` or `./build.sh` scripts at the root of the repo to build any of the CoreCLR native files.

However, you might find yourself needing to (re)build them because either you made changes to them, or you built CoreCLR in a different way using `build-runtime.sh` instead of the usual default script at the root of the repo. To build these tools, you need to run the `src/coreclr/build-runtime.sh` script, and pass the `-hostarch` flag with the architecture of the host machine, alongside the `-component crosscomponents` flag to specify that you only want to build the cross-targeting tools. Retaking our previous example of building for ARM64 using an x64 Linux machine:

```bash
./src/coreclr/build-runtime.sh -arm64 -hostarch x64 -component crosscomponents -cmakeargs "-DCLR_CROSS_COMPONENTS_BUILD=1"
```

The output of running this command is placed in `artifacts/bin/coreclr/linux.<target_arch>.<configuration>/<host_arch>`. For our example, it would be `artifacts/bin/coreclr/linux.arm64.Release/x64`.

On Windows, you can build these cross-targeting diagnostic libraries with the `linuxdac` and `alpinedac` subsets from the root `build.cmd` script. That said, you can also use the `build-runtime.cmd` script, like with Linux. These builds also require you to pass the `-os` flag to specify the target OS. For example:

```cmd
.\src\coreclr\build-runtime.cmd -arm64 -hostarch x64 -os linux -component crosscomponents -cmakeargs "-DCLR_CROSS_COMPONENTS_BUILD=1"
```

If you're building the cross-components in powershell, you'll need to wrap `"-DCLR_CROSS_COMPONENTS_BUILD=1"` with single quotes (`'`) to ensure things are escaped correctly for CMD.

## Cross-Building using Docker

When it comes to building, Docker offers the most flexibility when it comes to targeting different Linux platforms and other similar Unix-based ones, like FreeBSD. This is thanks to the multiple existing Docker images already configured for doing such cross-platform building, and Docker's ease of use of running out of the box on Windows machines with [WSL](https://learn.microsoft.com/windows/wsl/about) enabled, installed, and up and running, as well as Linux machines.

### Cross-Compiling for ARM32 and ARM64 with Docker

As mentioned in the [Linux Cross-Building section](#linux-cross-building), the `ROOTFS_DIR` environment variable has to be set to the _crossrootfs_ location. The prereqs Docker images already have _crossrootfs_ built, so you only need to specify it when creating the Docker container by means of the `-e` flag. These locations are specified in the [Docker Images table](/docs/workflow/building/coreclr/linux-instructions.md#docker-images).

In addition, you also have to specify the `--cross` flag with the target architecture. For example, the following command would create a container to build CoreCLR for Linux ARM64:

```bash
docker run --rm \
  -v <RUNTIME_REPO_PATH>:/runtime \
  -w /runtime \
  -e ROOTFS_DIR=/crossrootfs/arm64 \
  mcr.microsoft.com/dotnet-buildtools/prereqs:cbl-mariner-2.0-cross-arm64 \
  ./build.sh --subset clr --cross --arch arm64
```

### Cross-Compiling for FreeBSD with Docker

Using Docker to cross-build for FreeBSD is very similar to any other Docker Linux build. You only need to use the appropriate image and pass `--os` as well to specify this is not an architecture(-only) build. For example, to make a FreeBSD x64 build:

```bash
docker run --rm \
  -v <RUNTIME_REPO_PATH>:/runtime \
  -w /runtime \
  -e ROOTFS_DIR=/crossrootfs/x64 \
  mcr.microsoft.com/dotnet-buildtools/prereqs:ubuntu-18.04-cross-freebsd-12 \
  ./build.sh --subset clr --cross --os freebsd
```
