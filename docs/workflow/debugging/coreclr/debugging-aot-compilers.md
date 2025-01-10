# Отладка AOT-компиляторов CoreCLR

CoreCLR поставляется с двумя AOT-компиляторами, которые основаны на общей кодовой базе C# — crossgen2 и ilc. Crossgen2 генерирует образы ReadyToRun, которые могут быть загружены в CoreCLR runtime на основе JIT. ILC генерирует объектные файлы для каждой ОС (т.е., COFF на Windows, ELF на Linux и Mach-O на macOS). Такие файлы могут быть связаны с версией NativeAOT для CoreCLR, чтобы создать автономный исполняемый файл или общую библиотеку.

Управляемые AOT-компиляторы приносят с собой ряд трудностей для отладки процесса компиляции. Однако, компиляторы разработаны для оптимизации различных аспектов отладки.

## Важные моменты, о которых следует помнить при отладке управляемых компиляторов

* В отличие от JIT, AOT-компиляторы представляют собой управляемые приложения.
* По умолчанию AOT-компиляторы используют стратегию компиляции с многопоточностью (multi-core compilation strategy).
* В процессе компиляции будет одновременно работать 2 копии JIT: одна используется для компиляции целевого кода, а другая — для создания самого компилятора.
* Компиляторы не анализируют переменные окружения для управления JIT (или любым другим поведением). Все действия контролируются через командную строку.
* Командная строка AOT-компилятора, генерируемая системой проекта, довольно комплексная.

## Встроенные средства отладки в управляемых компиляторах

При отладке многопоточного компонента компилятора и отсутствии необходимости в исследовании самой проблемы многопоточности, рекомендуется отключить использование нескольких потоков. Для этого используйте переключатель `--parallelism 1`, чтобы указать максимальную параллельность процесса равную 1.

При отладке поведения компиляции одного метода компилятор может быть настроен на компиляцию только одного метода. Это делается с помощью различных опций --singlemethod:

* Эти опции работают через поиск конкретного метода по типу, имени, обобщенным аргументам метода, а если этого недостаточно — по индексу. Типы описываются с использованием того же формата, который использует управляемая функция __Type.GetType(string)__. Поскольку этот формат может быть довольно вербозным, компилятор предоставляет переключатель `--print-repro-instructions`, который выведет на консоль аргументы, необходимые для компиляции функции.
* `--singlemethodindex` используется в случаях, когда сигнатура метода является единственным уникальным фактором метода. Индекс используется вместо серии описательных аргументов, так как точное указание сигнатуры является чрезвычайно сложным процессом.
* Аргументы для воспроизведения будут выглядеть следующим образом:  ``--singlemethodtypename "Internal.Runtime.CompilerServices.Unsafe" --singlemethodname As --singlemethodindex 2 --singlemethodgenericarg "System.Runtime.Intrinsics.Vector256`1[[System.SByte]]" --singlemethodgenericarg "System.Runtime.Intrinsics.Vector256`1[[System.Double]]"``

Поскольку компиляторы по умолчанию многопоточные, из результаты выдаются довольно быстро, даже при компиляции в режиме _Debug JIT_. При отладке проблем с JIT рекомендуется использовать _Debug JIT_ независимо от того, какая среда вызвала проблему.

Компиляторы поддерживают произвольное кросс-таргетирование, включая кросс-таргетирование ОС и архитектуры. Однако есть и ограничение: 32-битные архитектуры не могут компилировать для 64-битных архитектур. Кросс-таргетирования позволяет использовать наиболее удобную для разработчика среду отладки. В частности, если возникает проблема между управляемым и нативным кодом, смешанный режим отладки на Windows x64 может помочь отладке.

Если компилятору передан правильный набор сборок/аргументов командной строки, он должен выдавать бинарно идентичный вывод на всех ОС-ях.

Обратите внимание, что компилятор не проверяет ОС/архитектуру, указанную для входных сборок, что позволяет компилировать с использованием версии фреймворка, не соответствующей архитектуре/ОС, для целевой системы. Такой подход может быть не совсем полезен при диагностике всех проблем, но может быть использован для определения общего поведения изменения на всем диапазоне поддерживаемых архитектур.

