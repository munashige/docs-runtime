# Инструкция по работе с репозиторием

## Введение

Платформа VitaCore работает на Windows, Linux, macOS и FreeBSD. Каждая операционная система имеет свои требования для корректной работы. Обратите внимание, что не все архитектуры поддерживаются для разработки. Однако, сами сборки могут быть ориентированы на более широкий спектр систем, чем те, которые были упомянуты ранее. То есть имеются два типа систем, которые одновременно задействованы всякий раз, когда вы работаете со сборками в репозитории платформы:

- **Система сборки** : Это машина, на которой вы клонировали репозиторий платформы VitaCore и на которой работают все ваши инструменты сборки. В таблице ниже показаны поддерживаемые комбинации операционных систем и архитектур, а также ссылки на документацию с требованиями для каждой операционной системы. Если вы используете WSL напрямую (т. е. не Docker), то следуйте документации с требованиями для Linux.

| Chip  | Windows  | Linux    | macOS    | FreeBSD  |
| :---: | :------: | :------: | :------: | :------: |
| x64   | &#x2714; | &#x2714; | &#x2714; | &#x2714; |
| x86   | &#x2714; |          |          |          |
| Arm32 |          | &#x2714; |          |          |
| Arm64 | &#x2714; | &#x2714; | &#x2714; |          |
|       | [Требования Windows](requirements/windows-requirements.md) | [Требования Linux](requirements/linux-requirements.md) | [Требования macOS](requirements/macos-requirements.md) | [Требования FreeBSD](requirements/freebsd-requirements.md)

- **Конечная система**: Это машина, для которой вы создаете артефакты. То есть система, на которой вы хотите запускать платформу VitaCore.

Система сборки и конечная система могут представлять одну и ту же систему или разные системы. Первый вариант проще, поскольку вы, вероятно, будете работать на одной и той же машине. Работа с разными системами называется *кросс-компиляцией* (cross-compiling). При кросс-компиляции необходимо следовать различным рабочим процессам для успешного завершения, так как невозможно собрать репозиторий в этих системах напрямую (например, Web Assembly (WASM), Browser, Mobiles). Инструкции по работе с данными процессами будут представлены дальше. 

Имейте в виду, что клонирование полной истории этого репозитория занимает примерно 400-500 МБ сетевого трафика, что приводит к увеличению репозитория до 1-1,5 ГБ. Сборка репозитория может занять примерно 10-20 ГБ места для одной конфигурации ОС в зависимости от собранных частей продукта. Имейте ввиду, что этот размер может увеличиться в будущем.

## Компоненты

Платформа состоит из трех основных компонентов:

- Runtimes (CoreCLR и Mono)
- Библиотеки
- Хосты и установщики

Можно запускать сборки из обычной консоли, из корня репозитория. Sudo и права администратора не требуются.

- Для инструкций по редактированию кода и внесению изменений см. [Редактирование и отладка](editing-and-debugging.md).
- Для инструкций по отладке CoreCLR см. [Отладка CoreCLR](debugging/coreclr/debugging-runtime.md).
- Для инструкций по использованию GitHub Codespaces см. [Использование Codespaces](Codespaces.md).


## Терминология и концепты

Ниже описана важная терминология, с которой следует ознакомиться перед тем, как начать работать с платформой. Для получения дополнительной информации и полного списка сокращений и их значений обратитесь к глоссарию [по этой ссылке](/docs/project/glossary.md).

### Конфигурация сборки

Для работы с репозиторием поддерживаются три конфигурации, которые определяют, как будет вести себя ваша сборка:

- **Debug**: Неоптимизированный код. Утверждения (asserts) включены. Эта конфигурация работает медленнее всего. Обеспечивает лучший опыт для отладки продукта.
- **Checked** (эксклюзивно для CoreCLR): Оптимизированный код. Утверждения включены.
- **Release**: Оптимизированный код. Утверждения выключены. Эта конфигурация обеспечивает максимальную производительность, но может затруднить процесс отладки из-за оптимизации компилятора.

### Компоненты сборки

