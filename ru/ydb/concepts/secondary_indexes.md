# Вторичные индексы

Этот раздел посвящен поддержке вторичных индексов в YDB.

В YDB автоматически создается индекс по первичному ключу, поэтому выборки с условием по первичному ключу всегда выполняются эффективно, затрагивая только требуемые строки. Выборка с условием, наложенным на одну или несколько неключевых колонок, как правило, приводит к полному сканированию таблицы. Для того, чтобы такие выборки были эффективными, нужно использовать вторичные индексы.

В текущей версии реализован глобальный вторичный индекс. Каждый индекс представляет собой скрытую таблицу, которая обновляется транзакционно при изменении основной таблицы. Когда пользователь присылает SQL-запрос на вставку/модификацию/удаление данных, база данных прозрачно для пользователя формирует команды на модификацию индексной таблицы на стадии компиляции запроса. Вторичных индексов у таблицы может быть несколько, и индекс может включать несколько колонок (при этом важен порядок указания колонок в индексе). А также одна колонка может состоять в нескольких индексах и даже быть одновременно частью первичного ключа и вторичного индекса.


##### Особености и ограничения
- Для одношардовой таблицы без индекса есть возможность выполнить транзакцию (как читающую, так и слепую запись) без стадии планирования и таким образом ощутимо сократить задержки. Для записи в таблицу с индексом такая оптимизация не возможна, из-за необходимости обеспечивать согласованность данных — это неизбежная плата за возможность производить быстрое чтение без фулскана. [Подробнее про механизм транзакций](transactions#distributed-tx).
- `UPSERT` в таблицу со вторичным индексом перестал быть слепым. Из-за этого возникает ограничение на один `UPSERT` в транзакции над таблицей.

#### Онлайн построение вторичного индекса

В YDB доступно построение вторичного индекса без остановки обслуживания, а так же удаление существующего вторичного индекса.

Для одной таблицы возможно построение только одного индекса за раз.

В целом онлайн построение состоит из следующих стадий:
 1. Взятие снапшота таблицы с данными, создание таблицы индекса с пометкой доступности для записи. Cразу после этой стадии пишущие транзакции становятся распределенными, пишут соответственно в основную таблицу и индекс. Для пользователя индекс пока не доступен.
 2. Запуск процесса чтения снапшота основной таблицы и записи в индекс. Причем предприняты меры для разрешения ситуации, когда обновления на стадии 1 меняют данные записанные на стадии 2 (другими словами реализована "запись в прошлое").
 3. Публикация результата, удаление снапшота. Индекс становится готов к использованию.

Возможные влияния на пользовательские транзакции:
* Сразу после первого этапа постоения, из за того, что транзакции стали распределенными наблюдается увеличение задержек.
* Во время заливки данных активно работает автосплит индексной таблицы, в ходе которого индексные шарды сплитятся для получения оптимального шардирования. Возможен повышенный фон ошибок OVERLOADED.

Скорость заливки данных подобрана таким образом, чтоб минимизировать влияние процесса заливки на пользовательские транзакции. Таким образом рекомендуется запускать online построение во время минимальной нагрузки для получения максимальной скорости сходимости процесса.


Более подробно примеры рассмотрены в разделе [Рекомендации - вторичные индексы](../best_practices/secondary_indexes.md).

Примеры запуска операции построения индекса с использованием cli представлены в разделе [Использование консольного клиента {{ ydb-short-name }}](../quickstart/examples-ydb-cli#zapusk-operacii-dobavleniya-vtorichnogo-indeksa).

