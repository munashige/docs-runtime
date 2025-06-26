# Работа с Benchmarks Driver 2

Этот документ описывает процесс запуска ASP.NET бенчмарков с использованием _crossgen2_ через последнюю версию драйвера и серверов.

## Требования

* Локальная копия [репозитория ASP.NET Benchmarks](https://github.com/aspnet/benchmarks)
* Локальная копия [репозитория Runtime](https://github.com/vitacore-company/runtime)
* Любой удобный редактор кода

## Настройка

Перед использованием удалённых серверов для бенчмарков необходимо выполнить следующие шаги.

### Сборка CoreCLR и генерация Core_Root

В репозитории Runtime вам понадобятся:
- бинарные файлы CoreCLR
- сгенерированный Core_Root

для выполнения кроссген-компиляции ASP.NET приложения. Базовые шаги:

Для Windows:

```powershell
.\build.cmd -subset clr+libs -c release
.\src\tests\build.cmd Release generatelayoutonly
```

Для Linux:

```bash
./build.sh -subset clr+libs -c release
./src/tests/build.sh -release -generatelayoutonly
```

### Генерация конфигурационного файла для запуска ASP.NET бенчмарков

Конфигурация ASP.NET бенчмарков задается с помощью профилей, которые определяются в `yml`-файлах. Ниже представлен простой пример конфигурационного файла, который используется в этом документе:

```yml
imports:
  - https://raw.githubusercontent.com/aspnet/Benchmarks/master/src/WrkClient/wrk.yml

jobs:
  aspnetbenchmarks:
    source:
      repository: https://github.com/aspnet/benchmarks.git
      branchOrCommit: master
      project: src/Benchmarks/Benchmarks.csproj
    readyStateText: Application started.
    variables:
      protocol: http
      server: Kestrel
      transport: Sockets
      scenario: plaintext
    channel: edge
    framework: net6.0
    arguments: "--nonInteractive true --scenarios {{scenario}} --server-urls {{protocol}}://[*]:{{serverPort}} --server {{server}} --kestrelTransport {{transport}} --protocol {{protocol}}"

scenarios:
  json:
    application:
      job: aspnetbenchmarks
      variables:
        scenario: json
    load:
      job: wrk
      variables:
        presetHeaders: json
        path: /json
        duration: 60
        warmup: 5
        serverPort: 5000

profiles:
  aspnet-physical-win:
    variables:
      serverUri: http://10.0.0.110
      cores: 12
    jobs:
      application:
        endpoints:
          - http://asp-perf-win:5001
      load:
        endpoints:
          - http://asp-perf-load:5001

  aspnet-physical-lin:
    variables:
      serverUri: http://10.0.0.102
      cores: 12
    jobs:
      application:
        endpoints:
          - http://asp-perf-lin:5001
      load:
        endpoints:
          - http://asp-perf-load:5001
```

Теперь разберём, что означает эта конфигурация и как она применяется. Рассмотрим ключевые поля:

* **Imports**: Внешние инструменты из репозитория Benchmarks. В данном случае используется только `wrk` - инструмент для нагрузочного тестирования веб-приложений.

* **Jobs**: Описание задач. Здесь задаются:
    * _Source_: Репозиторий с тестовым приложением
    * _Variables_: Параметры взаимодействия с сервером
    * _Channel_: Версия рантайма (`edge` - последняя nightly-сборка)
    * _Framework_: Версия .NET для сборки
    * _Arguments_: Аргументы командной строки для сервера

* **Scenarios**: Сценарии выполнения тестов:
    * _Application_: Выбор задачи для тестирования
    * _Load_: Инструмент нагрузочного тестирования (`wrk` с прогревом 5 сек и тестом 60 сек, заголовки `json`)

* **Profiles**: Профили машин для запуска тестов (предоставлены командой ASP.NET), в примере - для Windows и Linux

## Запуск бенчмарков

После подготовки конфигурации и сборки CoreCLR можно запускать тесты.

### Инициализация приложения

Из папки `BenchmarksDriver2` выполните команду:

Для Windows:

```powershell
dotnet run -- --config crossgen2-benchmarks.yml --scenario json --profile aspnet-physical-win
--application.options.fetch true
```

Для Linux:

```bash
dotnet run -- --config crossgen2-benchmarks.yml --scenario json --profile aspnet-physical-lin
--application.options.fetch true
```

#### Разбор и анализ предыдущей команды:

* `--config crossgen2-benchmarks.yml` - выбирает конфигурационный файл для использования
* `--scenario json` - запускает сценарий с меткой _json_ из конфигурационного файла
* `--profile aspnet-physical-win` - выбирает Windows-профиль из конфигурационного файла
* `--application.options.fetch true` - загружает собранное приложение, используемое для бенчмарков. Эти файлы нужны для применения _crossgen2_ и последующего сравнения результатов производительности.  
  Примечание: `application` - это просто метка из конфигурационного файла

После выполнения инструмент выведет сводку статистики о производительности.

### Crossgen2

1. Возьмите загруженный ZIP-файл с приложением и распакуйте его в отдельное место.  
   Это нужно чтобы избежать путаницы или случайного удаления при выполнении `git clean` и подобных команд.

2. В этом примере будет использоваться новая папка `results` вне репозиториев:
   * Распакуйте ZIP в папку, которую в этом примере будет называться `application`
   * Создайте внутри `results` другую папку `composite` - здесь будут храниться обработанные _crossgen2_ сборки

3. Перейдите в вашу папку `Core_Root` внутри репозитория _runtime_. Оттуда выполните _crossgen2_ следующей командой:

Для Windows:

```powershell
CoreRun.exe \runtime\artifacts\bin\coreclr\windows.x64.Release\crossgen2\crossgen2.dll
--Os --composite -o \path\to\results\composite\TotalComposite.dll \path\to\results\application\*.dll
```

Для Linux:

```bash
corerun /runtime/artifacts/bin/coreclr/Linux.x64.Release/crossgen2/crossgen2.dll
--Os --composite -o /path/to/results/composite/TotalComposite.dll /path/to/results/application/*.dll
```

Это сгенерирует новые сборки в папке `composite`, которые нужно скопировать в загруженную папку `application`, заменив уже существующие там файлы.

### Оптимизированное приложение

Чтобы запустить оптимизированную версию приложения:

1. Вернитесь в папку `BenchmarksDriver2`
2. Запустите драйвер со следующей командой:

Для Windows:

```powershell
dotnet run -- --config crossgen2-benchmarks.yml --scenario json --profile aspnet-physical-win
--application.options.outputFile \path\to\results\application\*
```

Для Linux:

```bash
dotnet run -- --config crossgen2-benchmarks.yml --scenario json --profile aspnet-physical-lin
--application.options.outputFile /path/to/results/application/*.dll
```

Эта команда аналогична первоначальной, с одним отличием:

* `--application.options.outputFile` - указывает инструменту загрузить ваше оптимизированное приложение (обработанное crossgen2) и проводить тестирование именно с этой версией

Как и ранее, после завершения теста будет показана сводка статистики производительности, которую можно сравнить с исходными результатами для последующего анализа.
