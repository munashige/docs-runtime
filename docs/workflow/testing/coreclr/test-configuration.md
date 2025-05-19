# Общая инфраструктура тестирования

## Тип теста ("Kind")

-   Только сборка (Build Only)
    -   Собирает исполняемый файл.
    -   Не выполняет его.
    -   Пример: `<CLRTestKind>BuildOnly</CLRTestKind>`
-   Только выполнение (Run Only)
    -   Может использовать выходные данные проектов `BuildOnly` или `BuildAndRun` с разными аргументами командной строки.
    -   Пример: `<CLRTestKind>RunOnly</CLRTestKind>`
-   Сборка и выполнение (Build And Run)
    -   Собирает исполняемый файл.
    -   Выполняет собранный файл.
    -   Пример: `<CLRTestKind>BuildAndRun</CLRTestKind>`
-   Общие библиотеки (Shared Libraries)
    -   Для сборки библиотек, общих для нуля или более тестов.
    -   Пример: `<CLRTestKind>SharedLibrary</CLRTestKind>`

По умолчанию (если не указаны аргументы) тип теста — `BuildAndRun`.

## Приоритет

Тестовые случаи категоризируются по уровням приоритета. Наиболее важное подмножество должно быть (и является) самым маленьким. Это подмножество называется приоритет 0.

-   По умолчанию тестовый случай имеет приоритет 0. Тесты должны быть явно понижены в приоритете.
-   Установите приоритет теста, задав свойство `<CLRTestPriority>` в файле проекта теста.
    -   Пример: `<CLRTestPriority>2</CLRTestPriority>`
-   Тесты с более низкими значениями приоритета всегда выполняются вместе с тестами более высокого приоритета.
    -   Т.е. если разработчик выбирает запуск тестов с приоритетом 2, то выполняются все тесты с приоритетами 0, 1 и 2.

## Рекомендации по добавлению тестов

-   Все исходные файлы тестов должны содержать следующий заголовок:
    ```
        // Licensed to the .NET Foundation under one or more agreements.
        // The .NET Foundation licenses this file to you under the MIT license.
    ```
-   Управляемая часть всех тестов должна иметь возможность собираться на любой платформе.
    Фактически, в CI управляемая часть всех тестов будет собираться на OSX.
    Каждая целевая платформа будет запускать подмножество тестов, собранных на OSX.
    Поэтому управляемая часть каждого теста **не должна содержать**:
    -   Условно компилируемый код, зависящий от целевой платформы.
    -   Условно включаемые файлы, зависящие от целевой платформы.
    -   Условные `<DefineConstants/>`, зависящие от целевой платформы.
-   Отключите сборку и выполнение теста для выбранных целевых платформ, условно установив свойство `<CLRTestTargetUnsupported>`.
    -   Пример: `<CLRTestTargetUnsupported Condition="'$(TargetArchitecture)' == 'arm64'">true</CLRTestTargetUnsupported>`
-   Отключите сборку теста, безусловно установив свойство `<DisableProjectBuild>`.
    -   Пример: `<DisableProjectBuild>true</DisableProjectBuild>`
-   Исключите тест из запусков с GCStress, добавив следующее в csproj:
    -   `<GCStressIncompatible>true</GCStressIncompatible>`
-   Исключите тест из тестирования с HeapVerify, добавив следующее в csproj:
    -   `<HeapVerifyIncompatible>true</HeapVerifyIncompatible>`
-   Исключите тест из запусков с JIT stress, добавив следующее в csproj:
    -   `<JitOptimizationSensitive>true</JitOptimizationSensitive>`
-   Исключите тест из запусков с NativeAOT, добавив следующее в csproj:
    -   `<NativeAotIncompatible>true</NativeAotIncompatible>`
-   Исключите тест из тестирования round-trip через ilasm, добавив следующее в csproj:
    -   `<IlasmRoundTripIncompatible>true</IlasmRoundTripIncompatible>`
-   Исключите тест из тестирования выгружаемости (collectible assemblies):
    -   `<UnloadabilityIncompatible>true</UnloadabilityIncompatible>`
-   Если тест специфичен для тестирования crossgen2 и должен компилироваться таким образом во всех режимах тестирования:
    -   `<AlwaysUseCrossGen2>true</AlwaysUseCrossGen2>`