Управляйте поведением компиляции, используя переключатели `--targetos` и `--targetarch`. По умолчанию используется ОС/архитектура компилятора. Однако, все 64-битные версии компиляторов способны нацеливаться на произвольные комбинации ОС/архитектуры.

На данный момент поддерживаемые наборы допустимых аргументов включают в себя:

| Command line arguments |
| --- |
| `--targetos windows --targetarch x86` |
| `--targetos windows --targetarch x64` |
| `--targetos windows --targetarch arm` |
|`--targetos windows --targetarch arm64` |
|`--targetos linux --targetarch x64` |
|`--targetos linux --targetarch arm` |
|`--targetos linux --targetarch arm64` |
|`--targetos osx --targetarch x64` |
|`--targetos osx --targetarch arm64` |

Передача специальных флагов поведения JIT компилятору осуществляется с помощью переключателя `--codegenopt`. Например, чтобы включить оптимизацию хвостовых вызовов и вывести весь скомпилированный код, используйте комбинацию флагов, таких как `--codegenopt JitDump=* --codegenopt TailCallLoopOpt=1`.

При использовании функции _JitDump JIT_ рекомендуется отключить parellelism, как описано выше, или указать метод, который будет скомпилирован. В противном случае вывод из нескольких функций будет переплетаться и станет неразборчивым.

В процессе отладки в JIT работают два компилятора, поэтому если исходные файлы совпадают, существует высокая вероятность того, что нативный отладчик остановится в нежелательных и неожиданных местах. Чтобы избежать остановок, рекомендуется использовать runtime, который не совпадает с используемым компилятором. Если это невозможно, также можно отключить загрузку символов в большинстве нативных отладчиков. Например, в Visual Studio можно использовать функцию "Specify excluded modules".

Компилятор определяет какой JIT использовать с помощью соглашения о наименовании. По умолчанию он будет использовать JIT, расположенный в той же директории, что и файл crossgen2.dll. Также поддерживается переключатель `--jitpath`, который позволяет использовать конкретный JIT. Эта опция предназначена для поддержки A/B тестирования командой JIT. Опцию -`-jitpath` следует использовать только в том случае, если интерфейс JIT не был изменен. JIT, указанный с помощью переключателя `--jitpath`, должен быть совместим с текущими настройками переключателей `--targetos` и `--targetarch`.

Параллельно проекту crossgen2 существует инструмент под названием _r2rdump_. Этот инструмент можно использовать для вывода содержимого созданного образа ReadyToRun, чтобы исследовать, что именно было сгенерировано в полученном бинарном файле. У инструмента есть множество опций для контроля того, что будет выведено. Инструмент также способен выводить любой образ, созданный с помощью crossgen2 и отображать его содержимое в читаемом формате. Укажите `--disasm`, чтобы отобразить дизассемблированный код.

Если возникает необходимость отладить график зависимостей компилятора (что на данный момент является очень редкой задачей), существует инструмент визуализации, расположенный в папке `src\coreclr\tools\aot\DependencyGraphViewer`. Чтобы использовать этот инструмент, скомпилируйте его и запустите на Windows перед началом компиляции. Он предоставит live-вид графика по мере его генерации и позволит определить, какие ноды находятся в графе и почему. Каждая нода в графике имеет уникальный идентификатор, который виден этому инструменту.  Его также можно использовать параллельно с отладчиком для исследования происходящего в процессе компиляции.

Работа с аргументами при использовании в официальной системе сборки, которые передаваются компилятору, является кропотливой (особенно при работе с директориями ссылок, где каждая сборка указывается индивидуально). Чтобы упростить использование crossgen2 из командной строки вручную, инструмент будет принимать постановочные знаки при разборе ссылок. Обратите внимание, что оболочка Unix может расширить эти аргументы самостоятельно, но это не будет работать корректно. В таких ситуациях заключите аргумент в кавычки, чтобы предотвратить расширение оболочки.

Crossgen2 поддерживает аргументы `--map` и `--mapcsv` для создания карт файлов сгенерированного вывода. Эти файлы в основном используются для диагностики проблем с размером, так как они описывают сгенерированный файл с довольно высокой детализацией, а также предоставляют ряд интересных статистических данных о полученном выводе.

ILC также поддерживает аргумент `--map`, но формат отличается от формата crossgen2, поскольку формат вывода также различен.

