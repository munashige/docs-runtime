# Workloads

Workloads, основанные на пакетах в `artifacts`, могут быть установлены и использованы для тестирования.

* Для этого требуется установка определённой версии SDK (`$(SdkVersionForWorkloadTesting)`) в `artifacts/bin/dotnet-workload`.
* Также нужно установить манифест для workload, указанный в `@(WorkloadIdForTesting)`.
* После этого устанавливаются пакеты workload.
* Пакеты или манифесты, не сгенерированные runtime, восстанавливаются из nuget.
* SDK устанавливается с помощью цели `ProvisionSdkForWorkloadTesting`, а workload — с помощью `InstallWorkloadUsingArtifacts`, используя задачу `InstallWorkloadFromArtifacts` для `@(WorkloadIdForTesting)`.

Пример для wasm:

```xml
<WorkloadIdForTesting Include="wasm-tools"
                      Name="microsoft.net.workload.mono.toolchain"
                      ManifestName="Microsoft.NET.Workload.Mono.ToolChain"
                      Version="$(PackageVersion)"
                      VersionBand="$(SdkBandVersion)" />
```

В настоящее время используется только в `src/tests/BuildWasmApps/Wasm.Build.Tests`.

## Несколько пакетов runtime

Workload зависит от трёх пакетов (packs) runtime  — однопоточного, многопоточного и для трассировки производительности. Например, при работе с локальной сборкой runtime и многопоточным вариантом, установка workload завершится ошибкой из-за отсутствия nuget-пакетов runtime для двух других вариантов.

Для сборок вне-CI создается один и тот же nuget-пакет runtime, который называется по-разному. Таким образом, вы получаете все nuget-пакеты, с одинаковым содержанием, но с разными именами.

Если у вас есть все необходимые nuget-пакеты и вы хотите избежать вышеописанных процедур, установите `WasmSkipMissingRuntimeBuild=true`.

## Ограничения

Пакет кросс-компилятора собирается вручную с помощью `InstallWorkloadUsingArtifacts`.
