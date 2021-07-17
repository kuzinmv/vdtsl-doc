# Инструкция по деплою и описание процессов

1. Разверните стек в портейнере используя этот файл docker-compose.prod.yml
2. В данном стеке описаны 4 контейнера nginx, postgresql, nodejs, rabbit 
   В postgresql хранятся описание районов и регионов для фильтрации точек.
   node основной рабочй контейнер с 2 пулами процессов см ниже,
   Nginx - прокси для апстрим процессов, rabbit - для межпроцессного взаимодействия.
3. Для nodejs контейнера задана переменная окружения - адрес сервера куда будут отправляться данные.   
  `DESTINATION_URL: 'http://92.124.155.145:8083/api/traffic/aggregate'`   
   Для тестовых запусков можно использовать любой сервер (https://webhook.site), например `DESTINATION_URL: 'https://webhook.site/76a7dc54-5c04-4081-b97d-09255071fd67'`
   
4. При старте nodejs контейнера производятся следующие операции
   * 4.0. Миграции схемы бд
   * 4.1. Загрузка метаданных 3х регионов (Омск, Спб, Краснодар)
   * 4.2. Загрузка 15-20 моделей (1..5 перекрестка) по каждому региону
   * 4.3. В супервизоре запускается пул из 4х процессов `mr-area-worker` в каждый из этих процессов загружается все территории и их границы. 
   На эти 4 процесса осуществляется первичное проксирование всех точек. Пример запроса:
   
   `curl http://domain.dom/api/traffic/v1/probedata -H "Content-Type:application/json" -X POST -d '{"provider":"STARLINE_EU_PB","pp":[{"id":"1","x":-76.9906933605,"y":38.9910785259,"s":51,"h":104,"t":"2020-03-04T11:19:00"}]}'`
   
   Все отфильтрованные точки передаются в шину, которую слушает пул воркеров второго типа: `mr-cluster-worker`.
   Каждый воркер обслуживает определенный регион и ряд конкретных моделей от 5 до 10 моделей
   
   Список процессов nodejs контейнера в результате успешного запуска.
   ```
   root	509849	00:00:00	sh /usr/src/app/runupstream.sh
   root	517808	00:00:00	/usr/bin/python /usr/bin/supervisord -c /etc/supervisor/supervisor.conf
   root	517810	00:00:00	./mr-cluster-worker omsk 55000001,55000002,55000003,55000004,55000005,55000006,55000007
   root	517811	00:00:01	./mr-cluster-worker omsk 55000008,55000009,55000010,55000011,55000012,55000013,55000014
   root	517812	00:00:01	./mr-cluster-worker saint-petersburg 78000001,78000002,78000003,78000004,78000005
   root	517813	00:00:00	./mr-cluster-worker krasnodar 23000001,23000002,23000003,23000004,23000005,23000006,23000007,23000008,23000009
   root	517814	00:00:01	./mr-cluster-worker saint-petersburg 78000011,78000012,78000013,78000014,78000015,78000016,78000017,78000018,78000019,78000020
   root	517815	00:00:00	./mr-cluster-worker krasnodar 23000010,23000011,23000012,23000013,23000014,23000015,23000016,23000017,23000018
   root	517826	00:00:00	./mr-area-worker 3002
   root	517832	00:00:00	./mr-area-worker 3003
   root	517838	00:00:00	./mr-area-worker 3001
   root	517844	00:00:00	./mr-area-worker 3004
   root	517855	00:00:00	./mr-cluster-worker saint-petersburg 78000006,78000007,78000008,78000009,78000010
   root	517938	00:00:00	./mr-area-worker 3000

   ```
    
    Каждый процесс типа `mr-cluster-worker` осуществляет вторичную фильтрацию точек, только в границах тех моделей, которые он обслуживает и
    накапливая точки создает из них треки и привязывает к объектам, обслуживаемой модели.
    Далее на основании результата привязки собирает ряд "агрегатов" для отправки их на внешний сервер, буфферизирует их 
    и по достижении лимита посылает запрос на внешний сервер.
    
5. Для нагрузочного тестирования и просто работоспособности системы была написана специальная утилита, которая парсит csv 
   файл и по 2000 точек в одном запросе отправляет напрямую в этот процесс `./mr-area-worker 3000`
   В сборке присутствует 2 csv файла точек из региона Спб за 26-05-2021 по одному часу 00 и 12 по UTC 
   
   Внутри nodejs контейнера
   `root@6dff1a3d673a:/usr/src/app# node push-csvmock.js mock/area1/8-2021052600.csv `    

   Результат запуска скрипта.
   
   ```json
    {"total":1636}statusCode: 200
    ...
   
    ...
    {"total":1636}statusCode: 200
    {"total":1570}Parsed 185754 rows   // *Файл (mock/area1/8-2021052612.csv) за 12-й час в 10 раз больше ~ 17000000 строк
    ```
    
   На внейшний сервер будет отправлено 29 http запросов, в каждом из которых будет массив объектов следующего вида:
   ```json
   {
       "t": {
         "wk": "2017160733-10f,8579086264-10f", // строка идентификаторов точек из модели
         "a_ids": [                             // массив идентификаторов точек из модели
           "2017160733-10f",
           "252131127-10f",
           "252131127-02f",
           "4217840842-00f",
           "1686646398-02f",
           "1686646398-11f",
           "2017160715-10f"
         ],
         "c_id": 1,                              // Идентификатор перекрестка int
         "m_id": 78000010,                       // ID модели int
         "t": "2021-05-26T00:00:32.000Z",        // Таймстамп string
         "c": 1                                  // всегда 1
       },
       "n": [                                    // массив размером [0..2]
         {
           "a_id": "2017160733-10f",             // Идентификатор точки модели string
           "s": 2,                               // тип byte
           "t": "2021-05-26T00:01:14.888Z"       // Таймстамп string
         }
       ],
       "f": [                                    // массив размером [0..2]
         {
           "a_id": "2017160733-10f",             // Идентификатор точки модели string
           "q_v": 0,                             // тип byte
           "t": "2021-05-26T00:01:14.888Z"       // Таймстамп string
         }
       ],
       "q": [                                    // массив размером [0..2]
         {
           "a_id": "2017160733-10f",            // Идентификатор точки модели string
           "s": 1,                              // тип byte 
           "q_m": 15,                           // тип byte   
           "d": 25,                             // тип byte 
           "dt": 230,                           // тип byte 
           "tt": 43                             // тип byte 
         }
       ]
     }
    ```

6. Результаты сбора и обработки агрегатов сервером назначения можно увидеть по адресу  [http://92.124.155.145:8083]. 
    Необходимо выбрать город и модель и день (уже загружены данные по Спб за 26-05-2021), затем нажав на кнопки "Программа", "Поток" дождаться ответа сервера.
    "Программа" - может работать достаточно долго до 1 минуты. 
    Далее кликая на прямоугольники(иногда 2 раза надо, бывает баг с отрисовкой) или синие точки наблюдать результаты.


