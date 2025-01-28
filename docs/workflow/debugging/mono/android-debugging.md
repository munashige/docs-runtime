# Отладка проблем с Android Runtime

## Включить verbose-логирование

Если доступен С#-проект `net6.0-android` (или новее), добавьте следующий код в файл проекта:

```xml
 <ItemGroup>
   <AndroidEnvironment Include="AndroidEnv.txt" />
 </ItemGroup>
```

Содержимое файла `AndroidEnv.txt`:

```
debug.mono.log=mono_log_level=debug,mono_log_mask=all
MONO_SDB_ENV_OPTIONS=loglevel=10    # only needed if you're debugging the managed debugger
```

Это включит дополнительное логирование Mono runtime в логах adb. Обычно этого достаточно для диагностики проблем, таких как отсутствующие ассамблеи или другие проблемы с loader.

## Управляемая отладка

Должна работать в Visual Studio (Windows).

## Нативная отладка

Установите Android Studio.

Скачайте пакет символов nupkg, который соответствует используемому runtime-у. Runtime для Android находится в папке, например:
`${DOTNET_ROOT}/packs/Microsoft.NETCore.App.Runtime.Mono.android-x86/6.0.0-rc.1.21451.13`

Символы находятся в пакете с именем:
`Microsoft.NETCore.App.Runtime.Mono.android-x86.6.0.0-rc.1.21451.13.symbols.nupkg`. Распакуйте его в какую-либо папку с помощью `unzip`, а в папке
`runtimes/android-x86/native/` переименуйте файлы `*.so.dbg` в `*.so.so`
(нужно добавить файлы символов в Android Studio, т.к. файловый менеджер может показывать только файлы с расширением `*.so`).

1. Соберите APK как обычно с помощью `dotnet build` (это создаст `AppName-Signed.apk` в выходной папке).
2. Запустите эмулятор Android.
3. Установите приложение на эмулятор с помощью `dotnet build -t:Install`.
4. Откройте APK в Android Studio с помощью "Profile or Debug APK".
5. В окне "Project" выберите папку "cpp", затем `libmonosgen-2.0.so`. Далее дважды щелкните `libmonosgen-2.0.so` в папке `libmonosgen-2.0.so`.
6. В окне "Debug Symbols" нажмите "Add", перейдите к распакованному пакету символов и выберите `libmonosgen-2.0.so.so`.
7. В разделе "Path Mappings" выберите корневую папку и добавьте локальный путь к git-репозиторию release/6.0, соответствующему пакету. Нажмите "Apply Changes".
8. Запустите эмулятор или подключите устройство.
9. В меню выберите `Run > Edit Configurations...` и на вкладке "Debugger" убедитесь, что "Debug Type" установлен на что-то, кроме "Java Only".
10. Начните отладку.
11. Теперь у вас должны быть имена функций, локальные переменные, а также возможность пошагово проходить через код на C.

Поскольку вы отлаживаете оптимизированную сборку, возможно, отладчик не сможет отобразить все локальные переменные.

## Работа с локальной отладочной сборкой Mono

Убедитесь, что выполнены предварительные условия для [Тестирования на Android](../../testing/libraries/testing-android.md#prerequisites).

Соберите runtime для вашей архитектуры Android `<ANDROID_ARCH>` и сохраните отладочные символы в бинарном файле:

`./build.sh -s mono+libs -os android -arch <ANDROID_ARCH> -c Debug /p:KeepNativeSymbols=true`

В исходном коде проекта C# добавьте следующее в .csproj (замените `<RUNTIME_GIT_ROOT>` на соответствующий путь и `<ANDROID_ARCH>` на архитектуру Android, для которой вы собирали):

```
  <Target Name="UpdateRuntimePack"
            AfterTargets="ResolveFrameworkReferences">
      <ItemGroup>
        <ResolvedRuntimePack PackageDirectory="<RUNTIME_GIT_ROOT>/artifacts/bin/microsoft.netcore.app.runtime.android-<ANDROID_ARCH>/Debug"
                             Condition="'%(ResolvedRuntimePack.FrameworkName)' == 'Microsoft.NETCore.App'" />
      </ItemGroup>
  </Target>
```

Затем пересоберите и переустановите проект, откройте APK в Android Studio (File > Profile or Debug APK) и начните отладку.

Примечание: Если отладка в Android Studio останавливается на сигналах SIGPWR и SIGXCPU во время запуска, настройте LLDB так, чтобы он не останавливал процесс для этих сигналов, выполнив команды `process handle -p true -s false -n true SIGPWR` и `process handle -p true -s false -n true SIGXCPU` на вкладке LLDB в Android Studio.

## Нативная и управляемая отладка или отладка управляемого отладчика

Этот рабочий процесс полезен для поиска проблем в самом отладчике или для отладки с использованием смеси C и C#.

Установите [sdb](https://github.com/mono/sdb).

Запустите `sdb` и настройте прослушивание `listen 127.0.0.1 5000` (номер порта на ваше усмотрение).

Выполните следующую команду `adb`, чтобы настроить Mono-приложения на подключение к отладчику при запуске:

```
$ adb shell setprop debug.mono.extra "debug=10.0.2.2:5000,loglevel=10"
```

(`loglevel=10` будет выводить сообщения протокола отладчика в логах adb. Если вы не отлаживаете отладчик, этот параметр можно опустить. Для других параметров отладчика см. [`print_usage()`](https://github.com/dotnet/runtime/blob/main/src/mono/mono/component/debugger-agent.c#L573) в `src/mono/mono/component/debugger-agent.c`)

Теперь запустите приложение из Android Studio. Оно должно запуститься и подключиться к отладчику.
