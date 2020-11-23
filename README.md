# Агрегатор IT мероприятий

Данный файл посвящен обзору технической стороны реализации проекта "Агрегатор IT мероприятий", реализуемого в рамках учебной практики 2020/21 года.

# Архитектура проекта:

Для реализации проекта мы решили использовать микросервисную архитектуру. Причин немало: из-за независимости микросервисов удобнее разделять работу между несколькими разработчиками, легко проводить тестирование. Благодаря библиотеке `nameko` на `python` код таких сервисов получается простой и понятный, а возможностей, предоставляемых библиотекой, вполне достаточно, чтобы разрабатывать крупные проекты. Давайте более подробно рассмотрим какие сервисы мы разработали.

## Погрузчик данных

В архитектуре можно выделить крупный модуль, связанный с подтягиванием данных из внешних источников, преобразованием их в удобную форму и сохранением в базе данных. Рассмотрим подробнее архитектуру этого модуля. Мы привлекаем данные с различных информационных источников с помощью микросервисов-краулеров (`it_events_crawler`, `it_world_crawler`, `softline_crawler`). Вследствие независимости этих краулеров, необходимо как-то инициализировать их работу и сохранять данные в некоторый промежуточный сервис-семафор. Этим занимается микросервис `raw_events_collector`. По таймеру (раз в сутки) он вызывает сбор "сырых", т.е. представленных в неокончательном виде, мероприятий с информационных ресурсов, получает патч событий и отправляет эти данные в сервис `primary_raw_events_handler`. Здесь происходит обработка сырых мероприятий: приведение их к единообразному виду и удаление повторяющихся мероприятий (с разных ресурсов могло поступить одно и то же мероприятие). Далее сервис посылает патч с мероприятиями в сервис `event_das` отвечающий за хранение и логику работы с мероприятиями. Для построения рекомендательной системы необходимо снабдить мероприятия тэгами, для чего `event_das` отсылает мероприятия сервису `event_theme_analyzer` и получает обратно мероприятия, размеченные тэгами. Таким образом в результате работы данного модуля мы получаем актуальные унифицированные размеченные мероприятия, находящиеся в нашей базе данных.


## Взаимодействие с пользователями

Дальше микросервисов становится только больше: нам еще необходимо реализовать авторизацию, систему оценивания (лайки, избранное), систему ранжирования и многое другое. Поэтому вполне разумно сделать еще и сервис, который будет медиатором для всех остальных, то есть который по мере необходимости будет посылать запросы к другим микросервисам. Это сервис API `gateway`. Здесь также происходит взаимодействие с front-частью: принимаются и посылаются http-запросы. По мере поступления того или иного запроса `gateway` и будет вызывать функции различных сервисов. Посмотрим, какие сервисы напрямую связаны с `gateway`.

