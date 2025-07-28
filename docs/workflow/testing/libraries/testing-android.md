# Тестирование библиотек на Android

## Необходимые компоненты

Для запуска тестов должны быть установлены следующие зависимости:

-   OpenJDK
-   Android NDK
-   Android SDK

Установить зависимости можно через терминал или с помощью Android Studio.

### Использование терминала

OpenJDK можно установить в Linux (Ubuntu) с помощью `apt-get`:

```bash
sudo apt-get install openjdk-8-jdk zip unzip
```

Android SDK и NDK могут быть автоматически установлены с помощью следующего скрипта:

```bash
#!/usr/bin/env bash
set -e

NDK_VER=r23c
SDK_VER=9123335_latest
SDK_API_LEVEL=33
SDK_BUILD_TOOLS=33.0.1

if [[ "$OSTYPE" == "darwin"* ]]; then
    HOST_OS=darwin
    HOST_OS_SHORT=mac
    BASHRC=~/.zprofile
else
    HOST_OS=linux
    HOST_OS_SHORT=linux
    BASHRC=~/.bashrc
fi

# скачать Android NDK
export ANDROID_NDK_ROOT=~/android-ndk-${NDK_VER}
curl https://dl.google.com/android/repository/android-ndk-${NDK_VER}-${HOST_OS}.zip -L --output ~/andk.zip
unzip ~/andk.zip -d $(dirname ${ANDROID_NDK_ROOT}) && rm -rf ~/andk.zip

# скачать Android SDK, принять лицензию, установить дополнительные пакеты, включая:
# platform-tools, platforms и build-tools
export ANDROID_SDK_ROOT=~/android-sdk
curl https://dl.google.com/android/repository/commandlinetools-${HOST_OS_SHORT}-${SDK_VER}.zip -L --output ~/asdk.zip
mkdir ${ANDROID_SDK_ROOT} && unzip ~/asdk.zip -d ${ANDROID_SDK_ROOT}/cmdline-tools && rm -rf ~/asdk.zip
yes | ${ANDROID_SDK_ROOT}/cmdline-tools/cmdline-tools/bin/sdkmanager --sdk_root=${ANDROID_SDK_ROOT} --licenses
${ANDROID_SDK_ROOT}/cmdline-tools/cmdline-tools/bin/sdkmanager --sdk_root=${ANDROID_SDK_ROOT} "platform-tools" "platforms;android-${SDK_API_LEVEL}" "build-tools;${SDK_BUILD_TOOLS}"
```

### Использование Android Studio

Android Studio предоставляет удобный интерфейс для:

-   установки всех зависимостей;
-   управления виртуальными Android-устройствами;
-   просмотра логов adb.

## Сборка библиотек и тестов для Android

Перед запуском сборки рекомендуется установить переменные окружения для Android SDK и NDK:

```
export ANDROID_SDK_ROOT=<PATH-TO-ANDROID-SDK>
export ANDROID_NDK_ROOT=<PATH-TO-ANDROID-NDK>
```

Таким образом, все должно быть готово к сборке на Android:

```
./build.sh mono+libs -os android -arch x64
```

И также для поочередного запуска тестов для каждой библиотеки:

```
./build.sh libs.tests -os android -arch x64 -test
```

