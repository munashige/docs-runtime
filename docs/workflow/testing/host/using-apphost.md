# Использование локальной сборки `apphost`

При сборке .NET приложения [`apphost`](../../../design/features/host-components.md#entry-point-hosts) используется в качестве исполняемого файла для приложения. Он переименовывается, чтобы соответствовать приложению, и обновляется, чтобы ассоциироваться с управляемым `.dll` приложения. .NET SDK ищет `apphost`, проверяя установленные пакеты `Microsoft.NETCore.App.Host` в каталоге `<dotnet_root>/packs`, соответствующие операционной системе, архитектуре и версии. Если совпадение не найдено, он загружает соответствующий пакет NuGet.

Чтобы заставить SDK использовать конкретный `apphost` при сборке проекта, установите параметр [`AppHostSourcePath`](https://github.com/dotnet/sdk/blob/f106bca2c28aeb4de8cafa8ff818bd8613908964/src/Tasks/Microsoft.NET.Build.Tasks/targets/Microsoft.NET.Sdk.FrameworkReferenceResolution.targets#L295) на полный путь к вашему локальному бинарному файлу `apphost` - например, `<repo_root>/artifacts/bin/<os>-<arch>.<configuration>/corehost/apphost[.exe]`.

```xml
<PropertyGroup>
  <AppHostSourcePath>[full_path_to_apphost]</AppHostSourcePath>
</PropertyGroup>
```

Для однофайлового приложения устан5овите свойство [`SingleFileHostSourcePath`](https://github.com/dotnet/sdk/blob/f106bca2c28aeb4de8cafa8ff818bd8613908964/src/Tasks/Microsoft.NET.Build.Tasks/targets/Microsoft.NET.Sdk.FrameworkReferenceResolution.targets#L305) на полный путь к локальному бинарному файлу `singlefilehost` - например, `<repo_root>/artifacts/bin/<os>-<arch>.<configuration>/corehost/singlefilehost[.exe]`.

```xml
<PropertyGroup>
  <PublishSingleFile>true</PublishSingleFile>
  <SingleFileHostSourcePath>[full_path_to_singlefilehost]</SingleFileHostSourcePath>
</PropertyGroup>
```

Сборка и публикация проекта теперь должны использовать указанные `apphost`/`singlefilehost`.

Альтернативно, можно скопировать желаемый apphost в соответствующие директории `<dotnet_root>/packs` и кэша NuGet. Можно также собрать пакеты NuGet локально и настроить приложение на использование собранных пакетов через файл **NuGet.config** и элемент `KnownAppHostPack`.

# Указать локальный .NET root

Для приложения, зависящего от фреймворка ([framework-dependent application](https://learn.microsoft.com/dotnet/core/deploying/#publish-framework-dependent)), вы можете установить переменную окружения `DOTNET_ROOT`, чтобы указать на локальную структуру .NET.

Тесты [библиотек](../libraries/testing.md) создают и используют такую структуру на основе вашей локальной сборки runtime, хоста и библиотек в рамках подмножества `libs.pretest`. Чтобы использовать эту структуру, укажите `DOTNET_ROOT=<repo_root>/artifacts/bin/testhost/net<version>-<os>-<configuration>-<arch>`, а затем запустите .NET приложение.
