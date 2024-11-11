Кросс-компиляция для Android на Linux
======================================

С помощью кросс-компиляции на Linux можно собрать CoreCLR для arm64 Android.

Требования
------------

Нужно сгенерировать набор инструментов и `sysroot` для Android. Для этого существует скрипт, который выполняет все необходимые шаги.

Генерация rootfs
---------------------

Чтобы сгенерировать rootfs, выполните следующую команду в папке `coreclr`:

```
cross/init-android-rootfs.sh
```

Эта команда загрузит NDK и все пакеты, необходимые для компиляции Android на вашей машине. Размер данных более 1 ГБ, поэтому процесс может занять некоторое время.


Кросс-компиляция CoreCLR
------------------------
После генерации rootfs станет доступна кросс-компиляция CoreCLR.

При кросс-компиляции необходимо установить переменные `CONFIG_DIR` и `ROOTFS_DIR`.

Используйте команду ниже, чтобы скомпилировать CoreCLR для arm64:

```
CONFIG_DIR=`realpath cross/android/arm64` ROOTFS_DIR=`realpath cross/android-rootfs/toolchain/arm64/sysroot` ./build.sh cross arm64 cmakeargs -DENABLE_LLDBPLUGIN=0
```

Скомпилированные бинарные файлы будут находиться в папке `artifacts/bin/coreclr/Linux.BuildArch.BuildType/`

Запуск PAL-тестов на Android
--------------------------------

Для запуска PAL-тестов необходимо использовать устройство Android. Также необходимо скопировать PAL-тесты на телефон Android с помощью `adb` и далее запустить их в интерактивной оболочке Android через `adb shell`.

Чтобы скопировать PAL-тесты на телефон Android:
```
adb push artifacts/obj/coreclr/Linux.arm64.Debug/src/pal/tests/palsuite/ /data/local/tmp/coreclr/pal/tests/palsuite
adb push cross/android/toolchain/arm64/sysroot/usr/lib/libandroid-support.so /data/local/tmp/coreclr/lib/
adb push cross/android/toolchain/arm64/sysroot/usr/lib/libandroid-glob.so /data/local/tmp/coreclr/lib/
adb push src/pal/tests/palsuite/paltestlist.txt /data/local/tmp/coreclr
adb push src/pal/tests/palsuite/runpaltests.sh /data/local/tmp/coreclr/
```

Далее используйте `adb shell`, чтобы запустить оболочку на Android. Внутри этой оболочки запустите PAL-тесты, используя команду ниже:
```
LD_LIBRARY_PATH=/data/local/tmp/coreclr/lib ./runpaltests.sh /data/local/tmp/coreclr/
```

Отладка CoreCLR на Android
--------------------------

Для отладки CoreCLR на Android используется удаленный сервер lldb, который запускается с устройства на Android.

Сначала загрузите сервер lldb на Android командой:

```
adb push cross/android/lldb/2.2/android/arm64-v8a/lldb-server /data/local/tmp/
```

Далее запустите сервер lldb на устройстве Android. Откройте оболочку через `adb shell` и выполните команды:

```
adb shell
cd /data/local/tmp
./lldb-server platform --listen *:1234
```

После этого нужно перенаправить порт 1234 с вашего устройства Android на ваш ПК:
```
adb forward tcp:1234 tcp:1234
```

Установите lldb на своем ПК и подключитесь к серверу отладки, который работает на вашем устройстве Android:

```
lldb-3.9
(lldb) platform select remote-android
  Platform: remote-android
 Connected: no
(lldb) platform connect connect://localhost:1234
  Platform: remote-android
    Triple: aarch64-*-linux-android
OS Version: 23.0.0 (3.10.84-perf-gf38969a)
    Kernel: #1 SMP PREEMPT Fri Sep 16 11:29:29 2016
  Hostname: localhost
 Connected: yes
WorkingDir: /data/local/tmp

(lldb) target create coreclr/pal/tests/palsuite/file_io/CopyFileA/test4/paltest_copyfilea_test4
(lldb) env LD_LIBRARY_PATH=/data/local/tmp/coreclr/lib
(lldb) run
```
