# Отключение тестов

Эта документ описывает, как отключать тесты в CI-системе и зачем это нужно.

## Зачем отключать тесты?

Тесты могут быть отключены навсегда или временно:

-   Навсегда: тесты никогда не будут выполняться в определенной конфигурации из-за особенностей его дизайна или продукта;
-   Временно: тест не проходит и его нужно отключить, чтобы не мешать работе CI. Ожидается, что такие тесты будут повторно включены, когда ошибка будет исправлена или необходимая функция реализована.

## Runtime или библиотека?

В репозитории есть два основных набора тестов: тесты runtime в папке `src/tests` и тесты библиотек, которые распределены среди соответствующих библиотек в папке `src/libraries`. (Кроме того, имеются и PAL-тесты в папке `src/coreclr/pal/tests`, но которые не рассматриваются в этом документе.)

Эти два типа имеют разные механизмы отключения.

## Конфигурации тестов

Для начала необходимо определить, для какой конфигурации отключить тест:

-   Для всех конфигураций
-   Только для одной архитектуры процессора (x86, x64, ARM32, ARM64)
-   Для одной версии runtime (Coreclr, Mono) или версии Mono (monointerpreter, llvmaot, llvmfullaot)
-   Только для одной операционной системы (Windows, Linux, MacOS, Android, iOS)
-   Для конкретного типа запуска:
    -   GCStress
    -   JIT стресс-тесты (любого типа)
    -   Тестирование ildasm/ilasm
    -   Тестирование ReadyToRun

Обычно нужно отключать тесты только при конфигурациях, которые вызывают сбои. Таким образом, если тест не проходит только на arm64, не отключайте его для всех архитектур. Если тест не проходит только на macOS, не отключайте его для Windows или Linux.

Если неясно, в каких конфигурациях тест не проходит, целесообразно отключить большее количество тестов, чем требуется обычно.

## Отключение тестов runtime (src/tests)

### Отключение тестов (src/tests) с помощью атрибутов xunit

Тесты runtime используют модель на основе XUnit. Существует [ряд атрибутов для фильтрации](../testing/libraries/filtering-tests.md)
на основе различных режимов тестирования. Ниже представлены некоторые примеры атрибутов, которые можно применить к тестам, чтобы предотвратить их запуск для отдельной конфигурации:

-   Запретить запуск теста для Mono: `[SkipOnMono]`
-   Запретить тест для CoreCLR: `[SkipOnCoreClr]`
-   Запретить тест под GCStress: `[SkipOnCoreClr("Reason", RuntimeTestModes.AnyGCStress)]`
-   Запретить тест под HeapVerify: `[SkipOnCoreClr("Reason", RuntimeTestModes.HeapVerify)]`
-   Запретить стресс-тест под режимами JIT: `[SkipOnCoreClr("Reason", RuntimeTestModes.AnyJitStress)]`

Кроме того, доступны атрибуты `ConditionalFact`, `ConditionalTheory`, `PlatformSpecific` и `ActiveIssue`, которые можно использовать для отключения или включения тестов только на определенных платформах или в определенных конфигурациях.

Некоторые режимы тестирования обрабатываются на уровне сборки. Для этих тестов необходимо пометить тесты как `<RequiresProcessIsolation>true</RequiresProcessIsolation>` и установить один из атрибутов в следующем разделе.

### Отключение тестов (src/tests) с помощью issues.targets

Тесты _вне процесса_ (_out-of-process_) отключаются через добавление теста в соответствующее место и под соответствующим условием конфигурации в файле [issues.targets](https://github.com/vitacore-company/runtime/blob/main/src/tests/issues.targets). Кроме того, тесты с атрибутом `[Fact]`могут быть отключены через `issues.targets`. Все временно отключенные тесты должны содержать ссылку на GitHub issue в элементе `<Issue>`. Отключение теста здесь может зависеть от архитектуры процессора, версии runtime и операционной системы.

### Отключение тестов (src/tests) с помощью свойств конфигурации

Некоторые конфигурации тестов должны быть отключены через редактирование файла `.csproj` или `.ilproj` и вставки свойства в `<PropertyGroup>`, как показано ниже:

-   Запретить запуск теста под GCStress: добавить `<GCStressIncompatible>true</GCStressIncompatible>`
-   Запретить запуск теста при сбоях: добавить `<UnloadabilityIncompatible>true</UnloadabilityIncompatible>`
-   Запретить тест ildasm/ilasm: добавить `<IlasmRoundTripIncompatible>true</IlasmRoundTripIncompatible>`
-   Запретить тест HeapVerify: добавить `<HeapVerifyIncompatible>true</HeapVerifyIncompatible>`
-   Запустить тест под режимами Mono AOT: добавить `<MonoAotIncompatible>true</MonoAotIncompatible>`
-   Запустить тесты под режимами JIT: добавить `<JitOptimizationSensitive>true</JitOptimizationSensitive>`

Эти параметры могут быть указаны с условиями, например:

```
<GCStressIncompatible Condition="'$(TargetArchitecture)' == 'arm64' and '$(TargetOS)' == 'osx'">true</GCStressIncompatible>
```

Больше информации о добавлении тестов для `src/test` см. [в этой главе](../testing/coreclr/test-configuration.md).

## Отключение тестов библиотек (src/libraries)

Информация об отключении тестов на библиотеки представлена [в этой главе](../testing/libraries/filtering-tests.md).

В частности, стоит обратить внимание на `ActiveIssueatTribute`,` skiponcoreclrattribute` и `skiponmonoattribute`.
