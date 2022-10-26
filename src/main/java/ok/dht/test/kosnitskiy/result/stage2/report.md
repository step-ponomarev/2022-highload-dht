ENVIRONMENT-DESCRIPTION
Тестирование производилось на 6-ядерном, 6-поточном процессоре. По этой причине было решено использовать 4 потока под
при тестировании wrk (и 64 connections, как сказано в задании), чтобы еще два ядра были не задействованы и использовались
под нужны системы, профайлера и так далее
после более адекватного взгляда на вопрос стало понятно, что количество тредов необходимо брать схожим с количеством 
логических ядер у процессора, а возможно лучше даже схожим с количеством физических (к сожалению не удалось проверить 
этот момент, так как у моего ноутбука количество логических и физических ядер одинаково), в нашем случае, правла, 
имеет смысл брать поменьше из-за сторонней нагрузки на ядра, такой как профайлер и врк, который мы прямо на нашей 
машине и запускаем, однако так как в общем случае для реальных юзкейсов наше приложение скорее всего будет целком 
занимать ноду, которая будет отдельным сервером, имеет смысл брать количество тредов по количеству логических ядер 
у процессора, а не константное число, которое я написал до этого
Размер очереди было решено тоже сделать зависимым от числа использующихся тредов, так как, скажем, для 1-ядерного
процессора очередь в 256 явно будет излишней. Для своих же 6 ядер я установил, что при 256 элементах в очереди мы
получаем наиболее оптимальное число элементов, так как при уменьшении у нас начинает падать максимальное количество
запросов в секунду, не сильно при этом влияя на скорость

GET-DESCRIPTION
Для тестов брался наихудший случай - настоящие ключи, перемешанные с несуществующими. В таких условиях базе зачастую
приходится проходиться сразу по всем файлам, проверяя наличие ключа в каждом из них, а так же не получается ничего
закешировать
UPD:
Из-за подозрения на то, что высокая производительность вызвана реквестами с ошибками 404 (подозрения, что они могли быть
на самом деле таймауты). Взял более благоприятный случай - ключи, которые я положил в дао, а потом закрыл сверху еще
полтора гигами записей с другими ключами. Это куда более благоприятная ситуация для нашей базы данных, так как в среднем
мы будем проходить только через треть файлов, а при несуществующих ключах проходили через всю базу данных. По этой причине
максимальное количество единовременных запросов выросло еще примерно на 10-20%. Стоит ответить, что получение ключей
средней степени свежести - очень распростарненный юзкейс в отличие от большого количества несуществующих, так что такое
тестирование даже более обосновано

PUT-DESCRIPTION
Для PUT мы генерировали разные ключи, чтобы гарантировать, что размер DAO будет увеличиваться и не будет 
перезаписываться одно и тоже значение кучу раз.
Так же мы стали генерировать строки разной длинны чтобы несколько усложить бинарный поиск по получившимся файлам


WRK-GET
wrk -d 10 -t 4 -c 64 -R 600 -s get-random.lua --latency "http://localhost:19234/v0/entity?id=1"
Running 10s test @ http://localhost:19234/v0/entity?id=1
4 threads and 64 connections
Thread Stats   Avg      Stdev     Max   +/- Stdev
Latency     2.11ms  819.45us  24.27ms   80.09%
Req/Sec       -nan      -nan   0.00      0.00%
Latency Distribution (HdrHistogram - Recorded Latency)
50.000%    2.04ms
75.000%    2.53ms
90.000%    2.96ms
99.000%    3.79ms
99.900%    4.57ms
99.990%   22.33ms
99.999%   24.29ms
100.000%   24.29ms

wrk -d 20s -t 4 -c 64 -R 6000 -s get-random.lua --latency "http://localhost:19234"
Running 20s test @ http://localhost:19234
4 threads and 64 connections
Thread Stats   Avg      Stdev     Max   +/- Stdev
Latency     2.56ms    1.77ms  14.36ms   77.81%
Req/Sec     1.57k   271.02     3.00k    76.55%
Latency Distribution (HdrHistogram - Recorded Latency)
50.000%    2.08ms
75.000%    3.31ms
90.000%    4.95ms
99.000%    8.74ms
99.900%   11.32ms
99.990%   13.03ms
99.999%   13.81ms
100.000%   14.37ms

