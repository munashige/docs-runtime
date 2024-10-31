# Codespaces

Codespaces позволяет вести разработку в Docker-контейнере, который работает в облаке. Вы можете использовать веб-версию VS Code или установленный VS Code с помощью расширения [GitHub Codespaces](https://marketplace.visualstudio.com/items?itemName=GitHub.codespaces).


## Создание Codespaces

Репозиторий работает с nightly GitHub Action для сборки кода. Это позволяет сразу начать разработку и тестирование после создания Codespaces, не дожидаясь сборки всего репозитория. После создания машины, репозиторий будет собран с использованием кода в 6 утра по всемирному координированному времени (UTC).

1. Из корневой страницы репозитория нажмите кнопку *<> Code*, а затем выберите вкладку *Codespaces*.

2. В правом верхнем углу вкладки *Codespaces* нажмите *...* и затем *+ New with options*.

    ![Configure and create codespace](https://docs.github.com/assets/cb-49317/images/help/codespaces/default-machine-type.png)

3. Выберите, какую конфигурацию Dev контейнера необходимо использовать.

    ![Dev container configuration](./codespace-dev-container-configuration.png)

     - Для работы с `библиотеками` (*libraries*) выберите `.devcontainer/libraries/devcontainer.json`.
     - Для работы с  `WASM` выберите `.devcontainer/wasm/devcontainer.json`.

4. Выберите Machine Type. Рекомендуется выбрать опцию `4-core` или выше.

    ![Codespace machine size](codespace-machine-size.png)

Если эти инструкции устарели, используйте [документацию Github](https://docs.github.com/codespaces/developing-in-codespaces/creating-a-codespace#creating-a-codespace) для созданию нового Codespace.

## Обновление конфигурации Codespaces

Конфигурация Codespaces распределена следующим образом:

1. Папка `.devcontainer` содержит подпапки для каждого сценария разработки:
    - *Библиотеки*: Используется разработчиками, работающими с `src/libraries`.
    - *Wasm*: Используется разработчиками, работающими c `browser-wasm`.
    - *Скрипты*: Содержит скрипты, которые выполняются во время создания Сodespaces. 
    Здесь находится команда сборки, которая собирает весь репозиторий.

2. Каждая вышеупомянутая подпапка содержит следующие файлы:
    * Файл `devcontainer.json`, который настраивает Codespaces и содержит настройки для VS Code/ окружения.
    * Файл для создания Docker-образа.
3. Github Action. Настройте GitHub Action, следуя [этой инструкции](https://docs.github.com/codespaces/prebuilding-your-codespaces/configuring-prebuilds).

Чтобы протестировать изменения в файлах `.devcontainer`, необходимо следовать процессу, описанному в [документации Github](https://docs.github.com/codespaces/customizing-your-codespace/configuring-codespaces-for-your-project#applying-changes-to-your-configuration). Таким образом, можно пересобрать Codespaces перед созданием PR.

## Тестирование изменений

Чтобы протестировать ваши изменения, вы можете запустить [Prebuilds Action для Codespaces](https://github.com/dotnet/runtime/actions/workflows/codespaces/create_codespaces_prebuilds) с вашей форки и с нужной ветки.