## Инструкция по обновлению версии modx на сайте, работающем с расширением gitmodx

Особенностью расширения [gitmodx](http://github.com/azernov/gitmodx) является то, что при его
установке происходит замена нескольких строк кода в файлах index.php ядра modx, поэтому при обновлении modx
следует учесть то, что эти файлы будут возвращены в свое изначальное состояние.

После обновления сайта в соответствии со стандартной инструкцией modx, необходимо изменить следующие файлы:

1. www/index.php

    Замените пути подключения класса modx в районе 27 строки (может менять от версии к версии) на:
    ```
    MODX_CORE_PATH . "components/gitmodx/model/gitmodx/gitmodx.class.php"
    ```

    Это должно выглядеть так:
    ```
    ...
    /* include the modX class */
    if (!@include_once (MODX_CORE_PATH . "components/gitmodx/model/gitmodx/gitmodx.class.php")) {
        $errorMessage = 'Site temporarily unavailable';
    ...
    ```

    И затем в районе 39 строки замените `new modX` на `new gitModx`:
    ```
    $modx = new gitModx();
    ```

2. Аналогичные замены в файле www/connectors/index.php

    Строки 24-26:
    ```php
    ...
    if (!include_once(MODX_CORE_PATH . 'components/gitmodx/model/gitmodx/gitmodx.class.php')) die();

    $modx = new gitModX('', array(xPDO::OPT_CONN_INIT => array(xPDO::OPT_CONN_MUTABLE => true)));
    ...
    ```

3. Аналогичные замены в файле www/manager/index.php

    Строка 37:
    ```
    ...
    if (!(include_once MODX_CORE_PATH . 'components/gitmodx/model/gitmodx/gitmodx.class.php')) {
    ...
    ```

    Строка 43:
    ```
    $modx= new gitModx('', array(xPDO::OPT_CONN_INIT => array(xPDO::OPT_CONN_MUTABLE => true)));
    ```



## Инструкция по обновлению пакета pdoTools до более новой версии на сайте, работающем с расширением gitmodx

При обновлении пакета pdoTools происходит замена двух системных настроек: `parser_class` и `parser_class_path` в
таблице `[prefix]_system_settings`.

Поэтому, после обновления этого компонента, необходимо заменить значение этих настроек в таблице `[prefix]_system_settings`
на прежние значения:

```
UPDATE [вставьте префикс таблиц сюда]_system_settings SET `value`='gitModParser' WHERE `key`='parser_class';
UPDATE [вставьте префикс таблиц сюда]_system_settings SET `value`='{core_path}components/gitmodx/model/gitmodx/' WHERE `key`='parser_class_path';
```

И затем не забудьте очистить кэш сайта, удалив содержимое папки www/core/cache/