Последовательные тесты показали значительное повышение производительности по сравнению с однопоточной реализацией,
используемой до этого. 
Для гета мы можем увидеть, что даже для 600 запросов в сек, что было максимумом для прошлой реализации, можно заметить
уменьшения задержек. Неадекватно высокие задержки вызникают только на 99.9 перцентиле, что можно списать на шалости
сборщика мусора
Итого мы получили улучшение в 10 раз по сравнению с прошлой реализацией, при этом количество потоков, которое мы используем
увеличилось всего в 6 раз. Получается, мы не только получили прирост на производительность в многопотоке сугубо из-за
числа потоков, но и увеличили КПД этих потоков в целом! Это просто уберсладко!
Можно заметить, что на 99-ом перцентиле время выполнения в 3 раза выше среднего, а на 99.99 - в 5 раз.
Это связано с тем, что некоторые GET запросы сильно сложнее других, ибо им приходится ходить куда глубже внутрь по цепочке
файлов, чтобы найти нужный ключ, а так же тем, что GET запросы аллоцируют много памяти а потому особо сильно триггерять
Garbage Collector.
Это доказывается еще и тем, что я ставил размер очереди очень маленьким (1 единица, фактически без очереди),
таким образом очередь не образовывалась, а все новые запросы просто отклонялись, и даже это не помогало нам приблизить 
время выполнения МAX запросов к среднему времени, так что предположение, что MAX время вызвано большим размером очереди
и толкучкой на выполнение, что там образуется, не подтвердилось.
Если вернуться к GC, то у меня есть теория, что нам могло бы с этой проблемой помочь создание системы запасной оперативной
памяти: когда garbage collector планирует чистить память - он сообщает об этой нашей программе, она начинает все аллоцировать
в запасной части памяти, а Garbage collector в это время в не используемой программой потоке, аккуратненько чистит эту
основную память потихоньку, потом они меняются местами и цикл повторяется.
Однако тут возникает новый вопрос - не лучше ли будет всю эту память и лишний поток просто заиспользовать для улучшения
средний производительности - ну тут все зависит от того что нам важнее: средняя скорость чуть повыше, или MAX значение
сильно поближе к среднему

WRK-PUT
wrk -d 10 -t 4 -c 64 -R 20000 -s put-different.lua --latency "http://localhost:19234"
Running 10s test @ http://localhost:19234
4 threads and 64 connections
Thread Stats   Avg      Stdev     Max   +/- Stdev
Latency     1.07ms    0.96ms  24.61ms   96.91%
Req/Sec       -nan      -nan   0.00      0.00%
Latency Distribution (HdrHistogram - Recorded Latency)
50.000%    0.99ms
75.000%    1.34ms
90.000%    1.64ms
99.000%    3.18ms
99.900%   17.74ms
99.990%   22.43ms
99.999%   23.39ms
100.000%   24.62ms

wrk -d 20s -t 4 -c 64 -R 115000 -s put-different.lua --latency "http://localhost:19234"
Running 20s test @ http://localhost:19234
4 threads and 64 connections
Thread Stats   Avg      Stdev     Max   +/- Stdev
Latency     2.40ms    2.39ms  32.62ms   88.66%
Req/Sec    30.26k     6.23k   63.60k    75.15%
Latency Distribution (HdrHistogram - Recorded Latency)
50.000%    1.67ms
75.000%    2.88ms
90.000%    5.13ms
99.000%   12.07ms
99.900%   20.00ms
99.990%   28.19ms
99.999%   31.36ms
100.000%   32.64ms