Диагностика причин, по которым конкретный метод не удалось скомпилировать в crossgen2, может быть выполнена с помощью передачи переключателя `--verbose` в crossgen2. Это выведет множество сообщений, но в частности, будет указана причина, по которой компиляция была прервана из-за ограничения формата R2R.

Компиляторы могут использовать версию _.NET_, которая используется для сборки продукта (как указано в скрипте dotnet.cmd или dotnet.sh, находящемся в корне репозитория среды выполнения), либо свежую версию _corerun.exe_, полученную в результате сборки тестовой версии. Если используется corerun.exe, настоятельно рекомендуется использовать релизную сборку corerun для этой цели, так как crossgen2 выполняет очень большое количество управляемого кода. В этом случае версия corerun не обязательно должна быть получена из той же сборки, что и crossgen2.dll/ilc.dll. Рекомендуется использовать другое задание для сборки этого corerun, чтобы избежать путаницы.

В тестовой среде runtime каждую проверку можно настроить на компиляцию с помощью crossgen2, используя переменные окружения. Просто установите переменную RunCrossgen2 в 1. При желании установите переменную CompositeBuildMode в 1, если вы хотите увидеть поведение R2R при создании составного образа.

По умолчанию тестовая среда (test bed) runtime будет использовать `dotnet` для запуска управляемого компилятора. Если вы запускаете пакетный скрипт тестов из корня репозитория на Windows, это будет работать без дополнительных настроек. В противном случае вам необходимо установить переменную окружения `__TestDotNetCmd`, чтобы указать на копию dotnet или `corerun`, которая может запускать компилятор. Это самыq простjq способ запустить обычный тест с AOT-компилятором для разработчиков, знакомых с тестовой средой CoreCLR. См. различные техники ниже, которые можно использовать при диагностике проблем в crossgen2.

При попытке собрать crossgen2 необходимо собрать сабсет `clr.tools`. Если вы пересобираете компонент JIT и хотите использовать его в своем внутреннем цикле, вам также нужно собрать с использованием сабсетов `clr.jit` или `clr.alljits`. Если интерфейс JIT изменяется, также необходимо пересобрать сабсет `clr.runtime`.

После завершения сборки продукта функциональная копия crossgen2.dll будет находиться в папке bin: `bin\coreclr\windows.x64.Debug\crossgen2`. После создания тестовой нативной компоновки (test native layout) с помощью команды `src\tests\build generatelayoutonly`, в каталоге `%CORE_ROOT%\crossgen2` также будет находиться копия crossgen2. Версия crossgen2 в каталоге test core_root будет содержать файлы для работы как с x64 dotnet.exe, так и с конечной архитектурой. Это сделано для упрощения кроссплатформенной разработки и предполагает, что основная машина для разработки — x64.

Объектные файлы, сгенерированные компилятором ILC, содержат отладочную информацию для методов и типов в определенном формате (CodeView на Windows, DWARF на других осях). Такие файлы содержат информацию об unwinding в системном формате. Таким образом, исполняемые файлы NativeAOT могут отлаживаться с помощью специфичных отладчиков (VS, WinDbg, GDB, LLDB) без каких-либо расширений (SOS и др). Эти файлы также можно изучить с помощью дизассемблеров и инструментов, работающих с нативным кодом (dumpbin, Ghidra, IDA). Убедитесь, что вы передали аргумент командной строки `-g`, чтобы включить генерацию отладочной информации.

Компилятор ILC обычно компилирует всю программу, что в некоторой степени соответствует составному режиму crossgen2. Существует многопоточный режим, в котором каждая управляемая сборка соответствует одному объектному файлу, но этот режим не используется в производстве.

Поддерживаемые объектные файлы, сгенерированные компилятором ILC, имеют форматы PE/ELF/Mach-O.

## Пример отладки тестового приложения в Crossgen2

Пример ниже демонстрирует отладку простого теста в testbed CoreCLR.

Пример предполагает, что `CORE_ROOT` и `__TestDotNetCmd` установлены корректно. См. комментарии выше для получения подробной информации о `__TestDotNetCmd`.

