Отладка core-библиотек на Unix
==============================

Отладка core-библиотек на системах Unix производится с помощью _lldb_ и _Visual Studio Code_.

## SOS и lldb

- Установите SOS и lldb. См. [инструкции по настройке](https://github.com/dotnet/diagnostics/blob/main/documentation/sos.md) и [документацию dotnet-sos](https://learn.microsoft.com/dotnet/core/diagnostics/dotnet-sos).
- Запустите тест с помощью msbuild хотя бы один раз с параметром `/t:Test`.

## Отладка дампов памяти с помощью lldb

SOS и lldb могут быть использованы для отладки crash-дампов .NET. Для этого понадобится следующее:

- Файл дампа памяти.
- На Linux понадобится утилита под названием `createdump` ([документация](../../../design/coreclr/botr/xplat-minidump-generation.md)), которую можно настроить для генерации дампов памяти, когда управляемое приложение вызывает необработанное исключение или сбой.

Инструкции по установке lldb и SOS можно найти [здесь](https://github.com/dotnet/diagnostics/blob/main/documentation/sos.md).

Если все перечисленное выше установлено, можно начинать отладку. Также нужно указать дополнительный параметр для lldb, чтобы он правильно разрешал символы для libcoreclr.so. Для этого используйте следующую команду:

```
lldb-3.9 -O "settings set target.exec-search-paths <runtime-path>" --core <core-file-path> <host-path>
```

- `<runtime-path>`: Путь, который содержит `libcoreclr.so.dbg` и остальные ассамблеи runtime и фреймворка.
- `<core-file-path>`: Путь к дампу памяти, который нужно отладить.
- `<host-path>`: Путь к исполняемому файлу dotnet или corerun, потенциально находящемуся в папке `<runtime-path>`.

На этом этапе lldb должен успешно начать отладку. Вы должны увидеть трассировки стека (stracktraces) с разрешенными символами для `libcoreclr.so`. Теперь вы можете начать использовать команды SOS, если вы настроили их, как описано в ссылках.

Для получения дополнительных сведений о coredump см. [эту ссылку](https://github.com/dotnet/diagnostics/blob/main/documentation/debugging-coredump.md).

##### Пример

```
lldb-3.9 -O "settings set target.exec-search-paths /home/parallels/Downloads/System.Drawing.Common.Tests/home/helixbot/dotnetbuild/work/2a74cf82-3018-4e08-9e9a-744bb492869e/Payload/shared/Microsoft.NETCore.App/$(ProductVersion)/" --core /home/parallels/Downloads/System.Drawing.Common.Tests/home/helixbot/dotnetbuild/work/2a74cf82-3018-4e08-9e9a-744bb492869e/Work/f6414a62-9b41-4144-baed-756321e3e075/Unzip/core /home/parallels/Downloads/System.Drawing.Common.Tests/home/helixbot/dotnetbuild/work/2a74cf82-3018-4e08-9e9a-744bb492869e/Payload/shared/Microsoft.NETCore.App/$(ProductVersion)/dotnet
```