-   Когда `CrossGenTest` установлен в `false`, этот тест не запускается со стандартной R2R-компиляцией, даже если выполняется R2R-тестовый прогон.
    -   `<CrossGenTest>false</CrossGenTest>`
-   Добавляйте ссылки на NuGet, обновляя следующий [тестовый проект](https://github.com/dotnet/runtime/blob/main/src/tests/Common/test_dependencies/test_dependencies.csproj).
-   Любые типы и методы System.Private.CoreLib, используемые тестами, должны быть доступны для сборки на всех платформах.
    Это означает, что должна быть достаточная реализация для того, чтобы компилятор C# мог найти ссылочные типы и методы. Неподдерживаемые платформы
    должны просто выбрасывать `throw new PlatformNotSupportedException()` в своих заглушках.
-   Обновите список исключений в [tests/issues.targets](https://github.com/dotnet/runtime/blob/main/src/tests/issues.targets), если тест не проходит из-за активной ошибки.

### Создание C# тестового проекта

1. Используйте существующий тест, например `<repo_root>\tests\src\Exceptions\Finalization\Finalizer.csproj`, в качестве шаблона и скопируйте его в новую папку в `<repo_root>\tests\src`.
1. Убедитесь, что свойство `<AssemblyName>` удалено.
    - Если его не удалить, это может вызвать путаницу с тем, как тесты обычно обрабатываются сборной системой.
1. Установите свойства `<CLRTestKind>`/`<CLRTestPriority>`.
1. Добавьте исходные файлы в новый проект.
1. Добавьте тестовые случаи, используя атрибут Xunit `Fact`.

    - Мы используем генератор исходного кода для создания точки входа `Main` для тестовых проектов. Генератор обнаружит все методы, помеченные `Fact`, и вызовет их из сгенерированного `Main`.
    - Альтернативно, `Main` может быть определён пользователем. В случае успеха тест возвращает `100`. О неудаче можно сигнализировать любым значением, кроме `100`.

        Пример:

      ```CSharp
          static public int Main(string[] notUsed)
          {
              try
              {
                  // Тестовый сценарий
              }
              catch (Exception e)
              {
                  Console.WriteLine($"Test Failure: {e}");
                  return 101;
              }

              return 100;
          }
      ```

1. Добавьте зависимости от других проектов, если необходимо.
    - Управляемая ссылка: `<ProjectReference Include="../ManagedDll.csproj" />`
    - Ссылка на CMake: `<CMakeProjectReference Include="../NativeDll/CMakeLists.txt" />`
1. Соберите тест.
1. Следуйте шагам для повторного запуска неудавшегося теста, чтобы проверить новый тест.

### Создание объединённого тестового проекта (merged test runner)

1. Используйте существующий тест, например `<repo_root>\src\tests\JIT\Methodical\Methodical_d1.csproj`, в качестве шаблона.
1. Если ваш новый объединённый тестовый проект содержит МНОГО тестов и выполняется слишком долго под GC Stress, установите `<NumberOfStripesToUseInStress>` в число, например 10, чтобы тест мог завершиться за разумное время.

#### Аргументы командной строки для объединённых тестовых проектов

Если тесты не запускаются вручную через командную строку для воспроизведения проблемы, эти параметры обрабатываются внутренней инфраструктурой тестирования. Однако для локального запуска тестов поддерживается набор стандартных параметров.

`[testFilterString] [-stripe <whichStripe> <totalStripes>]`

`testFilterString` — любая строка, кроме `-stripe`. Единственные поддерживаемые фильтры сегодня — это простые формы, поддерживаемые в 'dotnet test --filter' (подстроки полного имени теста).

Можно использовать либо параметр `-stripe <whichStripe> <totalStripes>`, либо переменную окружения `TEST_HARNESS_STRIPE_TO_EXECUTE` для управления striping. Переменная `TEST_HARNESS_STRIPE_TO_EXECUTE` должна быть установлена в строку вида `.<whichStripe>.<totalStripes>`, если используется. `<whichStripe>` — это 0-базированный индекс полосы, `<totalStripes>` — общее количество полос.