Тест начинается с установки `RunCrossgen2=1`. Эта команда укажет пакетному скрипту тестов запустить crossgen2 на входных бинарных файлах и также создаст копию входных бинарных файлов, которую необходимо удалить, если вы измените тест. Для этого удалите каталог, в котором находятся тестовые бинарные файлы, и пересоберите тест.

```cmd
C:\git2\runtime>set RunCrossgen2=1

C:\git2\runtime>c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\jit\Directed\Arrays\Complex1\Complex1.cmd
BEGIN EXECUTION
Complex1.dll
        1 file(s) copied.
Could Not Find c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\jit\Directed\Arrays\Complex1\Complex1.dll.rsp
Response file: c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\jit\Directed\Arrays\Complex1\\Complex1.dll.rsp
c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\jit\Directed\Arrays\Complex1\IL-CG2\Complex1.dll
-o:c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\jit\Directed\Arrays\Complex1\\Complex1.dll
--targetarch:x64
--verify-type-and-field-layout
-O
-r:c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\Tests\Core_Root\System.*.dll
-r:c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\Tests\Core_Root\Microsoft.*.dll
-r:c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\Tests\Core_Root\mscorlib.dll
-r:c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\Tests\Core_Root\netstandard.dll
" "dotnet" "c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\Tests\Core_Root\crossgen2\crossgen2.dll" @"c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\jit\Directed\Arrays\Complex1\\Complex1.dll.rsp"   -r:c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\jit\Directed\Arrays\Complex1\IL-CG2\*.dll"
Emitting R2R PE file: c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\jit\Directed\Arrays\Complex1\\Complex1.dll
 "c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\Tests\Core_Root\corerun.exe" Complex1.dll
Starting...
Everything Worked!
Expected: 100
Actual: 100
END EXECUTION - PASSED
PASSED
```

Из этого вызова вы можете увидеть, что crossgen2 был запущен с файлом ответа, содержащим список аргументов, включая все детали для ссылок. Затем вы можете вручную выполнить фактическую команду crossgen2 со значением переменной окружения `__TestDotNetCmd`. Например, сценарий ниже демонстрирует как скопировать и вставить упомянтую команду, а затем запустить ее.

```cmd
C:\git2\runtime>"dotnet" "c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\Tests\Core_Root\crossgen2\crossgen2.dll" @"c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\jit\Directed\Arrays\Complex1\\Complex1.dll.rsp"   -r:c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\jit\Directed\Arrays\Complex1\IL-CG2\*.dll
C:\git2\runtime\.dotnet
Emitting R2R PE file: c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\jit\Directed\Arrays\Complex1\\Complex1.dll
```

Затем в этом сценарии нужно отладить компиляцию отдельного метода, а также запустить его с флагом `-print-repro-instructions`.

```cmd
C:\git2\runtime>"dotnet" "c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\Tests\Core_Root\crossgen2\crossgen2.dll" @"c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\jit\Directed\Arrays\Complex1\\Complex1.dll.rsp"   -r:c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\jit\Directed\Arrays\Complex1\IL-CG2\*.dll --print-repro-instructions
C:\git2\runtime\.dotnet
Single method repro args:--singlemethodtypename "Complex,Complex1" --singlemethodname mul_em --singlemethodindex 1
Single method repro args:--singlemethodtypename "Complex_Array_Test,Complex1" --singlemethodname .ctor --singlemethodindex 1
Single method repro args:--singlemethodtypename "Complex_Array_Test,Complex1" --singlemethodname Main --singlemethodindex 1
Emitting R2R PE file: c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\jit\Directed\Arrays\Complex1\\Complex1.dll
```

Также сценарий демонстрирует как получить больше деталей от JIT. Чтобы уменьшить размер этого примера, в сценарии используется только переключатель `JitOrder=1`. Разработчики JIT обычно будут использовать переключатель `JitDump=*`.

