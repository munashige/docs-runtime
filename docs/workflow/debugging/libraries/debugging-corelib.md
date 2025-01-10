Отладка System.Private.CoreLib
==============================

`System.Console.Write`/`System.Console.WriteLine` не могут быть использованы в `System.Private.CoreLib`. Вместо этого используйте `Internal.Console.Write`, чтобы добавить временное логирование для отладки в стиле printf.

### Android
Логи можно найти через сгенерированный лог `Android Debug Bridge` или просмотреть напрямую через `ADB logcat`.
