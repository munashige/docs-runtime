Отладка core-библиотек на Windows
=================================

Отладка core-библиотек на Windows производится с помощью _Visual Studio_ или _WinDBG_.

Для работы в Visual Studio следуйте инструкциям в [этой главе](../../testing/visualstudio.md), чтобы запустить и начать отладку с тестами. Для ошибок, которые не удается воспроизвести в Visual Studio (проблемы с жизненным циклом SafeHandle, race conditions и т. д.), потребуется использовать WinDBG.


## Необходимое программное обеспечение

* [Скачать WDK и WinDBG](https://msdn.microsoft.com/en-us/windows/hardware/hh852365.aspx)

_Примечание_: Можно выбрать только инструменты отладки (debugging tools) для Windows или скачать WinDBG отдельно через WDK.

## Предварительные условия для работы с WinDBG

1. Сначала необходимо собрать весь репозиторий. Таким образом, все нужные пакеты будут успешно загружены.

2. Установите WinDBG в качестве отладчика _post-mortem_. Запустите от имени администратора:

```
windbg -I
```

Действия выше возможно потребуются как для x64, так и для x86 архитектур.
Любое приложение, которое закрывается с ошибкой, должно автоматически запускать сессию WinDBG.

## Debugging tests
Чтобы запустить один тест из командной строки:

1. Найдите папку с бинарными файлами с тем же именем, что и проект CSPROJ.

Например: `src\System.Net.Sockets\tests\Functional\System.Net.Sockets.Tests.csproj` выведет бинарные файлы в следующей директории:  `bin\tests\windows.AnyCPU.Debug\System.Net.Sockets.Tests\netcoreapp1.0`.

2. Запустите тесты.

Например, если репозиторий находится в `C:\root`:

```
cd C:\root\bin\tests\windows.AnyCPU.Debug\System.Net.Sockets.Tests\netcoreapp1.0
C:\root\bin\tests\windows.AnyCPU.Debug\System.Net.Sockets.Tests\netcoreapp1.0\CoreRun.exe xunit.console.dll System.Net.Sockets.Tests.dll -xml testResults.xml -notrait category=nonwindowstests -notrait category=OuterLoop -notrait category=failing
```

3. Если тест вернет ошибку или вызов метода `Debugger.Launch()`, WinDBG автоматически запустится и подключится к процессу `CoreRun.exe`.

Следующие команды корректно настроят расширение для отладки и исправят ссылки на символы, а также исходный код:

```
.symfix
.srcfix
.reload
!load C:\root\packages\runtime.win7-x64.Microsoft.NETCore.Runtime.CoreCLR\<version>\tools\sos
```

_Важно_: Укажите правильный путь к вашему расширению SOS на этапе выполнения предварительных условий (шаг 2).

Документация по использованию SOS доступна на сайте [MSDN](https://msdn.microsoft.com/en-us/library/bb190764\(v=vs.110\).aspx).

Для быстрой справки введите следующее в WinDBG:

```
0:000> !sos.help
```

## Трассировка

В Windows трассировки, сгенерированные EventSource, собираются через ETW с использованием logman или PerfView.

### Использование Logman
[Logman](https://technet.microsoft.com/en-us/library/bb490956.aspx) поставляется с Windows и не требует загрузки или установки.
Поскольку ETW-провайдеры динамически генерируются и регистрируются в .NET, нужно использовать GUID, а не имена, которые используется в logman.

#### Трассировка одного провайдера

Ниже приведен пример трассировки Sockets:

```
    logman -start SocketTrace -o %SYSTEMDRIVE%\sockets.etl -p "{e03c0352-f9c9-56ff-0ea7-b94ba8cabc6b}" -ets

    // Repro

    logman -stop SocketTrace -ets
```

Логи будут сохранены в `%SYSTEMDRIVE%\sockets.etl`.

#### Трассировка нескольких провайдеров

1. Создайте файл `providers.txt` со следующим кодом:

    ```
    "{e03c0352-f9c9-56ff-0ea7-b94ba8cabc6b}"
    "{066c0e27-a02d-5a98-9a4d-078cc3b1a896}"
    "{bdd9a83e-1929-5482-0d73-2fe5e1c0e16d}"
    ```

2. Создайте трассировку:

    ```
    logman create trace SystemNetTrace -o sn.etl -pf providers.txt
    ```

3. Запустите трассировку:

    ```
    logman start SystemNetTrace
    ```

4. Воспроизведите проблему.
5. Остановите трассировку:

    ```
    logman stop SystemNetTrace
    ```

   Трассировку можно перезапустить с шага 3.

6. Удалите профиль трассировки, если он не будет использоваться повторно:
    ```
    logman delete SystemNetTrace
    ```

7. Логи будут сохранены в sn.etl.

### Использование PerfView

1. Установите [PerfView](https://github.com/Microsoft/perfview/blob/master/documentation/Downloading.md).
2. Запустите `PerfView` от имени администратора.
3. Нажмите `Alt+C` для сбора событий..
5. Добавьте дополнительные провайдеры (см. ниже).

_Важно_: используйте символ `*` перед указанными именами.

![Пример PerfView](perfview_example.gif)

### Встроенная трассировка EventSource

Следующие EventSource встроены в платформу .NET. Те, которые не помечены как [TestCode], могут быть включены в производственных сценариях для сбора логов.

#### Global
* `*System.Diagnostics.Eventing.FrameworkEventSource {8E9F5090-2D75-4d03-8A81-E5AFBF85DAF1}`: Глобальный `EventSource`, который используется несколькими пространствами имен.

#### System.Collections
* `*System.Collections.Concurrent.ConcurrentCollectionsEventSource {35167F8E-49B2-4b96-AB86-435B59336B5E}`: Предоставляет источник событий для трассировки информации о коллекциях `Coordination Data Structure`.

#### System.Linq
* `*System.Linq.Parallel.PlinqEventSource {159eeeec-4a14-4418-a8fe-faabcd987887}`: Предоставляет источник событий для трассировки информации о `PLINQ`.

#### Пространства имен System.Net

Вспомогательные скрипты доступны по этому [адресу](https://github.com/dotnet/runtime/tree/main/src/libraries/Common/tests/Scripts/Tools). Запустите `net_startlog.cmd` от имени администратора. Далее запустите приложение и выполните `net_stoplog.cmd`. Откройте файл `.etl` с помощью `PerfView`.

* `*Microsoft-System-Net-Http {bdd9a83e-1929-5482-0d73-2fe5e1c0e16d}`: Трассировки, связанные с HTTP.
* `*Microsoft-System-Net-Http-WinHttpHandler {b71555b1-9566-5ce3-27f5-98405bbfde9d}`: Трассировки, связанные с WinHttpHandler.
* `*Microsoft-System-Net-Mail {42c8027b-f048-58d2-537d-a4a9d5ee7038}`: Трассировки, связанные с SMTP.
* `*Microsoft-System-Net-NameResolution {5f302add-3825-520e-8fa0-627b206e2e7e}`: Трассировки, связанные с DNS.
* `*Microsoft-System-Net-NetworkInformation {b8e42167-0eb2-5e39-97b5-acaca593d3a2}`: Трассировки, связанные с конфигурацией сети.
* `*Microsoft-System-Net-Ping {a771ec4a-7260-59ce-0475-db257437ed8c}`: Трассировки, связанные с Ping.
* `*Microsoft-System-Net-Primitives {a9f9e4e1-0cf5-5005-b530-3d37959d5e84}`: Трассировки, связанные с базовыми сетевыми типами.
* `*Microsoft-System-Net-Requests {3763dc7e-7046-5576-9041-5616e21cc2cf}`: Трассировки, связанные с WebRequest.
* `*Microsoft-System-Net-Sockets {e03c0352-f9c9-56ff-0ea7-b94ba8cabc6b}`: Трассировки, связанные с сокетами.
* `*Microsoft-System-Net-Security {066c0e27-a02d-5a98-9a4d-078cc3b1a896}`: Трассировки, связанные с безопасностью.

#### System.Threading
* `*System.Threading.SynchronizationEventSource {EC631D38-466B-4290-9306-834971BA0217}`: Предоставляет источник событий для трассировки информации о синхронизации Coordination Data Structure.
* `*System.Threading.Tasks.TplEventSource {2e5dba47-a3d2-4d16-8ee0-6671ffdcd7b5}`: Предоставляет источник событий для трассировки информации о TPL.
* `*System.Threading.Tasks.Parallel.EventSource`: Предоставляет источник событий для трассировки информации о TPL.
* `*System.Threading.Tasks.Dataflow.DataflowEventSource {16F53577-E41D-43D4-B47E-C17025BF4025}`: Предоставляет источник событий для трассировки информации о Dataflow.

## Примечание
Команду для вызова теста можно найти в логах, сгенерированных после выполнения `dotnet build /t:test` в папке с тестами.