```cmd
C:\git2\runtime>"dotnet" "c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\Tests\Core_Root\crossgen2\crossgen2.dll" @"c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\jit\Directed\Arrays\Complex1\\Complex1.dll.rsp"   -r:c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\jit\Directed\Arrays\Complex1\IL-CG2\*.dll --print-repro-instructions --singlemethodtypename "Complex_Array_Test,Complex1" --singlemethodname Main --singlemethodindex 1 --codegenopt JitOrder=1
C:\git2\runtime\.dotnet
Single method repro args:--singlemethodtypename "Complex_Array_Test,Complex1" --singlemethodname Main --singlemethodindex 1
         |  Profiled   | Method   |   Method has    |   calls   | Num |LclV |AProp| CSE |   Perf  |bytes | x64 codesize|
 mdToken |  CNT |  RGN |    Hash  | EH | FRM | LOOP | NRM | IND | BBs | Cnt | Cnt | Cnt |  Score  |  IL  |   HOT | CLD | method name
---------+------+------+----------+----+-----+------+-----+-----+-----+-----+-----+-----+---------+------+-------+-----+
06000002 |      |      | f656934b |    | rsp | LOOP |   3 |   0 |  35 |  56 |  48 |   7 |   43056 |  490 |   761 |   0 | Complex_Array_Test:Main(System.String[]):int
Emitting R2R PE file: c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\jit\Directed\Arrays\Complex1\\Complex1.dll
```

Используйте флаги  `--targetarch` и `--targetos `, чтобы указать другую архитектуру для исследований.

```cmd
C:\git2\runtime>"dotnet" "c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\Tests\Core_Root\crossgen2\crossgen2.dll" @"c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\jit\Directed\Arrays\Complex1\\Complex1.dll.rsp"   -r:c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\jit\Directed\Arrays\Complex1\IL-CG2\*.dll --print-repro-instructions --singlemethodtypename "Complex_Array_Test,Complex1" --singlemethodname Main --singlemethodindex 1 --codegenopt JitOrder=1 --targetarch arm64
C:\git2\runtime\.dotnet
Single method repro args:--singlemethodtypename "Complex_Array_Test,Complex1" --singlemethodname Main --singlemethodindex 1
         |  Profiled   | Method   |   Method has    |   calls   | Num |LclV |AProp| CSE |   Perf  |bytes | arm64 codesize|
 mdToken |  CNT |  RGN |    Hash  | EH | FRM | LOOP | NRM | IND | BBs | Cnt | Cnt | Cnt |  Score  |  IL  |   HOT | CLD | method name
---------+------+------+----------+----+-----+------+-----+-----+-----+-----+-----+-----+---------+------+-------+-----+
06000002 |      |      | f656934b |    |  fp | LOOP |   3 |   0 |  35 |  59 |  48 |  10 |   63828 |  490 |  1048 |   0 | Complex_Array_Test:Main(System.String[]):int
Emitting R2R PE file: c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\jit\Directed\Arrays\Complex1\\Complex1.dll
```

Обратите внимание, что единственное отличие в командной строке заключалось в передаче флага `--targetarch arm64`. Поэтому JIT компилирует метод как arm64.

Далее прикрепите отладчик к crossgen2.

Поскольку в этом примере используется `dotnet` в качестве `__TestDotNetCmd`, вам нужно будет отлаживать процесс `c:\git2\runtime\.dotnet\dotnet.exe`.

```cmd
devenv /debugexe C:\git2\runtime\.dotnet\dotnet.exe "c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\Tests\Core_Root\crossgen2\crossgen2.dll" @"c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\jit\Directed\Arrays\Complex1\\Complex1.dll.rsp"   -r:c:\git2\runtime\artifacts\tests\coreclr\windows.x64.Debug\jit\Directed\Arrays\Complex1\IL-CG2\*.dll --print-repro-instructions --singlemethodtypename "Complex_Array_Test,Complex1" --singlemethodname Main --singlemethodindex 1 --codegenopt JitOrder=1 --targetarch arm64
```

Далее запустится отладчик Visual Studio с решением, настроенным для отладки процесса dotnet.exe. По умолчанию это решение будет отлаживать только нативный код процесса. Чтобы отлаживать управляемые компоненты, отредактируйте свойства решения и установите  `Debugger Type` на `Managed (.NET Core, .NET 5+)` или `Mixed (.NET Core, .NET 5+)`.

## Отладка графа компиляции

AOT-компиляция управляется графиком зависимостей. Если вам нужно устранить неполадки в графике зависимостей (чтобы выяснить, почему что-то было или не было сгенерировано), следуйте [этому руководству](debugging-compiler-dependency-analysis.md).
