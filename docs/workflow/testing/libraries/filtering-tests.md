# Фильтрация тестов библиотек с атрибутами traits

Тесты могут быть отфильтрованы с помощью атрибутов xunit traits, указанных в библиотеке [`Microsoft.DotNet.XUnitExtensions`](https://github.com/dotnet/arcade/tree/master/src/Microsoft.DotNet.XUnitExtensions).

Некоторые атрибуты принимают аргументы для ограничения фильтрации определённым подмножеством конфигураций, включая платформу, runtime и моникер целевого фреймворка (Target Framework Moniker):

-   `TestPlatforms` указаны [здесь](https://github.com/dotnet/arcade/blob/master/src/Microsoft.DotNet.XUnitExtensions/src/TestPlatforms.cs)
-   `TestRuntimes` (CoreCLR, Mono) указаны [здесь](https://github.com/dotnet/arcade/blob/main/src/Microsoft.DotNet.XUnitExtensions/src/TestRuntimes.cs)
-   `TargetFrameworkMonikers` (Netcoreapp, NetFramework) указаны [здесь](https://github.com/dotnet/arcade/blob/main/src/Microsoft.DotNet.XUnitExtensions/src/TargetFrameworkMonikers.cs)

Эти атрибуты указываются над определением тестового метода. Доступные атрибуты включают:

## OuterLoopAttribute

```cs
[OuterLoop()]
```

Тесты, помеченные атрибутом `OuterLoop`, предназначены для сценариев, которые не требуется запускать при каждой сборке. Они могут выполняться дольше обычных тестов, так как покрывают редко используемые пути выполнения кода или требуют специальной настройки и ресурсов для запуска.

По умолчанию эти тесты исключаются при выполнении `dotnet build`, но их можно включить вручную, добавив параметр: `-testscope outerloop` или `/p:TestScope=outerloop`. Пример:

```cmd
build -test -testscope outerloop
cd src/System.Text.RegularExpressions/tests && dotnet build /t:Test /p:TestScope=outerloop
```

Этот атрибут указан [здесь](https://github.com/dotnet/arcade/blob/main/src/Microsoft.DotNet.XUnitExtensions/src/Attributes/OuterLoopAttribute.cs).

## PlatformSpecificAttribute

```cs
[PlatformSpecific(TestPlatforms platforms)]
```

Используйте этот атрибут для тестовых методов, чтобы указать, что тест должен запускаться только на определённых платформах. Атрибут возвращает следующие категории в зависимости от платформы:

-   `nonwindowstests` - для тестов, которые не выполняются на Windows;
-   `nonlinuxtests` - для тестов, которые не выполняются на Linux;
-   `nonosxtests` - для тестов, которые не выполняются на macOS.

**[Доступные тестовые платформы](https://github.com/dotnet/arcade/blob/master/src/Microsoft.DotNet.XUnitExtensions/src/TestPlatforms.cs)**

При запуске тестов через сборку тестового проекта, тесты, не соответствующие `TargetOS`, не выполняются. Например, для запуска Linux-специфичных тестов на Linux-машине используйте следующую команду:

```sh
dotnet build <csproj_file> /t:Test /p:TargetOS=linux
```

Для запуска всех тестов, совместимых с Linux, а также с категорией failing:

```sh
dotnet build <csproj_file> /t:Test /p:TargetOS=linux /p:WithCategories=failing
```

## ActiveIssueAttribute

Этот атрибут предназначен для использования, когда есть активная проблема, отслеживающая сбой теста, и этот сбой нужно исправить. Это временный атрибут для пропуска теста до устранения проблемы. Важно ограничить область действия атрибута только теми платформами и моникерами целевых фреймворков, где проблема применима.

Атрибут может применяться либо к тестовому классу (отключит все тесты в этом классе), либо к тестовому методу. Он допускает множественное использование на одном и том же члене.

Атрибут также возвращает категорию 'failing', которая по умолчанию отключена.

Атрибут указан [здесь](https://github.com/dotnet/arcade/blob/main/src/Microsoft.DotNet.XUnitExtensions/src/Attributes/ActiveIssueAttribute.cs).

**Отключить для всех платформ и всех фреймворков:**

```cs
[ActiveIssue(string issue)]
```

Пример:

```cs
[ActiveIssue("https://github.com/dotnet/runtime/issues/17845")]
```

**Отключить для конкретной платформы:**

```cs
[ActiveIssue(string issue, TestPlatforms platforms)]
```

Пример:

```cs
[ActiveIssue("https://github.com/dotnet/runtime/issues/67853", TestPlatforms.tvOS)]
[ActiveIssue("https://github.com/dotnet/runtime/issues/52072", TestPlatforms.iOS | TestPlatforms.tvOS | TestPlatforms.MacCatalyst)]
```

**Отключить для runtime:**

```cs
[ActiveIssue(string issue, TestRuntimes runtimes)]
```

Пример:

```cs
[ActiveIssue("https://github.com/dotnet/runtime/issues/2337", TestRuntimes.Mono)]
```

**Отключить для конкретного фреймворка:**

```cs
[ActiveIssue(string issue, TargetFrameworkMonikers frameworks)]
```

Пример:

```cs
[ActiveIssue("https://github.com/dotnet/runtime/issues/26624", TargetFrameworkMonikers.Netcoreapp)]
```

**Отключить для конкретных тестовых платформ и конкретных фреймворков**

```cs
[ActiveIssue(string issue, TestPlatforms platforms, TargetFrameworkMonikers frameworks)]
```

**Отключить при помощи фильтра PlatformDetection:**

```cs
[ActiveIssue(string issue, typeof(PlatformDetection), nameof(PlatformDetection.{member name}))]
```

Пример:

```cs
[ActiveIssue("https://github.com/dotnet/runtimelab/issues/155", typeof(PlatformDetection), nameof(PlatformDetection.IsNativeAot))]
```

Используйте этот атрибут над тестовыми методами, чтобы пропускать падающие тесты только на определенных платформах и определенных целевых фреймворках.

## SkipOnPlatformAttribute

Этот атрибут предназначен для постоянного отключения теста на платформе, где API недоступен или где специально расписано различие в поведении между тестируемой платформой и пропускаемой платформой.

Этот атрибут может быть применен либо к тестовой сборке/классу (что отключит все тесты в этой сборке/классе), либо к тестовому методу. Он допускает множественное использование на одном и том же элементе.

```cs
[SkipOnPlatform(TestPlatforms platforms, string reason)]
```

Пример:

```cs
[SkipOnPlatform(TestPlatforms.Browser, "Credentials is not supported on Browser")]
```

Используйте этот атрибут над тестовыми методами, чтобы пропускать тесты только на определенных целевых платформах. Параметр reason не влияет на traits, но мы всегда используем его, чтобы при виде этого атрибута было понятно, почему тест пропускается на данной платформе.

Если тест нужно пропустить на нескольких платформах по разным причинам, используйте два атрибута на одном тесте, чтобы указать разные причины для каждой платформы.

При добавлении атрибута на всю тестовую сборку рекомендуется также добавить в тестовый файл .csproj:`<IgnoreForCI Condition="'$(TargetOS)' == '...'">true</IgnoreForCI>`

Это позволяет CI-сборке полностью пропустить отправку тестовой сборки в Helix, так как она все равно не запустит тесты.

**В настоящее время поддерживаются следующие [Test Platforms](https://github.com/dotnet/arcade/blob/master/src/Microsoft.DotNet.XUnitExtensions/src/TestPlatforms.cs)**

## SkipOnTargetFrameworkAttribute

Этот атрибут предназначен для постоянного отключения теста на фреймворке, где API недоступно или существует преднамеренное различие в поведении между тестируемым фреймворком и пропускаемым фреймворком.

Атрибут может быть применен либо к тестовому классу (что отключит все тесты в этом классе), либо к тестовому методу. Он допускает множественное использование на одном элементе.

```cs
[SkipOnTargetFramework(TargetFrameworkMonikers frameworks, string reason)]
```

Пример:

```cs
[SkipOnTargetFramework(TargetFrameworkMonikers.NetFramework, ".NET Framework throws a NullReferenceException")]
```

Используйте этот атрибут над тестовыми методами, чтобы пропускать тесты только для определенных целевых фреймворков. Параметр reason не влияет на traits, но мы всегда используем его, чтобы при виде этого атрибута было понятно, почему тест пропускается для данного фреймворка.

Если тест нужно пропустить для нескольких фреймворков по разным причинам, используйте несколько атрибутов на одном тесте, чтобы указать разные причины для каждого фреймворка.

**В настоящее время поддерживаются следующие [Framework Monikers](https://github.com/dotnet/arcade/blob/master/src/Microsoft.DotNet.XUnitExtensions/src/TargetFrameworkMonikers.cs#L23-L26)**

## ConditionalFactAttribute

Используйте этот атрибут для запуска теста только при выполнении условия (когда условие возвращает `true`). Этот атрибут применяется, когда `ActiveIssueAttribute` или `SkipOnTargetFrameworkAttribute` недостаточно гибки из-за необходимости выполнения пользовательской логики во время выполнения теста. Такой тест ведет себя как тест `[Fact]` без передачи тестовых данных в качестве параметра.

```cs
[ConditionalFact(params string[] conditionMemberNames)]
```

Условный метод должен быть статическим методом или свойством, принадлежащим текущему или любому родительскому типу, с любой областью видимости, не принимающим аргументов и возвращающим значение типа `Boolean`.

**Пример:**

```cs
public class TestClass
{
    public static bool ConditionProperty => true;

    [ConditionalFact(nameof(ConditionProperty))]
    public static void TestMethod()
    {
        Assert.True(true);
    }
}
```

## ConditionalTheoryAttribute

Используйте этот атрибут для выполнения теста только при выполнении условия (`true`). Этот атрибут применяется, когда `ActiveIssueAttribute` или `SkipOnTargetFrameworkAttribute` недостаточно гибки из-за необходимости выполнения пользовательской логики во время теста. Такой тест ведет себя как тест `[Theory]`, но без передачи тестовых данных в качестве параметра.

```cs
[ConditionalTheory(params string[] conditionMemberNames)]
```

Этот атрибут должен сопровождаться либо атрибутом `[MemberData(string member)]`, либо атрибутом `[ClassData(Type class)]`, которые представляют `IEnumerable<object>` с данными, передаваемыми в качестве параметров теста. Альтернативный вариант - добавление одного или нескольких атрибутов `[InlineData(object params[] parameters)]`.

Условный метод должен быть статическим методом или свойством, принадлежащим текущему или любому родительскому типу, с любой областью видимости, не принимающим аргументов и возвращающим значение типа `Boolean`.

**Пример:**

```cs
public class TestClass
{
    public static bool ConditionProperty => true;

    public static IEnumerable<object[]> Subtract_TestData()
    {
        yield return new object[] { new IntPtr(42), 6, (long)36 };
        yield return new object[] { new IntPtr(40), 0, (long)40 };
        yield return new object[] { new IntPtr(38), -2, (long)40 };
    }

    [ConditionalTheory(nameof(ConditionProperty))]
    [MemberData(nameof(Subtract_TestData))]
    public static void Subtract(IntPtr ptr, int offset, long expected)
    {
        IntPtr p1 = IntPtr.Subtract(ptr, offset);
        VerifyPointer(p1, expected);

        IntPtr p2 = ptr - offset;
        VerifyPointer(p2, expected);

        IntPtr p3 = ptr;
        p3 -= offset;
        VerifyPointer(p3, expected);
    }
}
```

**Обратите внимание, что все вышеуказанные атрибуты должны включать ссылку на issue и/или содержать комментарий с кратким обоснованием причины. ActiveIssueAttribute и SkipOnTargetFrameworkAttribute должны использовать свои параметры конструктора для этого**

_**Несколько распространенных примеров с вышеуказанными атрибутами:**_

-   Запустить все тесты, которые работают на Windows и не являются падающими:

```cmd
dotnet build <csproj_file> /t:Test /p:TargetOS=windows
```

-   Запустить все тесты категории OuterLoop, которые работают на OS X и в настоящее время связаны с активными issues:

```sh
dotnet build <csproj_file> /t:Test /p:TargetOS=osx /p:WithCategories="OuterLoop;failing""
```

## SkipOnCoreClrAttribute

Этот атрибут используется для отключения теста при определенных условиях, только при запуске с CoreCLR (не влияет на тесты, запускаемые с Mono). Обычно применяется, когда возникает ошибка в конкретной конфигурации запуска теста, например при GCStress или JitStress.

Атрибут может быть применен:

-   К тестовому классу (отключит все тесты в классе)
-   К отдельному тестовому методу
    Допускается множественное использование на одном элементе.

Атрибут определен [здесь](https://github.com/dotnet/arcade/blob/main/src/Microsoft.DotNet.XUnitExtensions/src/Attributes/SkipOnCoreClrAttribute.cs).

**Отключение для всех платформ и всех целевых фреймворков:**

```cs
[SkipOnCoreClr(string reason)]
```

Пример:

```cs
[SkipOnCoreClr("CoreCLR does track thread specific JIT information")]
```

**Отключить для конкретных платформ:**

```cs
[SkipOnCoreClr(string reason, TestPlatforms testPlatforms)]
```

Пример:

```cs
[SkipOnCoreClr("Long running tests: https://github.com/dotnet/runtime/issues/11980", TestPlatforms.Linux)
```

**Отключить для конкретного тестового режима:**

Режим теста - это конфигурация запуска, такая как JitStress, JitStressRegs, JitMinOpts, TailcallStress, GCStress.
`RuntimeTestModes` определен [здесь](https://github.com/dotnet/arcade/blob/main/src/Microsoft.DotNet.XUnitExtensions/src/RuntimeTestModes.cs).

```cs
[SkipOnCoreClr(string reason, RuntimeTestModes testMode)]
```

Пример:

```cs
[SkipOnCoreClr("https://github.com/dotnet/runtime/issues/60240", RuntimeTestModes.JitStressRegs)]
[SkipOnCoreClr("Long running tests: https://github.com/dotnet/runtime/issues/10680", RuntimeTestModes.JitMinOpts)]
```

**Отключить для конкретной конфигурации runtime:**

Конфигурация runtime имеет следующие варианты сборки: Debug, Checked, Release.
`RuntimeConfiguration` определен [здесь](https://github.com/dotnet/arcade/blob/main/src/Microsoft.DotNet.XUnitExtensions/src/RuntimeConfiguration.cs).

```cs
[SkipOnCoreClr(string reason, RuntimeConfiguration runtimeConfigurations)]
```

Пример:

```cs
[SkipOnCoreClr("https://github.com/dotnet/runtime/issues/45464", ~RuntimeConfiguration.Release)]
```

**Отключение для комбинаций параметров:**

Существуют дополнительные сигнатуры атрибутов для комбинаций этих конфигураций, где должны быть выполнены все условия:

```cs
SkipOnCoreClr(string reason, RuntimeConfiguration runtimeConfigurations, RuntimeTestModes testModes)
SkipOnCoreClr(string reason, TestPlatforms testPlatforms, RuntimeConfiguration runtimeConfigurations)
SkipOnCoreClr(string reason, TestPlatforms testPlatforms, RuntimeTestModes testMode)
SkipOnCoreClr(string reason, TestPlatforms testPlatforms, RuntimeConfiguration runtimeConfigurations, RuntimeTestModes testModes)
```

**Отключение с использованием нескольких атрибутов:**

Этот атрибут можно использовать несколько раз - в этом случае тест будет отключен при любом из указанных условий. В данном примере
тест будет выполняться только для Release-сборок, где не установлен `DOTNET_JITMinOpts`.

```cs
[SkipOnCoreClr("https://github.com/dotnet/runtime/issues/67886", ~RuntimeConfiguration.Release)]
[SkipOnCoreClr("https://github.com/dotnet/runtime/issues/67886", RuntimeTestModes.JitMinOpts)]
```

## SkipOnMonoAttribute

Этот атрибут используется для отключения теста только при запуске с Mono.

Атрибут может быть применен к сборке, классу или методу.

Атрибут определен [здесь](https://github.com/dotnet/arcade/blob/main/src/Microsoft.DotNet.XUnitExtensions/src/Attributes/SkipOnMonoAttribute.cs).

**Отключить для всех платформ:**

```cs
[SkipOnMonoAttribute(string reason, TestPlatforms testPlatforms = TestPlatforms.Any)]
```

Пример:

```cs
[SkipOnMono("No SAPI on Mono")]
```

## CollectionAttribute

Это стандартный атрибут xunit, определенный [здесь](https://github.com/xunit/xunit/blob/07663749ab0f62597acc5ff5f163df9f5a0ab8d5/src/xunit.v3.core/CollectionAttribute.cs).

Типичное использование в тестах библиотек выглядит следующим образом:

```cs
[Collection(nameof(DisableParallelization))]
```

Этот атрибут применяется к тестовым классам, чтобы указать, что ни один из тестов в этом классе (которые, как обычно, выполняются последовательно друг относительно друга) не может выполняться параллельно с тестами из другого класса. Это используется для тестов, которые потребляют много дискового пространства или памяти, или загружают все ядра, что может нарушить работу параллельно выполняемых тестов.

## FactAttribute и параметр `Skip`

Еще один способ полностью отключить тест - использовать именованный параметр `Skip` в атрибуте `FactAttribute`.

Пример:

```cs
[Fact(Skip = "<reason for skipping>")]
```

Если причина пропуска теста - ссылка на issue, рекомендуется использовать `ActiveIssueAttribute`. В остальных случаях параметр `Skip` позволяет указать более детализированную причину.
