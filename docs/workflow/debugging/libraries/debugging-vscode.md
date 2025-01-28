# Отладка библиотек с помощью Visual Studio Code

1. Установите [Visual Studio Code](https://code.visualstudio.com/)
2. Установите [расширение C#](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csharp)
3. Если необходимо, установите [C# Dev Kit](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csdevkit)
4. Откройте папку, содержащую исходный код, который вы хотите отлаживать в VS Code, например, если вы отлаживаете сбой теста в System.Net.Sockets, откройте `runtime/src/libraries/System.Net.Sockets`
5. Откройте окно отладки: `ctrl-shift-D` или нажмите на соответствующую кнопку на панели слева.
6. Нажмите "create a launch.json file"выберите опцию, которая включает `.NET Core` из выпадающего списка.
7. В файле конфигурации `launch.json` для ".NET Core Launch (console)" внесите следующие изменения:
    1. Удалить свойство `prelaunchtask`
    2. Установить `program` на полный путь к` dotnet` в папке `artifacts/bin/testhost`
        - например `{полный путь к dotnet/runtime directory}/artifacts/bin/testhost/net{Version}-{OS}-{Configuration}-{Architecture}/dotnet`.
    3. Установить `cwd` на директорию бинарных файлов теста.
        - например, для System.Net.Sockets: `{полный путь к dotnet/runtime directory}/artifacts/bin/System.Net.Sockets.Tests/Debug/net{Version}-{OS}`. Точное наименование или структура могут отличаться в зависимости от библиотеки, с которой вы работаете.
    4. Установить `args` на аргументы командной строки, которые нужно передать тесту:
        - например: `[ "exec", "--runtimeconfig", "{TestProjectName}.runtimeconfig.json", "xunit.console.dll", "{TestProjectName}.dll", "-notrait", ... ]`, здесь TestProjectName будет представлять `System.Net.Sockets.Tests`;
        - чтобы запустить конкретный тест, вы можете добавить следующее: `[ "-method", "System.Net.Sockets.Tests.{ClassName}.{TestMethodName}", ...]`;
        - чтобы найти точные аргументы для воспроизведения запуска теста, который вы пытаетесь отладить, выполните команду теста в терминале и ищите вывод, начинающийся с `exec`. Скопируйте все аргументы и отформатируйте их в `launch.json` , как показано выше;
            - например, команда `dotnet build /t:Test` в папке `runtime/src/libraries/System.Net.Sockets/tests/FunctionalTests` выводит в терминал `"exec --runtimeconfig System.Net.Sockets.Tests.runtimeconfig.json ... -notrait category=failing"`, что можно отформатировать в следующее: `["exec","--runtimeconfig","System.Net.Sockets.Tests.runtimeconfigjson", ... ,"-notrait","category=failing"]`
            - аналогично, комнда `dotnet build /t:Test /p:xUnitMethodName=System.Net.Sockets.Tests.{ClassName}.{TestMethodName}` возвращает аргументы, которые необходимы для отладки конкретного теста.
8. Установите breakpoint и запустите отладчик (выбрав ".NET Core Launch (console)"). Запустится инспекция переменных и стеков вызовов (call stacks).
9. Если необходимо, сохраните настройки запуска в файле [workspace](https://code.visualstudio.com/docs/editor/workspaces). Файл не обязательно должен находиться в `.vscode` и в текущей открытой папке, поэтому его гораздо проще сохранить при помощи `git clean -dfx`.

## Отладка библиотек с помощью VS Code на Mono

Чтобы отлаживать библиотеки на "десктопной" ОС-и (Linux/Mac/Windows, не WebAssembly, iOS или Android), которая будет работать с Mono runtime, следуйте инструкциям ниже.
Также смотрите главу [отладка Android](../mono/android-debugging.md) и [отладка WebAssembly](../mono/wasm-debugging.md)

-   Установите [расширение Mono Debugger (`ms-vscode.mono-debug`)](https://marketplace.visualstudio.com/items?itemName=ms-vscode.mono-debug) для VS Code
-   Создайте файл `launch.json` с типом `mono`:

    ```json
    {
        "version": "0.2.0",
        "configurations": [
            {
                "name": "Attach to Mono",
                "type": "mono",
                "request": "attach",
                "address": "localhost",
                "port": 1235
            }
        ]
    }
    ```

-   Запустите тест из командной строки, установив переменную окружения `MONO_ENV_OPTIONS`, чтобы настроить отладчик:

    ```sh
    DOTNET_REMOTEEXECUTOR_SUPPORTED=0 MONO_ENV_OPTIONS="--debug --debugger-agent=transport=dt_socket,address=127.0.0.1:1235,server=y,suspend=y" ./dotnet.sh build /t:Test /p:RuntimeFlavor=Mono src/libraries/System.Buffers/tests
    ```

    Обратите внимание, что нужно установить `DOTNET_REMOTEEXECUTOR_SUPPORTED=0`, иначе несколько экземпляров runtime будут слушать один и тот же порт.

    При работе на Windows не передавайте `--debug` в `MONO_ENV_OPTIONS`.

-   Установите breakpoint тестов в VS Code и начните отладку в конфигурации "Attach to Mono".
-   Обратите внимание, что Mono не останавливается на first chance exceptions. xUnit перехватывает все исключения, и поэтому, если тест вызывает такое исключение, отладчик не остановится на необработанном исключении.
