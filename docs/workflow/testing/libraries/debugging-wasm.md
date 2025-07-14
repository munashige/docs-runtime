# Отладка библиотек

Для сборки библиотек и их тестировании без отладки см.:

-   [Сборка библиотек](https://github.com/dotnet/runtime/blob/main/docs/workflow/building/libraries/README.md),
-   [Тестирование библиотек](https://github.com/dotnet/runtime/blob/main/docs/workflow/testing/libraries/testing.md).

## Запуск тестов с поддержкой отладчика

Запустите выбранные тесты библиотек в браузере. Например, `System.Collections.Concurrent.Tests`:

```
dotnet run -r browser-wasm -c Debug --project src/libraries/System.Collections/tests/System.Collections.Tests.csproj --debug --host browser -p:DebuggerSupport=true
```

Здесь выбран `browser-wasm` в качестве runtime. Также используется настройка `DebuggerSupport=true`, чтобы тесты не запустились, пока отладчик не будет подключен. Вывод будет содержать следующие строки:

```
Debug proxy for chrome now listening on http://127.0.0.1:58346/. And expecting chrome at http://localhost:9222/
App url: http://127.0.0.1:9000/index.html?arg=--debug&arg=--run&arg=WasmTestRunner.dll&arg=System.Collections.Concurrent.Tests.dll
```

В следующем шаге будет использован URL/порт прокси .

Может понадобиться закрыть все экземпляры Chrome. Затем запустить браузер с включенным режимом отладки:

`chrome --remote-debugging-port=9222 <APP_URL>`

Теперь можно выбрать IDE для начала отладки. Стоит отметить, что тесты ждут только до тех пор, пока отладчик не будет подключен. При подключении они запустятся сразу. Поэтому рекомендуется создать точки останова (breakpoints) заранее, перед подключением отладчика. Например, можно создать такую точку в `src\libraries\Common\tests\WasmTestRunner\WasmTestRunner.cs` на первой строке `Main()`, чтобы предотвратить запуск тестов и дать время на подготовку.

## Отладка с помощью Chrome DevTools

Откройте `chrome://inspect/#devices` в новой вкладке браузера, который вы запустили. Выберите `Configure`:

![image](https://user-images.githubusercontent.com/32700855/201867874-7f707eb1-e859-441c-8205-abb70a7a0d0b.png)

Вставьте адрес прокси, который был предоставлен в выводе программы.

![image](https://user-images.githubusercontent.com/32700855/201862487-df76a06c-b24d-41a0-bf06-6959bba59a58.png)

Будут отображены новые удаленные цели (remote targets). Выберите адрес, который вы открыли в другой вкладке, нажав `Inspect`.

![image](https://user-images.githubusercontent.com/32700855/201863048-6a4fe20b-a215-435d-b594-47750fcb2872.png)

Откроется новое окно с Chrome DevTools. На вкладке `sources` вам следует искать каталог `file://`. Здесь можно просматривать дерево файлов библиотек и открывать исходный код. Загрузка файлов также может занять некоторое время. Когда IDE будет готов, тесты запустятся.

Невозможно установить точки останова в Chrome DevTools, пока файлы не будут загружены. Поэтому можно использовать первый запуск для создания начальной точки останова в `WasmTestRunner.cs`, а затем перезапустить приложение. DevTools остановится на ранее созданной точке останова, и таким образом останется время создать точки останова в библиотеках, которые вы хотите отлаживать, и нажать "Resume".

## Отладка с помощью VS Code

Добавьте следующую конфигурацию в `.vscode/launch.json`:

```
        {
            "name": "Libraries",
            "request": "attach",
            "type": "chrome",
            "address": "localhost",
            "port": <PROXY'S_PORT>
        }
```

Создайте как минимум одну точку останова в библиотеках, например, изначально в `WasmTestRunner.cs`.

Запустите конфигурацию и ждите, это может занять некоторое время. Когда VS Code остановится в `WasmTestRunner`, создайте точки останова в библиотеках, которые вы хотите отлаживать, и нажмите "Resume".

![image](https://user-images.githubusercontent.com/32700855/201894003-fc5394ad-9848-4d07-a132-f687ecd17c50.png)
