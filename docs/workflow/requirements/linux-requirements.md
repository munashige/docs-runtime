# Требования для Linux

Существует два способа сборки репозитория runtime на Linux: настроить свою среду на машине с Linux или использовать образы Docker, которые применяются в официальных сборках. Инструкции ниже расписывают оба подхода. Использование Docker позволяет вам воспользоваться предоставленными образами, которые уже имеют настроенную среду, в то время как использование вашей собственной среды предоставляет большую гибкость в наличии других инструментов, которые могут вам понадобиться.

**ПРИМЕЧАНИЕ**: Если вы используете WSL, следуйте инструкциям для дистрибутива, который у вас установлен.


## Использование вашей среды Linux

Разделы ниже описывают требования для разных дистрибутивов Linux. Минимально необходимый объем оперативной памяти составляет 1 ГБ (сборки, как известно, завершаются неудачей на виртуальных машинах с [512 МБ](https://github.com/dotnet/runtime/issues/4069)). Рекомендуется использовать оперативы побольше, чтобы значительно сократить время сборки.

Чтобы начать, вы можете использовать вспомогательный скрипт для установки зависимостей. Или вы можете установить их самостоятельно, следуя инструкциям ниже. Если вы решите попробовать этот скрипт, убедитесь, что вы запускаете его с правами `sudo`:

```bash
sudo eng/install-native-dependencies.sh
```

После использование скрипта рекомендуется перепроверить, что все зависимости были установлены корректно.

### Debian/Ubuntu

Инструкции ниже написаны с учетом версии *Ubuntu LTS*. Ниже перечислены пакеты, которые вам необходимо установить:

- `CMake` (версии 3.20 и выше)
- `llvm`
- `lld`
- `Clang` (см. [Clang для WASM](#clang-for-wasm), если вы планируете работать с *Web Assembly (Wasm)*)
- `build-essential`
- `python-is-python3`
- `curl`
- `git`
- `lldb`
- `libicu-dev`
- `liblttng-ust-dev`
- `libssl-dev`
- `libkrb5-dev`
- `ninja-build` (Установка опциональна. Позволяет собирать нативный код с использованием `ninja` вместо `make`)

```bash
sudo apt install -y cmake llvm lld clang build-essential \
  python-is-python3 curl git lldb libicu-dev liblttng-ust-dev \
  libssl-dev libkrb5-dev ninja-build
```

**ПРИМЕЧАНИЕ**: Если вы используете *Ubuntu* версии старше *22.04 LTS* или *Debian* версии старше 12, не устанавливайте `cmake` напрямую с помощью `apt`. Следуйте инструкциям *CMake на старых версиях Ubuntu и Debian*.

#### CMake на старых версиях Ubuntu и Debian

На момент написания документации в `apt` для Ubuntu доступна версия CMake 3.16.3, если вы используете Ubuntu 20.04 LTS, и версия 3.18.4 в Debian 11. Это ниже требуемой версии 3.20, что делает ее несовместимой с репозиторием runtime. Чтобы обойти это ограничение, у вас есть два варианта: использовать менеджер пакетов snap, который имеет более новую версию CMake, или напрямую использовать APT-репозиторий *Kitware*.

Установить cmake через `snap`:

```bash
sudo snap install cmake
```

Инструкции по установка через *Kitware* доступны [по этой ссылке](https://apt.kitware.com/).

#### Clang for WASM

As of now, *WASM* builds have a minimum requirement of `Clang` version 16 or later (version 18 is the latest at the time of writing this doc). If you're using *Ubuntu 22.04 LTS* or older, then you will have to add an additional repository to `apt` to be able to get said version. Run the following commands on your terminal to do this:

```bash
sudo add-apt-repository -y "deb http://apt.llvm.org/$(lsb_release -s -c)/ llvm-toolchain-$(lsb_release -s -c)-18 main"
sudo apt update -y
sudo apt install -y clang-18
```

You can also take a look at the Linux-based *Dockerfile* [over here](/.devcontainer/Dockerfile) for another example.

#### Additional Tools for Cross Building

If you're planning to use your environment to do Linux cross-building to other architectures (e.g. Arm32, Arm64), and/or other operating systems (e.g. Alpine, FreeBSD), you'll need to install a few additional dependencies. It is worth mentioning these other packages are required to build the `crossrootfs`, which is used to effectively do the cross-compilation, not to build the runtime itself.

- `qemu`
- `qemu-user-static`
- `binfmt-support`
- `debootstrap`

### Fedora

These instructions are written assuming *Fedora 40*.

Install the following packages for the toolchain:

- `cmake`
- `llvm`
- `lld`
- `lldb`
- `clang`
- `python`
- `curl`
- `git`
- `libicu-devel`
- `openssl-devel`
- `krb5-devel`
- `lttng-ust-devel`
- `ninja-build` (Optional. Enables building native code using `ninja` instead of `make`)

```bash
sudo dnf install -y cmake llvm lld lldb clang python curl git \
  libicu-devel openssl-devel krb5-devel lttng-ust-devel ninja-build
```

### Gentoo

In case you have Gentoo you can run following command:

```bash
emerge --ask clang dev-util/lttng-ust app-crypt/mit-krb5
```

## Using Docker

As mentioned at the beginning of this doc, the other method to build the runtime repo for Linux is to use the prebuilt Docker images that our official builds use. In order to be able to run them, you first need to download and install the Docker Engine. The binaries needed and installation instructions can be found at the Docker official site [in this link](https://docs.docker.com/get-started/get-docker).

Once you have the Docker Engine up and running, you can follow our docker building instructions [over here](/docs/workflow/using-docker.md).