- **Runtime**: Среда выполнения для управляемого кода. Существуют две различные реализации, написанные на C или C++:
    - *CoreCLR*: Комплексный движок, который изначально появился из .NET Framework. Его исходный код находится в папке [src/coreclr](https://github.com/vitacore-company/runtime/tree/main/src/coreclr).
    - *Mono*: Среда выполнения, которая легче чем CoreCLR. Изначально Mono появился как открытый исходный код для поддержки .NET и C# на не-Windows системах. Благодаря легковесности работает без замедлений и с конфигурацией *Debug*. Исходный код находится в папке [src/mono](https://github.com/vitacore-company/runtime/tree/main/src/mono).

- **CoreLib** *(aka System.Private.CoreLib)*: Наименее управляемая библиотека. Она напрямую связана с runtime, поэтому также должна быть собрана в соответствующей конфигурации (например, сборка *Debug* runtime означает, что CoreLib также должна быть с конфигурацией *Debug*). Подмножество `clr` включает включает в себя как Runtime, так и CoreLib компоненты, поэтому волноваться о работе с одинаковыми конфигурациями не стоит. Однако стоит обратить внимание на особые случаи, когда когда вам может потребоваться собрать компоненты отдельно. Код библиотеки, который не зависит от runtime, можно найти в [src/libraries/System.Private.CoreLib/src](https://github.com/vitacore-company/runtime/tree/main/src/libraries/System.Private.CoreLib/src/README.md).

- **Библиотеки**: Группа DLL-файлов, которая обеспечивают дополнительную функциональность runtime. Библиотеки можно собирать в собственной конфигурации, независимо от того, какая конфигурация используется в runtime. Их исходный код находится в папке [src/libraries](https://github.com/vitacore-company/runtime/tree/main/src/libraries).

## Сборка репозитория

Основной скрипт, который отвечает за большинство билдов — это `build.sh` (или `build.cmd` на Windows), который находится в корневой папке репозитория. Этот скрипт принимает в качестве аргументов подмножества, которые вам могут быть необходимы для сборки. Кроме того, для сборки необходимо указывать и другие параметры, такие как конфигурация, конечная система, архитектура конечной системы и т.д.

**Примечание:** Если вы планируете использовать Docker для работы с репозиторием, ознакомьтесь сначала с документацией [по этой ссылке](using-docker.md).

### General Overview

Running the script (`build.sh`/`build.cmd`) with no arguments will build the whole repo in *Debug* configuration, for the OS and architecture of your machine. A typical dev workflow only one or two components at a time, so it is more efficient to just build those. This is done by means of the `-subset` flag. For example, for CoreCLR, it would be:

```bash
./build.sh -subset clr
```

The main subset values are:

- `Clr`: The full CoreCLR runtime, which consists of the runtime itself and the CoreLib components.
- `Libs`: All the libraries components, excluding their tests. This includes the libraries' native parts, refs, source assemblies, and their packages and test infrastructure.
- `Packs`: The shared framework packs, archives, bundles, installers, and the framework pack tests.
- `Host`: The .NET hosts, packages, hosting libraries, and their tests.
- `Mono`: The Mono runtime and its CoreLib.

Some subsets are subsequently divided into smaller pieces, giving you more flexibility as to what to build/rebuild depending on what you're working on. For a full list of all the supported subsets, run the build script, passing `help` as the argument to the `subset` flag.

It is also possible to build more than one subset under the same command-line. In order to do this, you have to link them together with a `+` sign in the value you're passing to `-subset`. For example, to build both, CoreCLR and Libraries in Release configuration, the command-line would look like this:

```bash
./build.sh -subset clr+libs -configuration Release
```

If you require to use different configurations for different subsets, there are some specific flags you can use:

- `-runtimeConfiguration (-rc)`: The CoreCLR build configuration
- `-librariesConfiguration (-lc)`: The Libraries build configuration
- `-hostConfiguration (-hc)`: The Host build configuration

The behavior of the script is that the general configuration flag `-c` affects all subsets that have not been qualified with a more specific flag, as well as the subsets that don't have a specific flag supported, like `packs`. For example, the following command-line would build the libraries in *Release* mode and the runtime in *Debug* mode:

```bash
./build.sh -subset clr+libs -configuration Release -runtimeConfiguration Debug
```

In this example, the `-lc` flag was not specified, so `-c` qualifies `libs`. In the first example, only `-c` was passed, so it qualifies both, `clr` and `libs`.

As an extra note here, if your first argument to the build script are the subsets, you can omit the `-subset` flag altogether. Additionally, several of the supported flags also include a shorthand version (e.g. `-c` for `-configuration`). Run the script with `-h` or `-help` to get an extensive overview on all the supported flags to customize your build, including their shorthand forms, as well as a wider variety of examples.

**NOTE:** On non-Windows systems, the longhand versions of the flags can be passed with either single `-` or double `--` dashes.

### Get Started on your Platform and Components

Now that you've got the general idea on how to get started, it is important to mention that, while the procedure is very similar among platforms and subsets, each component has its own technicalities and details, as explained in their own specific docs:

**Component Specifics:**

- [CoreCLR](/docs/workflow/building/coreclr/README.md)
- [Libraries](/docs/workflow/building/libraries/README.md)
- [Mono](/docs/workflow/building/mono/README.md)

**NOTE:** *NativeAOT* is part of CoreCLR, but it has its own specifics when it comes to building. We have a separate doc dedicated to it [over here](/docs/workflow/building/coreclr/nativeaot.md).

### General Recommendations

- If you're working with the runtimes, then the usual recommendation is to build everything in *Debug* mode. That said, if you know you won't be debugging the libraries source code but will need them (e.g. for a *Core_Root* build), then building the libraries on *Release* instead will provide a more productive experience.
- The counterpart to the previous point: When you are working in libraries. In this case, it is recommended to build the runtime on *Release* and the libraries on *Debug*.
- If you're working on *CoreLib*, then you probably want to try to get the job done with a *Release* runtime, and fall back to *Debug* if you need to.

## Testing the Repo

Building the components of the repo is just part of the experience. The runtime repo also includes vast test suites you can run to ensure your changes work properly as expected and don't inadvertently break something else. Each component has its own methodologies to run their tests, which are explained in their own specific docs:

- [CoreCLR](/docs/workflow/testing/coreclr/testing.md)
  - [NativeAOT](/docs/workflow/building/coreclr/nativeaot.md#running-tests)
- [Libraries](/docs/workflow/testing/libraries/testing.md)
- [Mono](/docs/workflow/testing/mono/testing.md)

### Performance Analysis

Fixing bugs and adding new features aren't the only things to work on in the runtime repo. We also have to ensure performance is kept as optimal as can be, and that is done through benchmarking and profiling. If you're interested in conducting these kinds of analysis, the following links will show you the usual workflow you can follow:

* [Benchmarking Workflow for dotnet/runtime repository](https://github.com/dotnet/performance/blob/master/docs/benchmarking-workflow-dotnet-runtime.md)
* [Profiling Workflow for dotnet/runtime repository](https://github.com/dotnet/performance/blob/master/docs/profiling-workflow-dotnet-runtime.md)

## Warnings as Errors

The repo build treats warnings as errors, including many code-style warnings. Dealing with warnings when you're in the middle of making changes can be annoying (e.g. unused variable that you plan to use later). To disable treating warnings as errors, set the `TreatWarningsAsErrors` environment variable to `false` before building. This variable will be respected by both the `build.sh`/`build.cmd` root build scripts and builds done with `dotnet build` or Visual Studio. Some people may prefer setting this environment variable globally in their machine settings.

## Submitting a PR

Before submitting a PR, make sure to review the [contribution guidelines](/CONTRIBUTING.md). After you get familiarized with them, please read the [PR guide](/docs/workflow/ci/pr-guide.md) to find more information about tips and conventions around creating a PR, getting it reviewed, and understanding the CI results.

## Triaging Errors in CI

Given the size of the runtime repository, flaky tests are expected to some degree. There are a few mechanisms we use to help with the discoverability of widely impacting issues. We also have a regular procedure that ensures issues get properly tracked and prioritized. You can find more information on [triaging failures in CI](/docs/workflow/ci/failure-analysis.md).
