# Тестирование управляемых инструментов

Для ряда инструментов, включая компилятор для NativeAOT (`ILCompiler`) и триммер(`illink`), предоставляются модульные и функциональные тесты.

## Добавление новых наборов тестов

Чтобы добавить новый набор тестов (test suit), создайте новый `.csproj` с именем, оканчивающимся на _Tests_, например: `MyTool.Tests.csproj`.

Свойство `IsTestProject` будет установлено файлом `Directories.Build.props` в
корне репозитория. Это свойство, в свою очередь, добавит ссылки на пакет xunit и
соответствующий тестовый раннер.

Далее нужно добавить элемент `ProjectToBuild` в `eng/Subsets.props` в один из существующих сабсетов, например `clr.toolstests` или новый сабсет.

## Добавление новых наборов тестов в CI

Чтобы запускать тесты в CI, добавьте новый пайплайн или дополните существующий, например `CLR_Tools_Tests` в `eng/pipelines/runtime.yml`. Обновите условие запуска, например, добавив новый набор путей в `eng/pipelines/common/evaluate-default-paths.yml`, чтобы тесты запускались при изменении исходного кода инструмента
или тестов.

## Локальный запуск тестов

Соберите и запустите тесты локально с помощью одной из следующих команд:

```console
./build.[sh|cmd] -s clr.toolstests -c [Release|Debug] -build -test
```

или

```console
./dotnet.[sh|cmd] test .../MyTool.Tests.csproj -c [Release|Debug]
```

Механизмы фильтрации xunit для `dotnet-tes` позволяют запустить один тест или подмножество тестов:

```console
./dotnet.[sh|cmd] test .../MyTool.Tests.csproj -c [Release|Debug] --filter "FullyQualifiedName~MyTest"
```

Приведённая выше команда запускает все тесты, полное имя которых содержит подстроку `MyTest`. Полный синтаксис см. [на сайте learn.microsoft.com](https://learn.microsoft.com/dotnet/core/testing/selective-unit-tests?pivots=mstest#syntax).