Наш потолок нагрузки поднялся с 20к до 115к запросов в секунду! Это выше на треть по сравнению с результатами, полученными
до исправления второго стейджа. Скорее всего тут очень положительно повлияло переписывание handleDefault логики, а так
же правильная настройка количества потоков. Это особенно сильно повлияло на PUT, так как сама по себе эта операция очень 
легкая, так что мои слова ниже подтвердились и мы смогли прорефлексировать над ними и оптимизировать еще сильнее!
Так же можно заметить, что среднее время выполнения запроса и время исполнения на 99.9 перцентиле поднялись, даже для
аналогичных нагрузках (20к Stage1 отрабатывает лучше, чем 20к Stage2). Это связано с тем, что PUT - это все же очень легкая
операция для нашего ДАО, поэтому тут накладные расходы на thread executor и синхронизацию многопотока видны как никогда:
в каком-то плане быстрее селектору дождаться пока быстрый пут сделается, чем отдавать запрос в пулл на обработку, а потом
его еще от туда забирать. Однако мы готовы на такую небольшую жертву, ведь при больших нагрузках (115к) наш подход все еще
отрабатывает достойно, когда как Stage1 уже на 25к полностью сдувается.
Про максимальное время выполнение рассуждения аналогичны с GET'ами, однако стоит отметить, что здесь GC, ибо запросы очень
"легкие", меняет максимальное время исполнения очень сильно по сравнению с средним временем исполнения. Так что такая
оптимизация в случае PUT запросов будет еще пользительней GET

Так же можно отметить, что прогрев действительно очень важен как PUT, так и для GET запросов.
1000 непрогретых GET запросов имеет ниже среднюю скорость выполнения как 2000 прогретых запросов
Аналогично 20000 непрогретых PUT аналогичны по скорости 30000 прогретым PUT запросам.
Это, естественно, показывает нам, что Джит реально не зря ресурсы наши жрет, но еще и действительно оптимизирует наш код
на лету, понимая какая нагрузка этот наш код будет ожидать в дальнейшем


PROFILER-GET
Аллокации - на аллкациях GET видно сильное улучшение. Раньше аллокации шли в шахматном порядке - теперь весь график
равномерно красный. Хотя это, бесспорно, можно списать на погрешность семплирования, мне кажется, бесспрорно так же
можно сказать, что скорее всего процессор стал нагружен более равномерно.
Про CPU можно сказать аналогичную вещь - теперь у нас куда меньше простоя на CPU, мы всегда очень яркие, очень красные,
очень сладкие, однако явно видны места, когда работал JIT, а значит в остальное время можно считать, что загрузка
процессора была не полная и есть немного пространства для улучшения
"По графику локов можно заметить, что возникают они только в самом начале - когда код еще не прогрет и джит шалит, и в
конце, когда нагрузка максимальная. Возможно можно судить, что появление локов - это показатель того, что все у нас 
становится очень плохо и надо что-то с этим делать, например начинать троттлить и пропускать запросы, или не принимать
новые запросы, отпавлять их на другой, менее загруженный инстанс нашей базы данных" - после улучшения и лучшей подгонки
настроек тредов локи при GET исчезли, можно сказать, совсем, скорее всего это связано с правильным количеством работающих
одновременно тредов, когда как при грамадном их количестве - 512, они могли постоянно друг друга блокать во время взятия
новых запросов из очереди, мешая друг другу.

PROFILER-PUT
Аллокации - справедливо то же, что было сказано про GET, но можно еще отметить, что аллокации в PUT были еще более редкими
до этого, чем аллокации в GET, сейчас же они постоянные, это очень славно показывает, что запросы больше не ждут друг друга,
а выполняются синхронно, сладота! Так же на графике выделения памяти PUT запросов видны яркие красные точки - это моменты
флашей на диск, а так же выделения памяти под новые inMemory storages.
С CPU все примерно так же, как и с GET, тоже стал куда более равномерно нагруженный график. Очень много инергии и процессора
забирают SYSCALL'ы, возможно их количество в путах можно оптимизировать
Локи у путов происходят намного чаще, чем у Гетов, это связано с тем, что пут - куда более быстрая операция и тредам 
легче нарваться друг на друга, и таким образом залокать друг дружку. Возможно стоит подумать над созданием не блокирующей
очереди, ибо она сильно поможет PUT'ам стать менее локающими.

Спасибо за прочтение!