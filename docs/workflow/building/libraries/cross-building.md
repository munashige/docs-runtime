# Кросс-компиляция

## Кросс-компиляция нативных библиотек runtime на Linux

Кросс-компиляция библиотек на Linux возможна для `arm`, `armel`, `arm64` и други хархитектур. Этот процесс во многом схож с кросс-компиляцией CoreCLR.

### Требования

Для хостов на базе Debian нужно установить следующие пакеты:

    $ sudo apt-get install qemu qemu-user-static binfmt-support debootstrap

Кроме того, для кросс-компиляции библиотек требуются утилиты `binutils`. Для `arm` архитектур нужно установить следующий пакет:

    $ sudo apt-get install binutils-arm-linux-gnueabihf

для `armel`:

    $ sudo apt-get install binutils-arm-linux-gnueabi

для `arm64`:

    $ sudo apt-get install binutils-aarch64-linux-gnu

И аналогичные пакеты для других архитектур.

### Генерация rootfs

Скрипт `eng/common/cross/build-rootfs.sh` используется для загрузки файлов, которые необходимы для кросс-компиляции. Этот скрипт генерирует `rootfs` для различных операционных систем и архитектур. Для получения дополнительных сведений см. `eng/common/cross/build-rootfs.sh --help`.

Скрипт `build-rootfs.sh` требует запуска от имени суперпользователя, так как ему нужно создать некоторые символические ссылки в системе. По умолчанию этот скрипт генерирует rootfs в папке `.tools/rootfs/<BuildArch>`. Папку можно изменить, установив переменную окружения `ROOTFS_DIR` или используя флаг `--rootfsdir`.

Например, используйте следующую команду, чтобы сгенерировать rootfs для Ubuntu 18.04 arm- архитектуры:

    $ ./eng/common/cross/build-rootfs.sh arm bionic

Чтобы сгенерировать rootfs в другой папке:

    $ ./build-rootfs.sh arm bionic --rootfsdir /mnt/rootfs/arm

### Компиляция нативных библиотек

Используйте следующую команду, чтобы собрать библиотеки runtime для arm-архитектур:

    $ ROOTFS_DIR=`pwd`/.tools/rootfs/arm ./build.sh libs.native --cross --arch arm --librariesConfiguration Release

Артефакты сборки можно найти в папке `artifacts/bin/native/net10.0-<TargetOS>-<BuildArch>-<BuildType>/`:

    $ ls artifacts/bin/native/net10.0-Linux-Release-arm/*
    artifacts/bin/native/net10.0-Linux-Release-arm/libSystem.Globalization.Native.a
    artifacts/bin/native/net10.0-Linux-Release-arm/libSystem.Globalization.Native.so
    artifacts/bin/native/net10.0-Linux-Release-arm/libSystem.Globalization.Native.so.dbg
    artifacts/bin/native/net10.0-Linux-Release-arm/libSystem.IO.Compression.Native.a
    artifacts/bin/native/net10.0-Linux-Release-arm/libSystem.IO.Compression.Native.so
    artifacts/bin/native/net10.0-Linux-Release-arm/libSystem.IO.Compression.Native.so.dbg
    artifacts/bin/native/net10.0-Linux-Release-arm/libSystem.IO.Ports.Native.a
    artifacts/bin/native/net10.0-Linux-Release-arm/libSystem.IO.Ports.Native.so
    artifacts/bin/native/net10.0-Linux-Release-arm/libSystem.IO.Ports.Native.so.dbg
    artifacts/bin/native/net10.0-Linux-Release-arm/libSystem.Native.a
    artifacts/bin/native/net10.0-Linux-Release-arm/libSystem.Native.so
    artifacts/bin/native/net10.0-Linux-Release-arm/libSystem.Native.so.dbg
    artifacts/bin/native/net10.0-Linux-Release-arm/libSystem.Net.Security.Native.a
    artifacts/bin/native/net10.0-Linux-Release-arm/libSystem.Net.Security.Native.so
    artifacts/bin/native/net10.0-Linux-Release-arm/libSystem.Net.Security.Native.so.dbg
    artifacts/bin/native/net10.0-Linux-Release-arm/libSystem.Security.Cryptography.Native.OpenSsl.a
    artifacts/bin/native/net10.0-Linux-Release-arm/libSystem.Security.Cryptography.Native.OpenSsl.so
    artifacts/bin/native/net10.0-Linux-Release-arm/libSystem.Security.Cryptography.Native.OpenSsl.so.dbg

    $ file artifacts/bin/native/net10.0-linux-release-arm/libSystem.Native.so
    artifacts/bin/native/net10.0-linux-release-arm/libSystem.Native.so: ELF 32-bit LSB shared object, ARM, EABI5 version 1 (SYSV), dynamically linked, BuildID[sha1]=5f6f6f9c4012dffed133624867adf32ac2af130d, stripped

## Компиляция управляемых библиотек на Linux

Компоненты библиотек независимы от версий архитектур и поэтому не требуют кросс-компиляции для `arm`, `armel`, `arm64` и других архитектур. Но для этого требуется отключить`ILLinker` с помощью параметра `/p:ILLinkTrimAssembly=false`).

Большая часть бинарных файлов также не зависит от ОСи (напр. `System.Linq.dll`). Однако некоторые файлы (напр. `System.IO.FileSystem.dll`) зависят и имеют разные версии для Windows и Linux.

Компиляция управляемых библиотек runtime требует наличия собранных нативных библиотек runtime.

Чтобы собрать управляемые библиотеки runtime для `arm` (зависят от архитектуры, не могут быть использованы для других архитектур):

    $ ./build.sh libs.sfx --arch arm --librariesConfiguration Release

Обратите внимание, что `ILLinker` включен по умолчанию. Поэтому эти библиотеки не могут быть использованы на других архитектурах. Чтобы собрать управляемые библиотеки runtime для arm, которые не зависят от архитектуры:

    $ ./build.sh libs.sfx --arch arm --librariesConfiguration Release /p:ILLinkTrimAssembly=false

Артефакты сборки находятся в папке `artifacts/bin/microsoft.netcore.app.runtime.<TargetOS>-<BuildArch>/<BuildType>/runtimes/<TargetOS>-<BuildArch>/lib/net10.0/`. Больше информации о конфигурациях сборке см. [в этой главе](/docs/coding-guidelines/project-guidelines.md).

Нативные и управляемые библиотеки runtime могут быть собраны одновременно с помощью команды:

    $ ROOTFS_DIR=`pwd`/.tools/rootfs/arm ./build.sh --cross --arch arm --librariesConfiguration Release --subset libs.native+libs.sfx
