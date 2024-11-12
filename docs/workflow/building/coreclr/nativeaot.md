# Работа с Native AOT

На данный момент набор инструментов Native AOT может быть собран на Linux (x64/arm64), macOS (x64) и Windows (x64/arm64).

## Сборка

1. [Установите предварительные требования](../README.md#build-requirements).
2. Запустите `build[.cmd|.sh] clr.aot+libs -rc [Debug|Release]` из корневой папки репозитория, чтобы собрать бинарные файлы для локальной разработки. Эта команда соберет отдельные компоненты (но не соберет NuGet-пакеты).

### Работа с бинарными файлами

Пути к основным компонентам можно изменить с помощью свойств `IlcToolsPath`, `IlcSdkPath`, `IlcFrameworkPath`, `IlcFrameworkNativePath` и `IlcMibcPath` для `dotnet publish`. Например, команду 
```
/p:IlcToolsPath=<repo root>\artifacts\bin\coreclr\windows.x64.Debug\ilc
``` 
можно использовать для перезаписи компилятора с помощью локальной отладочной сборки для устранения неполадок или быстрого итерационного процесса.

### Сборка пакетов

Запустите скрипт `build[.cmd|.sh] -c Release` из из корневой папки репозитория, чтобы собрать пакеты инструментальной цепочки NativeAOT. Операция сборки поместит пакеты инструментария в папку `artifacts\packages\Release\Shipping`. Для публикации проекта с помощью этих пакетов::

1. Добавьте каталог пакетов в файл `nuget.config`. Например: `<add key="local" value="C:\runtime\artifacts\packages\Release\Shipping" />`
2. Запустите `dotnet add package Microsoft.DotNet.ILCompiler -v 10.0.0-dev` чтобы добавить локальную ссылку на пакеты в ваш проект.
3. Запустите `dotnet publish --packages pkg -r [win-x64|linux-x64|osx-64] -c [Debug|Release]`, чтобы опубликовать ваш проект. Опция `--packages pkg` восстанавливает пакет в локальный каталог, который легко очистить после завершения работы. Что также позволяет избежать загрязнения глобального кэша nuget вашими локально собранным пакетами.

## Верхнеуровневое описание Native AOT

Native AOT — это упрощенная версия CoreCLR runtime, которая специализируется на компиляции ahead-of-time, с сопутствующим AOT-компилятором.

Основные компоненты инструментария включают в себя:

* AOT-компилятор (ILC/ILCompiler) собран на общей кодовой базе с crossgen2 (src/coreclr/tools/aot). Crossgen2 генерирует модули ReadyToRun, которые содержат код и структуры данных для CoreCLR. ILC генерирует код и структуры данных для упрощенной версии CoreCLR в файлы объектов. Эти файлы используют специальные форматы для каждой системы: COFF и CodeView для Windows, ELF и DWARF для Linux, Mach-0 и DWARF для macOS.
* Упрощенный CoreCLR runtime. Файлы NativeAOT находятся в папке `src/coreclr/nativeaot/Runtime`, остальные в `src/coreclr`. Этот упрощенный собирается в статическую библиотеку, которая связывается с объектным файлом. Этот файл генерируется AOT-компилятором с использованием специального компоновщика (`link.exe` на Windows, `ld` на Linux/macOS), чтобы сформировать самостоятельный исполняемый файл.
* Библиотека Bootstrap (`src/coreclr/nativeaot/Bootstrap`). Это небольшая нативная библиотека, которая содержит фактическую нативную точку входа `main()`, инициализирует runtime, а также оперирует управляемым кодом. Строится два варианта бутстраппера — один для исполняемых файлов и другой для динамических библиотек.
* Основные библиотеки (`src/coreclr/nativeaot`): System.Private.CoreLib (corelib), System.Private.Reflection.* (имплементация reflection) и System.Private.TypeLoader (возможность загружать новые типы, которые не были сгенерированы статически).
* Интеграция dotnet (`src/coreclr/nativeaot/BuildIntegration`). Представляет из себя набор файлов .targets/.props, которые подключаются к `dotnet publish`, чтобы запустить AOT-компилятор и компоновщик платформы.

AOT-компилятор принимает приложение, основные библиотеки и библиотеки фреймворка в качестве входных данных. Затем он компилирует всю программу в один объектный файл. После этого привязывается объектный файл, чтобы сформировать исполняемый файл. Исполняемый файл работает самостоятельно (не требует runtime), за исключением любых управляемых DllImports.

Исполняемый файл представлен как нативный исполняемый, поскольку его можно дебажить с помощью нативных отладчиков, а также получить полный доступ к его локальным переменным и информации о пошаговом выполнении.

Компилятор также имеет режим, в котором каждая управляемая сборка может быть скомпилирована в отдельный объектный файл. Далее такие объектные файлы связываются в один исполняемый файл с использованием платформенного компоновщика. Этот режим в основном используется в тестировании, так как процесс сборки в таком режиме проходит быстрее из-за того, что не нужно повторно компилировать один и тот же код (напр. код из CoreLib). Не входит в конфигурацию поставки и имеет ряд проблем (требует точного совпадения настроек компиляции, несовместим с рядом оптимизаций и имеет проблемы с реализацией стандартных виртуальных методов между модулями).

## Решения Visual Studio

В репозитории есть несколько решений Visual Studio (файлы в формате `*.sln`), которые могут быть использованы для редактирования репозитория. Прежде чем начать работу с файлами решений, нужно собрать репозиторий при помощи командной строки. Необходимо также использовать соответствующую конфигурацию, с которой вы собрали репозиторий. По умолчанию `build.cmd` работает с *Debug x64*, поэтому в выпадающих списках конфигурации сборки решения должны быть выбраны `Debug` и `x64`.

Решения включают в себя:

* `src\coreclr\nativeaot\nativeaot.sln`: Решение для библиотек runtime.
* `src\coreclr\tools\aot\ilc.sln`: Решение для компилятора.

Ниже представлен типичный процесс работы с компилятором:

1. Откройте `ilc.sln` в Visual Studio.
2. Укажите проект *ILCompiler* в обозревателе решений (solution explorer) в качестве вашего стартового проекта.
3. В параметрах отладки (debug options) проекта укажите рабочую папку для тестового проекта (например, `C:\test`).
4. В параметрах отладки проекта укажите аргументы приложения для файла ответа (response file), который был сгенерирован при обычной публикации native AOT вашего тестового проекта. Например: `@obj\Release\net8.0\win-x64\native\HelloWorld.ilc.rsp`
5. Начните сборку и запустите проект с помощью **F5**

## Проект "repro" в Visual Studio

Обычно для работы runtime и native AOT требуется скомпилировать небольшой код на C# и запустить его. Репозиторий содержит вспомогательные проекты, которые упрощают отладку AOT-компилятора и runtime.

Рабочий процесс выглядит следующим образом:

1. Соберите репозиторий, используя инструкции по сборке выше.
2. Откройте файл `ilc.sln`, который упомянут выше. Это решение содержит компилятор, а также несвязанный проект под названием "repro". Проект repro — это небольшой Hello World для компилятора. Поместите в него любой кусок кода на C#, который хотите скомпилировать. Проект скомпилирует исходный код в IL, а также сгенерирует файл ответа, который подходит для передачи AOT-компилятору.
3. Убедитесь, что вы установили конфигурацию решения в VS на ту конфигурацию, которую вы только что собрали (напр. *x64 Debug*).
4. Откройте свойства проекта ILCompiler и перейдите во вкладку *Debug*. Далее укажите в Application arguments: `@$(ArtifactsBinDir)repro\$(TargetArchitecture)\$(Configuration)\compile-with-Release-libs.rsp`. Символ `@` в начале аргумента указывает на то, что это путь к файлу ответа, который был сгенерирован при сборке "repro". Замените "*compile-with-Release-libs*" на "*compile-with-Debug-libs*" если собраны соответствующие библиотеки (аргумент `-lc` для `build.cmd`).  Visual Studio расширит путь, например: `@C:\runtime\artifacts\bin\repro\x64\Debug\compile-with-Release-libs.rsp`.
5. Соберите и запустите ILCompiler при помощи **F5**. Проект *repro* скомпилируется в файл `.obj`. На этом этапе вы можете отлаживать компилятор и создавать точки прерывания (breakpoints).
6. Далее необходимо связать файл `obj` с исполняемым файлом, чтобы запустить результат AOT-компиляции:
- Откройте проект `src\coreclr\tools\aot\ILCompiler\reproNative\reproNative.vcxproj` в Visual Studio. Этот проект предназначен для использования вашего скомпилированного файла `.obj` и связывания этого файла с runtime.
- Установите конфигурацию решения на ту пару, которую вы использовали ранее (например, *x64 Debug*).
- Запустите компиляцию при помощи **F5**. Этот процесс также запустит платформенный компоновщик для связывания файла `obj` с runtime, а также запустит runtime. На этом этапе вы можете отлаживать runtime и различные библиотеки `System.Private`.

## Running tests

If you haven't built the tests yet, run `src\tests\build.cmd nativeaot [Debug|Release] tree nativeaot` on Windows, or `src/tests/build.sh -nativeaot [Debug|Release] -tree:nativeaot` on Linux. This will build the smoke tests only - they usually suffice to ensure the runtime and compiler is in a workable shape. To build all Pri-0 tests, drop the `tree nativeaot` parameter. The `Debug`/`Release` parameter should match the build configuration you used to build the runtime.

To run all the tests that got built, run `src\tests\run.cmd runnativeaottests [Debug|Release]` on Windows, or `src/tests/run.sh --runnativeaottests [Debug|Release]` on Linux. The `Debug`/`Release` flag should match the flag that was passed to `build.cmd` in the previous step.

To build an individual test, follow the instructions for compiling a individual test project located in [Building an Individual Test](/docs/workflow/testing/coreclr/testing.md#building-an-individual-test), but add `/t:BuildNativeAot /p:TestBuildMode=nativeaot` to the build command.

To run an individual test (after it was built), navigate to the `artifacts\tests\coreclr\[windows|linux|osx[.x64.[Debug|Release]\$path_to_test` directory. `$path_to_test` matches the subtree of `src\tests`. You should see a `[.cmd|.sh]` file there. This file is a script that will compile and launch the individual test for you. Before invoking the script, set the following environment variables:

* CORE_ROOT=$repo_root\artifacts\tests\coreclr\[windows|linux|osx].x64.[Debug|Release]\Tests\Core_Root
* CLRCustomTestLauncher=$repo_root\src\tests\Common\scripts\nativeaottest[.cmd|.sh]

`$repo_root` is the root of your clone of the repo.

Sometimes it's handy to be able to rebuild the managed test manually or run the compilation under a debugger. A response file that was used to invoke the ahead of time compiler can be found in `$repo_root\artifacts\tests\coreclr\obj\[windows|linux|osx].x64.[Debug|Release]\Managed`.

For more advanced scenarios, look for at [Building the Tests](/docs/workflow/testing/coreclr/testing.md#building-the-tests) and [Building the Core_Root](../../testing/coreclr/testing.md#building-the-coreroot)

### Running library tests

Build library tests by passing the `libs.tests` subset together with the `/p:TestNativeAot=true` to build the libraries, i.e. `clr.aot+libs+libs.tests /p:TestNativeAot=true` together with the full arguments as specified [above](#building). Then, to run a specific library, go to the tests directory of the library and run the usual command to run tests for the library (see [Running tests for a single library](/docs/workflow/testing/libraries/testing.md#running-tests-for-a-single-library)) but add the `/p:TestNativeAot=true` and the build configuration that was used, i.e. `dotnet.cmd build /t:Test /p:TestNativeAot=true -c Release`.

## Design Documentation

* [ILC Compiler Architecture](/docs/design/coreclr/botr/ilc-architecture.md)
* [Managed Type System](/docs/design/coreclr/botr/managed-type-system.md)

## Native Sanitizers

Using native sanitizers with NativeAOT requires additional care compared to using them with CoreCLR. In addition to passing the `-fsanitize` flag to the command that builds NativeAOT, you must also pass the `EnableNativeSanitizers` MSBuild property to any commands that build projects with a sanitized NativeAOT build to ensure that any sanitizer runtimes are correctly linked with the project.

## Further Reading

If you want to know more about working with _NativeAOT_ in general, you can check out their [more in-depth docs](/src/coreclr/nativeaot/docs/README.md) in the `src/coreclr/nativeaot` subtree.
