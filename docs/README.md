# Платформа VitaCore

VitaCore — это это программное обеспечение, представляющее собой
кроссплатформенную площадку, для создания различных типов приложений. VitaCore является частью [.NET Foundation](https://learn.microsoft.com/dotnet/core/) и способствует повышению эффективности разработки за счет использования базовых библиотек для построения моделей приложений. Основными языками программирования, на которых основаны эти модели, являются C++ и C#.

VitaCore состоит из среды выполнения (Runtime) и пакета для разработки
(SDK). Среда выполнения используется для запуска приложения .NET и может быть
включена в приложение. Пакет SDK используется для создания приложений и библиотек
.NET. Среда выполнения .NET всегда устанавливается вместе с пакетом SDK.
Установка пакета SDK включает все три среды выполнения: ASP.NET Core, Desktop и
.NET.

## Установка VitaCore

Инструкция по установке программного обеспечения, ссылки для получения архивов для инсталляции, а также требования к установке доступны [в руководстве по установке VitaCore](https://drive.google.com/file/d/1fxfn0LoQHp54wtgxjSnMeovps8gtttAD/view?usp=sharing).

## Работа с репозиторием

Работа с репозиторием включает в себя разработку, тестирование, бенчмаркинг, профилирование и т.д.
Если вы хотите внести изменения в этом репозитории, ознакомьтесь с этой инструкцией:

-   [Инструкции по рабочему процессу](workflow/README.md)

## Документы по проектированию

Проектирование функций Runtime доступны в [соответствующей главе](design/features/).
Дополнительные документы о проектировании функций доступны в этой ссылке : [github.com/dotnet/designs](https://github.com/dotnet/designs).

### Книга Runtime

[Книга Runtime](design/coreclr/botr/README.md) — это набор глав, которые подробно рассматривают различные аспекты проектирования фреймворка .NET.

Для быстрого старта рекомендуется начать с наиболее используемых глав:

-   [Введение в Common Language Runtime (CLR)](design/coreclr/botr/intro-to-clr.md)
-   [Проектирование сборки мусора (Garbage Collection)](design/coreclr/botr/garbage-collection.md)
-   [Система Типов](design/coreclr/botr/type-system.md)

### Исходный код CoreCLR

Для "глубокого погружения" в исходный код CoreCLR, ознакомьтесь со статьями по [этой ссылке](deep-dive-blog-posts.md).

## Инструкции по разработке

-   [Руководство по работе с CLR](coding-guidelines/clr-code-guide.md)
-   [Стандарты кода CLR JIT](coding-guidelines/clr-jit-coding-conventions.md)
-   [Проектирование кроссплатформенной производительности и событий](coding-guidelines/cross-platform-performance-and-eventing.md)
-   [Добавление новых событий в виртуальную машину](coding-guidelines/EventLogging.md)
-   [Стиль кода C#](coding-guidelines/coding-style.md)
-   [Проектирования фреймворка](coding-guidelines/framework-design-guidelines-digest.md)
-   [Кроссплатформа](coding-guidelines/cross-platform-guidelines.md)
-   [Производительность](coding-guidelines/performance-guidelines.md)
-   [Взаимодействие (Interop)](coding-guidelines/interop-guidelines.md)
-   [Ломающие изменения (Breaking Changes)](coding-guidelines/breaking-changes.md)
-   [Определения ломающих изменений](coding-guidelines/breaking-change-definitions.md)
-   [Правила ломающих изменений](coding-guidelines/breaking-change-rules.md)
-   [Руководство по проекту](coding-guidelines/project-guidelines.md)
-   [Руководство по добавлению API](coding-guidelines/adding-api-guidelines.md)

## Документы проекта

Документы проекта доступны в [соответствующей главе](project/). В будущем будет добавлено больше документов.

## Дополнительная информация

-   [Глоссарий .NET](project/glossary.md)
-   [Энциклопедия имен файлов .NET](project/dotnet-filenames.md)
-   [Портирование .NET Core](https://learn.microsoft.com/dotnet/standard/analyzers/portability-analyzer)
-   [Стандарты (Ecma) .NET](project/dotnet-standards.md)
-   [Конфигурация CLR](../src/coreclr/inc/clrconfigvalues.h)
-   [Обзор CLR](https://learn.microsoft.com/dotnet/standard/clr)
-   [CLR в Википедии](https://ru.wikipedia.org/wiki/Common_Language_Runtime)
