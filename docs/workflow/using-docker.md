# Работа с Docker

Документация ниже описывает работу использовать образы и контейнеры Docker для ваших.

## Начало работы

Для начала работы необходимо установить Docker Engine. Если Docker Engine не установлен, следуйте инструкциям [на официальном сайте](https://docs.docker.com/get-started/get-docker).

При использовании Docker операционная система вашего компьютера не так важна. Например, при работе с *Ubuntu 22.04*, вы можете без проблем использовать образ *Ubuntu 18.04*. Также можно запускать образы Linux на Windows, если у вас включен WSL.

Инструкции по запуску Docker Engine доступны [по этой ссылке]((https://learn.microsoft.com/windows/wsl/install)). Стоит отметить, что вы не можете запускать несколько ОС на одном Docker Daemon, так как он использует ресурсы из основного ядра по мере необходимости. Поэтому можно запускать либо Linux на WSL, либо контейнеры Windows. Необходимо переключаться между ними вручную , а также перезапускать Docker.

Архитектура конечной системы имеет большее значение при использовании контейнеров Docker. Архитектура образа должна соответствовать системе вашего компьютера. Например, вы можете запускать как x64, так и Arm64 образы на Apple Silicon Mac благодаря эмулятору x64 Rosetta, который он предоставляет. Точно так же вы можете запускать образы Linux Arm32 на хосте Linux Arm64. 

Docker использует WSL для запуска контейнеров Linux на Windows, но вам не нужно запускать терминал WSL, чтобы запускать контейнеры. Достаточно использовать терминал `cmd` или `powershell` с командой `docker`.

## Образы Docker

В таблицах ниже представлены имена образов, требуемая архитектура, ссылки для скачивания и другая информация.

### Основные образы

Основные образы Docker — это наиболее часто используемые образы. При работе с другими сценариями (например, Android или Risc-V), используйте таблицу *Дополнительные образы*.

| Host OS           | Target OS    | Target Arch     | Image                                                                                  | crossrootfs dir      |
| ----------------- | ------------ | --------------- | -------------------------------------------------------------------------------------- | -------------------- |
| Azure Linux (x64) | Alpine 3.13  | x64             | `mcr.microsoft.com/dotnet-buildtools/prereqs:azurelinux-3.0-net9.0-cross-amd64-alpine` | `/crossrootfs/x64`   |
| Azure Linux (x64) | Ubuntu 16.04 | x64             | `mcr.microsoft.com/dotnet-buildtools/prereqs:azurelinux-3.0-net9.0-cross-amd64`        | `/crossrootfs/x64`   |
| Azure Linux (x64) | Alpine 3.13  | Arm32 (armhf)   | `mcr.microsoft.com/dotnet-buildtools/prereqs:azurelinux-3.0-net9.0-cross-arm-alpine`   | `/crossrootfs/arm`   |
| Azure Linux (x64) | Ubuntu 22.04 | Arm32 (armhf)   | `mcr.microsoft.com/dotnet-buildtools/prereqs:azurelinux-3.0-net9.0-cross-arm`          | `/crossrootfs/arm`   |
| Azure Linux (x64) | Alpine 3.13  | Arm64 (arm64v8) | `mcr.microsoft.com/dotnet-buildtools/prereqs:azurelinux-3.0-net9.0-cross-arm64-alpine` | `/crossrootfs/arm64` |
| Azure Linux (x64) | Ubuntu 16.04 | Arm64 (arm64v8) | `mcr.microsoft.com/dotnet-buildtools/prereqs:azurelinux-3.0-net9.0-cross-arm64`        | `/crossrootfs/arm64` |
| Azure Linux (x64) | Ubuntu 16.04 | x86             | `mcr.microsoft.com/dotnet-buildtools/prereqs:azurelinux-3.0-net9.0-cross-x86`          | `/crossrootfs/x86`   |

### Дополнительные образы

| Host OS           | Target OS                  | Target Arch   | Image                                                                                   | crossrootfs dir        |
| ----------------- | -------------------------- | ------------- | --------------------------------------------------------------------------------------- | ---------------------- |
| Azure Linux (x64) | Android Bionic             | x64           | `mcr.microsoft.com/dotnet-buildtools/prereqs:azurelinux-3.0-net9.0-cross-android-amd64` |        *N/A*           |
| Azure Linux (x64) | Android Bionic (w/OpenSSL) | x64           | `mcr.microsoft.com/dotnet-buildtools/prereqs:azurelinux-3.0-net9.0-android-openssl`     |        *N/A*           |
| Azure Linux (x64) | Android Bionic (w/Docker)  | x64           | `mcr.microsoft.com/dotnet-buildtools/prereqs:azurelinux-3.0-net9.0-android-docker`      |        *N/A*           |
| Azure Linux (x64) | Azure Linux 3.0            | x64           | `mcr.microsoft.com/dotnet-buildtools/prereqs:azurelinux-3.0-net9.0-fpm`                 |        *N/A*           |
| Azure Linux (x64) | FreeBSD 13                 | x64           | `mcr.microsoft.com/dotnet-buildtools/prereqs:azurelinux-3.0-net9.0-cross-freebsd-13`    | `/crossrootfs/x64`     |
| Azure Linux (x64) | Ubuntu 18.04               | PPC64le       | `mcr.microsoft.com/dotnet-buildtools/prereqs:azurelinux-3.0-net9.0-cross-ppc64le`       | `/crossrootfs/ppc64le` |
| Azure Linux (x64) | Ubuntu 24.04               | RISC-V        | `mcr.microsoft.com/dotnet-buildtools/prereqs:azurelinux-3.0-net9.0-cross-riscv64`       | `/crossrootfs/riscv64` |
| Azure Linux (x64) | Ubuntu 18.04               | S390x         | `mcr.microsoft.com/dotnet-buildtools/prereqs:azurelinux-3.0-net9.0-cross-s390x`         | `/crossrootfs/s390x`   |
| Azure Linux (x64) | Ubuntu 16.04 (Wasm)        | x64           | `mcr.microsoft.com/dotnet-buildtools/prereqs:azurelinux-3.0-net9.0-webassembly-amd64`   | `/crossrootfs/x64`     |
| Debian (x64)      | Debian 12                  | x64           | `mcr.microsoft.com/dotnet-buildtools/prereqs:debian-12-gcc14-amd64`                     |        *N/A*           |
| Ubuntu (x64)*     | Ubuntu 22.04               | x64           | `mcr.microsoft.com/dotnet-buildtools/prereqs:ubuntu-22.04-debpkg`                       |        *N/A*           |
| Ubuntu (x64)      | Tizen 9.0                  | Arm32 (armel) | `mcr.microsoft.com/dotnet-buildtools/prereqs:ubuntu-22.04-cross-armel-tizen`            | `/crossrootfs/armel`   |
| Ubuntu (x64)      | Ubuntu 20.04               | Arm32 (v6)    | `mcr.microsoft.com/dotnet-buildtools/prereqs:ubuntu-20.04-cross-armv6-raspbian-10`      | `/crossrootfs/armv6`   |

**ПРИМЕЧАНИЕ:** Образы *Ubuntu* со звездочкой (*) используется только для создания deb пакетов, но не могут использоваться для сборки какого-либо продуктового кода.

## Сборка репозитория

Скачайте необходимый образ и используйте команду `docker run` с требуемыми флагами для того, чтобы использовать копию репозитория и запустить скрипты сборки. Ниже представлен небольшой пример команды с описанием каждого использованного флага:

```bash
docker run --rm \
  -v <RUNTIME_REPO_PATH>:/runtime \
  -w /runtime \
  mcr.microsoft.com/dotnet-buildtools/prereqs:azurelinux-3.0-net9.0-cross-amd64 \
  ./build.sh --subset clr --configuration Checked
```

Разберем предоставленный пример:

- `--rm`: Удаляет созданный контейнер после завершения его работы.
- `-v <RUNTIME_REPO_PATH>:/runtime`: Монтирует клон репозитория, расположенный по пути <RUNTIME_REPO_PATH>, в папку `/runtime`.
- `-w /runtime`: Запускает контейнер в папке `/runtime`.
- `mcr.microsoft.com/dotnet-buildtools/prereqs:azurelinux-3.0-net9.0-cross-amd64`: Полное имя Docker-образа для загрузки. В данном случае используется образ Azure Linux для архитектуры x64.
- `./build.sh --subset clr --configuration Checked`: Команда сборки, которую нужно выполнить в репозитории. В данном случае соберется подмножество `Clr` в конфигурации `Checked`.

С контейнером можно взаимодействовать напрямую по ряду причин. Например, для выполнения нескольких команд сборок в разных путях. В этом случае вместо команды сборки можно использовать флаг `-it`. Таким образом, будет получен доступ к оболочке внутри контейнера, что позволит вам исследовать его, выполнять сборки вручную и т.д. - как и в обычном терминале на вашем компьютере. Стоит отметить, что встроенные инструменты оболочки контейнера очень ограничены по сравнению с теми, которые есть на вашем компьютере.

Чтобы выполнить кросс-сборку с использованием Docker, убедитесь, что вы выбрали необходимый образ для системы сборки. Команды сборки представлены [в соответствующей главе](/docs/workflow/building/coreclr/cross-building.md) с тем отличием, что вам не нужно генерировать *ROOTFS*, так как образы для кросс-сборки включают его по умолчанию.