Убедитесь, что эмулятор запущен (см. [`AVD Manager`](#avd-manager)) или устройство подключено и разблокировано.
`AVD Manager` по умолчанию рекомендует устанавливать образы `x86`, поэтому убедитесь, что в скрипте сборки использован параметр `-arch x86`.

### Запуск отдельных тестовых наборов

Ниже показано, как запускать тесты для конкретной библиотеки:

```
./dotnet.sh build /t:Test src/libraries/System.Numerics.Vectors/tests /p:TargetOS=android /p:TargetArchitecture=x64
```

### Запуск функциональных тестов

Доступны [функциональные тесты](https://github.com/dotnet/runtime/tree/main/src/tests/FunctionalTests/), предназначенные для проверки определенных функций/конфигураций/режимов на целевой мобильной платформе.

Функциональные тесты запускаются так же, как и любые другие тестовые наборы библиотек, например:

```
./dotnet.sh build /t:Test -c Release /p:TargetOS=android /p:TargetArchitecture=x64 src/tests/FunctionalTests/Android/Device_Emulator/PInvoke/Android.Device_Emulator.PInvoke.Test.csproj
```

В настоящее время успешное выполнение функциональных тестов ожидает возврат кода `42` - учитывайте это при добавлении новых тестов.

### Тестирование различных конфигураций

Возможно тестирование различных конфигураций путем установки комбинации дополнительных свойств MSBuild, таких как `RunAOTCompilation`, `MonoForceInterpreter` и другие.

1. AOT  
   Для сборки в режиме только AOT добавьте `/p:RunAOTCompilation=true /p:MonoForceInterpreter=false` в команду сборки.

2. AOT-LLVM  
   Для сборки в режиме AOT-LLVM добавьте `/p:RunAOTCompilation=true /p:MonoForceInterpreter=false /p:MonoEnableLLVM=true`.

3. Интерпретатор  
   Для сборки в режиме интерпретатора добавьте `/p:RunAOTCompilation=false /p:MonoForceInterpreter=true`.

### Дизайн тестового приложения

Android-приложение представляет собой [Java Instrumentation](https://github.com/dotnet/runtime/blob/main/src/tasks/AndroidAppBuilder/Templates/MonoRunner.java) и простую Activity, инициализирующую Mono Runtime через JNI. Этот Mono Runtime запускает простой xunit test runner под названием XHarness.TestRunner (см. https://github.com/dotnet/xharness), который выполняет тесты для всех `*.Tests.dll` библиотек в пакете. Также имеется инструмент XHarness.CLI со встроенным ADB для развертывания `*.apk` на целевом устройстве (физическом или эмуляторе) и получения логов после завершения тестов.

### Получение логов

XHarness для Android не выводит много информации и сохраняет результаты тестов в файл. Однако вы можете получать логи в реальном времени с помощью команды:

```
adb logcat -s "DOTNET"
```

Или просто откройте окно `logcat` в Android Studio или Visual Studio.

### AVD Manager

При установленном Android Studio можно использовать [AVD Manager](https://developer.android.com/studio/run/managing-avds) из IDE для создания и запуска виртуальных Android-устройств. В противном случае Android SDK предоставляет [инструмент командной строки avdmanager](https://developer.android.com/studio/command-line/avdmanager).

Пример установки, создания и запуска эмуляторов из командной строки (где `SDK_API_LEVEL` соответствует установленному Android SDK, а `EMULATOR_NAME_X86`/`EMULATOR_NAME_X64` - выбранные вами имена):

```bash
# Установить образ x86
${ANDROID_SDK_ROOT}/cmdline-tools/tools/bin/sdkmanager "system-images;android-${SDK_API_LEVEL};default;x86"

# Создать образ x86
${ANDROID_SDK_ROOT}/cmdline-tools/tools/bin/avdmanager create avd --name ${EMULATOR_NAME_X86} --package "system-images;android-${SDK_API_LEVEL};default;x86"

# Запустить эмулятор с образом x86
${ANDROID_SDK_ROOT}/emulator/emulator -avd ${EMULATOR_NAME_X86} &

# Установить образ x64
${ANDROID_SDK_ROOT}/cmdline-tools/tools/bin/sdkmanager "system-images;android-${SDK_API_LEVEL};default;x86_64"

# Создать образ  x64
${ANDROID_SDK_ROOT}/cmdline-tools/tools/bin/avdmanager create avd --name ${EMULATOR_NAME_X64} --package "system-images;android-${SDK_API_LEVEL};default;x86_64"

# Запустить эмулятор с образом 64
${ANDROID_SDK_ROOT}/emulator/emulator -avd ${EMULATOR_NAME_X64} &
```

Эмулятор можно запускать с различными параметрами. Используйте `emulator -help` для просмотра полного списка опций.

### Существующие ограничения

-   `-os android` пока не поддерживается в Windows (можно использовать `WSL`)
-   XHarness.CLI пока не умеет запускать эмуляторы (необходим запуск через `AVD Manager` или IDE)
-   Режимы AOT и интерпретатора пока не поддерживаются

### Отладка нативного runtime-кода в Android Studio

См. [Отладка на Android](../../debugging/mono/android-debugging.md)
