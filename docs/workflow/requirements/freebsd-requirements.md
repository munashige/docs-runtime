# Требования для FreeBSD

Ниже представлены требования для сборки и запуска runtime на FreeBSD. Имеется три способа установки и конфигурации, которые перечислены ниже, а также отсортированы от самого простого способа к самому сложному:

- Кросс-компиляция с использованием образов Docker
- Кросс-компиляция на Linux с использованием вашей среды
- Прямая сборка на FreeBSD

## Docker

Установите Docker. Инструкция по установке доступна [по этой ссылке](https://docs.docker.com/install/).

Все необходимые инструменты для сборки включены в образы Docker, используемые для сборки, поэтому дополнительная настройка не требуется.

## Linux

Кросс-сборка FreeBSD в вашей среде Linux требует [установки зависимостей Linux](../linux-requirements). Затем необходимо создать crossrootfs для FreeBSD, что требует установки дополнительных пакетов:

* libbz2-dev
* liblzma-dev
* libarchive-dev
* libbsd-dev

## FreeBSD

Инструкции ниже предполагают, что вы используете `pkg` - стандартный инструмент бинарных пакетов FreeBSD  (аналог `apt`, `apt-get` или `yum` на Linux). Компиляция зависимостей из исходников с использованием дерева портов также может работать, но не тестировалась в рамках написания этой документации.

Требования FreeBSD будут обновлены в скором времени. Инструкции ниже предполагают работу со старыми версиями.

### Установка зависимостей

Сборка репозитория требует установки следующих пакетов:

* Bash
* CMake
* icu
* libunwind
* krb5
* openssl (не обязательно)
* python39
* libinotify
* ninja (не обязательно - является альтернативой make)

```sh
sudo pkg install --yes libunwind icu libinotify lttng-ust krb5 cmake openssl ninja
```

### Запуск на FreeBSD

Установите следующие пакеты:

* icu
* libunwind
* lttng-ust (не обязательно, поддержка отладки)
* krb5
* openssl (не обязательно, поддержка SSL)
* libinotify
* terminfo-db (не обязательно, цвета терминала)

```sh
sudo pkg install --yes libunwind icu libinotify lttng-ust krb5 openssl terminfo-db
```

Извлечение SDK:
По умолчанию SDK располагается тут: `/usr/share/dotnet`

"VERSION" — это версия SDK, которую вы хотите распаковывать.

```sh
sudo mkdir /usr/share/dotnet
tar xf /tmp/dotnet-sdk-VERSION-freebsd-x64.tar.gz -C /usr/share/dotnet/
```

NuGet пакеты:
По умолчанию эти пакеты располагаются тут: `/var/cache/nuget`

"VERSION" — это та же версия, что и SDK выше.

* Microsoft.NETCore.App.Host.freebsd-x64.VERSION.nupkg
* Microsoft.NETCore.App.Runtime.freebsd-x64.VERSION.nupkg
* Microsoft.AspNetCore.App.Runtime.freebsd-x64.VERSION.nupkg

Добавьте следующую строку в любой `NuGet.config` в разделе `<packageSources>`:


```xml
<add key="local" value="/var/cache/nuget" />
```
Добавьте `/usr/share/dotnet` в ваш PATH или создайте символическую ссылку на `/usr/share/dotnet/dotnet`.