# Отладка WASM runtime

-   Отключите удаление символов, установив свойство WasmNativeStrip в msbuild на false. Инструкции по сбору стектрейсов (stack-trace) с символами в Blazor представлены [ниже](#blazor).

-   Emscripten генерирует отладочную информацию в формате DWARF, и Chrome начиная с версии 80 может её использовать.

-   Для остановки JS-отладчика кода runtime:

```
#include <emscripten.h>
EM_ASM(debugger;);
```

-   Для вывода стектрейса из кода runtime:

```
#ifdef HOST_WASM
#include <emscripten.h>
EM_ASM(
	var err = new Error();
	console.log ("Stacktrace: \n");
	console.log (err.stack);
	);
#endif
```

Существует функция `mono_wasm_print_stack_trace()`, которая делает то же самое:

```
#ifdef HOST_WASM
mono_wasm_print_stack_trace ();
#endif
```

Директива `ifdef` нужна, чтобы избежать ошибок компиляции при сборке кросс-компилятора.

-   Тестовый раннер runtime-tests.js поддерживает различные опции для отладки:

    1.  Параметры командной строки runtime можно передать с помощью опции `--runtime-arg=<arg>`. В частности, `--trace` можно использовать для включения трассировки выполнения при использовании интерпретатора.
    2.  Переменные окружения можно задать с помощью `--setenv=<var>=<value>`. В частности, можно задать `MONO_LOG_LEVEL` и `MONO_LOG_MASK`.

-   Опция `--stack-trace-limit=1000` для V8 позволяет избежать усечения стектрейсов.

-   Emscripten поддерживает опцию `-fsanitize=address` в clang, а также может декомпилировать wasm-образы во время выполнения для создания читаемых стектрейсов для C-кода.

-   Cтектрейсы используют числа, например:

```
WebAssembly.instantiate:wasm-function[8003]:0x12b564
```

Эти числа означают индекс функции wasm и смещение внутри wasm-бинарника.
Инструмент wasm-objdump из https://github.com/WebAssembly/wabt можно использовать для поиска соответствующего wasm-кода:

```
12b551 func[8003] <mono_wasm_load_runtime>:
```

-   Инструмент `wasm-dis` из [github.com/WebAssembly/binaryen](https://github.com/WebAssembly/binaryen) можно использовать для дизассемблирования wasm-исполняемых файлов (файлов .wasm).

## Детерминированное выполнение

Работу wasm можно сделать детерминированной (determenistic execution) при помощи опции `-s DETERMINISTIC=1` в `emcc`. Эта команда заставит приложение всегда выполняться одинаково, т.е. использовать одни и те же адреса памяти, случайные числа и т.д. Это можно использовать для того, чтобы случайные сбои происходили предсказуемо. Однако иногда включение этой опции может привести к незримым проблемам. В этом случае может быть полезно добавить немного контролируемой недетерминированности (indeterminism). Например, чтобы сделать генератор случайных чисел частично детерминированным, измените `$getRandomDevice` в `upstream/emscripten/src/library.js` на:

```
	var randomBuffer2 = new Uint8Array(1);
	crypto.getRandomValues(randomBuffer2);

	FS.seed2 = randomBuffer2 [0];
	console.log('SEED: ' + FS.seed2);
	return function() {
		FS.seed2 = FS.seed2 * 16807 % 2147483647;
		return FS.seed2;
	};
```

Запустите приложение до возникновения сбоя. Запишите значение seed, выведенное в начале и измените строку
`FS.seed2 = randomBuffer...` на :
`FS.seed2 = <seed value>`.
Это должно заставить сбой происходить предсказуемо.

Также есть ещё один генератор случайных чисел в `upstream/emscripten/src/deterministic.js`, который требует аналогичных изменений.

Запуск `make patch-deterministic` в `src/mono/wasm` применит эти изменения к установке
emscripten в `src/mono/browser/emsdk`.

## Отладка ошибок несоответствия сигнатур

Когда v8 завершается с ошибкой `RuntimeError: function signature mismatch`, это означает, что был вызов функции по указателю с несовместимой сигнатурой или по указателю NULL. Эта ветка v8 содержит некоторые модификации для вывода фактического значения указателя на функцию при возникновении такой ошибки: [github.com/vargaz/v8/tree/sig-mismatch](https://github.com/vargaz/v8/tree/sig-mismatch). Значение является индексом в таблице функций внутри wasm-исполняемого файла.

Следующий скрипт можно использовать для вывода таблицы:

```
#!/usr/bin/env python3

#
# print-table.py: Print the function table for a webassembly .wast file
#

import sys

prefix=" (elem (i32.const 1) "

if len(sys.argv) < 2:
    print ("Usage: python print-table.py <path to mono.wast>")
    sys.exit (1)

f = open (sys.argv [1])
table_line = None
for line in f:
     if prefix in line:
         table_line = line[len(prefix):]
         break

for (index, v) in enumerate (table_line.split (" ")):
    print ("" + str(index) + ": " + v)
    index += 1
```

Входные данные для скрипта — текстовый ассемблер, созданный инструментом wasm-dis.

Такие ошибки обычно возникают из-за того, что в runtime mono есть вспомогательные функции, которые никогда не должны вызываться, например, `no_gsharedvt_in_wrapper` или `no_llvmonly_interp_method_pointer`. Эти функции используются как заглушки для указателей на функции с разными сигнатурами, поэтому если они вызываются из-за ошибки, возникает ошибка несоответствия сигнатур.

## Сбор стектрейсов с символами в Blazor

При отладке нативного сбоя в приложении .NET 6 Blazor или другом фреймворке WebAssembly, использующем наш стандартный `dotnet.wasm`, нативные стектреймы не будут содержать имён C-символов, а будут выглядеть как `$func1234`.

Например, эта страница Razor вызовет сбой при нажатии пользователем кнопки `Crash`:

```csharp
<button class="btn btn-warning" @onclick="Crash">Crash</button>

@code {
    private void Crash ()
    {
        IntPtr p = (IntPtr)0x01;
        Console.WriteLine ("About to crash");
        System.Runtime.InteropServices.Marshal.FreeHGlobal(p);
    }
}
```

Нажатие на кнопку `Crash` выведет в консоль следующее (индексы функций могут отличаться):

```console
dotnet.wasm:0x1d8355 Uncaught (in promise) RuntimeError: memory access out of bounds
    at _framework/dotnet.wasm
    at _framework/dotnet.wasm
    at _framework/dotnet.wasm
    at _framework/dotnet.wasm
    at _framework/dotnet.wasm
    at _framework/dotnet.wasm
    at _framework/dotnet.wasm
    at _framework/dotnet.wasm
    at _framework/dotnet.wasm
    at _framework/dotnet.wasm
$free @ dotnet.wasm:0x1d8355
$func4027 @ dotnet.wasm:0xead6a
$func219 @ dotnet.wasm:0x1a03a
$func167 @ dotnet.wasm:0xcaf7
$func166 @ dotnet.wasm:0xba0a
$func2810 @ dotnet.wasm:0xabacf
$func1615 @ dotnet.wasm:0x6f8eb
$func1613 @ dotnet.wasm:0x6f85d
$func966 @ dotnet.wasm:0x502dc
$func219 @ dotnet.wasm:0x1a0e2
$func167 @ dotnet.wasm:0xcaf7
$func166 @ dotnet.wasm:0xba0a
$func2810 @ dotnet.wasm:0xabacf
$func1615 @ dotnet.wasm:0x6f8eb
$func1619 @ dotnet.wasm:0x6ff58
$mono_wasm_invoke_jsexport @ dotnet.wasm:0x96c9
Module.mono_wasm_invoke_jsexport @ dotnet.6.0.1.hopd7ipo8x.js:1
managed__Microsoft_AspNetCore_Components_WebAssembly__Microsoft_AspNetCore_Components_WebAssembly_Services_DefaultWebAssemblyJSRuntime_BeginInvokeDotNet @ managed__Microsoft_AspNetCore_Components_WebAssembly__Microsoft_AspNetCore_Components_WebAssembly_Services_DefaultWebAssemblyJSRuntime_BeginInvokeDotNet:19
beginInvokeDotNetFromJS @ blazor.webassembly.js:1
b @ blazor.webassembly.js:1
invokeMethodAsync @ blazor.webassembly.js:1
(anonymous) @ blazor.webassembly.js:1
invokeWhenHeapUnlocked @ blazor.webassembly.js:1
S @ blazor.webassembly.js:1
C @ blazor.webassembly.js:1
dispatchGlobalEventToAllElements @ blazor.webassembly.js:1
onGlobalEvent @ blazor.webassembly.js:1
```

Чтобы получить символы (symbols):

1. Установите `workload wasm-tools` с помощью `dotnet workload install wasm-tools`.
2. Укажите дополнительные свойства в файле `.csproj`:

    ```xml
      <!-- Builds a dotnet.wasm with debug symbols preserved -->
      <PropertyGroup>
        <WasmBuildNative>true</WasmBuildNative>
        <WasmNativeStrip>false</WasmNativeStrip>
      </PropertyGroup>
    ```

3. Удалите папки `bin` и `obj`. Далее нужно пересобрать проект и запустить его снова.

Теперь нажатие на кнопку `Crash` выведет стектрейс с символами:

```console
dotnet.wasm:0x224878 Uncaught (in promise) RuntimeError: memory access out of bounds
    at dlfree (dotnet.wasm:0x224878)
    at SystemNative_Free (dotnet.wasm:0x20f0e2)
    at do_icall (dotnet.wasm:0x190f9)
    at do_icall_wrapper (dotnet.wasm:0x18429)
    at interp_exec_method (dotnet.wasm:0xa56c)
    at interp_runtime_invoke (dotnet.wasm:0x943a)
    at mono_jit_runtime_invoke (dotnet.wasm:0x1dec32)
    at do_runtime_invoke (dotnet.wasm:0x95fca)
    at mono_runtime_invoke_checked (dotnet.wasm:0x95f57)
    at mono_runtime_try_invoke_array (dotnet.wasm:0x9a87e)
$dlfree @ dotnet.wasm:0x224878
$SystemNative_Free @ dotnet.wasm:0x20f0e2
$do_icall @ dotnet.wasm:0x190f9
$do_icall_wrapper @ dotnet.wasm:0x18429
$interp_exec_method @ dotnet.wasm:0xa56c
$interp_runtime_invoke @ dotnet.wasm:0x943a
$mono_jit_runtime_invoke @ dotnet.wasm:0x1dec32
$do_runtime_invoke @ dotnet.wasm:0x95fca
$mono_runtime_invoke_checked @ dotnet.wasm:0x95f57
$mono_runtime_try_invoke_array @ dotnet.wasm:0x9a87e
$mono_runtime_invoke_array_checked @ dotnet.wasm:0x9af17
$ves_icall_InternalInvoke @ dotnet.wasm:0x702ed
$ves_icall_InternalInvoke_raw @ dotnet.wasm:0x7777f
$do_icall @ dotnet.wasm:0x191c5
$do_icall_wrapper @ dotnet.wasm:0x18429
$interp_exec_method @ dotnet.wasm:0xa56c
$interp_runtime_invoke @ dotnet.wasm:0x943a
$mono_jit_runtime_invoke @ dotnet.wasm:0x1dec32
$do_runtime_invoke @ dotnet.wasm:0x95fca
$mono_runtime_try_invoke @ dotnet.wasm:0x966fe
$mono_runtime_invoke @ dotnet.wasm:0x98982
$mono_wasm_invoke_jsexport @ dotnet.wasm:0x227de2
Module.mono_wasm_invoke_jsexport @ dotnet..y6ggkhlo8e.js:9927
managed__Microsoft_AspNetCore_Components_WebAssembly__Microsoft_AspNetCore_Components_WebAssembly_Services_DefaultWebAssemblyJSRuntime_BeginInvokeDotNet @ managed__Microsoft_AspNetCore_Components_WebAssembly__Microsoft_AspNetCore_Components_WebAssembly_Services_DefaultWebAssemblyJSRuntime_BeginInvokeDotNet:19
beginInvokeDotNetFromJS @ blazor.webassembly.js:1
b @ blazor.webassembly.js:1
invokeMethodAsync @ blazor.webassembly.js:1
(anonymous) @ blazor.webassembly.js:1
invokeWhenHeapUnlocked @ blazor.webassembly.js:1
S @ blazor.webassembly.js:1
C @ blazor.webassembly.js:1
dispatchGlobalEventToAllElements @ blazor.webassembly.js:1
onGlobalEvent @ blazor.webassembly.js:1
```

## Включение дополнительного логирования в Blazor

В .NET 8+ запуск Blazor можно контролировать, установив атрибут `autostart="false"` в
теге `<script>`, который загружает фреймворк Blazor WebAssembly. После этого функцию `globalThis.Blazor.start()` можно использовать для передачи дополнительных
параметров конфигурации, включая установку переменных окружения mono или дополнительных аргументов командной строки.

Имя скрипта и расположение тега `<script>` зависят от типа проекта: Blazor WebAssembly (шаблон blazorwasm) или Blazor (шаблон blazor).

См. интерфейс DotnetHostBuilder в [dotnet.d.ts](https://github.com/vitacore-company/runtime/blob/main/src/mono/browser/runtime/dotnet.d.ts) для дополнительных функций конфигурации.

## Blazor WebAssembly

Скрипт для работы на проекте `blazorwasm` называется `_framework/blazor.webassembly.js`. Он загружается в `wwwroot/index.html`:

```html
<body>
    <div id="app">...</div>

    <div id="blazor-error-ui">...</div>
    <script src="_framework/blazor.webassembly.js"></script>
</body>
```

Замените скрипт на:

```html
<body>
    <div id="app">...</div>

    <div id="blazor-error-ui">...</div>
    <script src="_framework/blazor.webassembly.js" autostart="false"></script>

    <script>
        Blazor.start({
            configureRuntime: (dotnet) => {
                dotnet.withEnvironmentVariable("MONO_LOG_LEVEL", "debug");
                dotnet.withEnvironmentVariable("MONO_LOG_MASK", "all");
            },
        });
    </script>
</body>
```

## Blazor {#blazor}

Cкрипт для работы на проекте `blazor` называется `_framework/blazor.web.js`. Он загружается в `Components/App.razor` в серверной части проекта:

```html
<body>
    <Routes />
    <script src="_framework/blazor.web.js"></script>
</body>
```

Замените срипт следующим образом (обратите внимание, что для проекта blazor в `Blazor.start` нужен дополнительный словарь с ключом `webAssembly`):

```html
<body>
    <Routes />
    <script src="_framework/blazor.web.js" autostart="false"></script>
    <script>
        Blazor.start({
            webAssembly: {
                configureRuntime: (dotnet) => {
                    console.log("in configureRuntime");
                    dotnet.withEnvironmentVariable("MONO_LOG_LEVEL", "debug");
                    dotnet.withEnvironmentVariable("MONO_LOG_MASK", "all");
                },
            },
        });
    </script>
</body>
```
