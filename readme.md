# Агрегатор служб доставок

## Цель проекта

Обеспечить возможность доставки товара от продавца к покупателю при помощи
любых сторонних логистических компаний, предоставляющих API для автоматизации.

## Глоссарий
**Агрегатор служб доставок** (delivery aggregator) — собственно, сам проектируемый сервис.

**Служба доставки** (delivery service) — подключаемая к агрегатору логистическая компания, предоставляющая API для автоматизации (получение возможных вариантов доставки, размещение отправления, определение статуса отправления).

**Продавец** — продавец товара в системе Авито, он же отправитель в терминах доставки.

**Покупатель** — покупатель товара в системе Авито, он же получатель в терминах доставки.

**Заказ** (order) — покупка в системе Авито.

**Отправление** (delivery) — отправление заказа от продавца к покупателю при помощи выбранной покупателем службы доставки из возможных вариантов.

**Статус отправления** — характеристика отправления из набора возможных, однозначно определяющая текущее состояние отправления.

## Требования
  1. Предоставить API для определения на основании параметров товара и местоположения продавца и покупателя:
      - возможности доставки из населённого пункта отправителя в населённый пункт получателя;
      - вариантов доставки разных типов (курьерская доставка, доставка в постомат, доставка в пункт выдачи заказов);
      - рассчёта стоимости доставки.
  2. Предоставить API для размещения отправления в подключенных службах доставки на основании выбранного покупателем варианта.
  3. Уведомлять заинтересованные сервисы о событиях, происходящих с отправлением (см. [Жизненный цикл отправления](./readme.md#жизненный-цикл-отправления))
  4. Обеспечивать всевозможную защиту от сбоев на стороне службы доставки:
      - при невозможности создания отправления за несколько попыток отключать службу доставки;
      - при недоступности API службы доставки отключать службу доставки, при доступности — включать обратно;
      - при нарушении сроков доставки автоматически устанавливать статус "сроки доставки изменились", после истечения максимального времени ожидания переводить отправление в статус "товар потерян или поврежден".
  5. Давать возможность конфигурировать:
      - диапазоны времени проверки товаров отдельно для каждой службы доставки (если есть, по умолчанию использовать данные из API службы доставки);
      - максимальное время ожидания отправления в промежуточных статусах;
      - количество попыток создания отправления в службе доставки при неудаче и интервалы времени между ними.
  6. Предоставлять метрики за произвольный интервал времени для каждой из служб доставки по отдельности (или всех вместе):
      - количество созданных отправлений;
      - количество отправлений, не созданных по вине службы доставки;
      - количество отправлений, не созданных по вине агрегатора;
      - количество отправлений в финальных состояниях:
        - "получатель забрал товар"
        - "товар не был принят службой доставки"
        - "товар потерян или поврежден"
        - "отправитель забрал товар"
      - количество отправлений, "подвисших" в промежуточных состояниях до истечения срока максимального ожидания.
      Для всех метрик кроме количества можно считать также и стоимость товара, чтобы лучше понимать финансовые объёмы обслуживаемых отправлений.

## Общая схема взаимодействия компонентов

![Общая схема](./img/common.png)

## Жизненный цикл отправления

![Жизненный цикл](./img/statuses.png)

0. Неудачная попытка создания отправления, статус `pending`. Не удалось создать отправление в API службы доставки. Запланирована повторная попытка.
00. Лимит количества попыток исчерпан, статус `failed`.
1. Cоздание отправления, статус `new`. Данные о получателе и отправителе, параметры товара и выбранного способа доставки переданы в службу доставки. Идентификатор покупки в Авито связан с идентификатором отправления на стороне службы доставки.
2. Товар был принят службой доставки, статус `scheduled`. Отправитель передал товар службе доставки.
3. **NEW** Товар не был принят службой доставки (в т.ч. по истечении времени ожидания). *Как нужно реагировать на такие случаи? Пытаться сделать повторную доставку? Кто понесёт расходы за неудачную или повторную попытку в случае, например, курьерской доставки?*
4. Товар был доставлен до получателя, статус `delivered_to_recipient`.
5. Получатель забрал товар (после успешной проверки), статус `completed`.
6. Получатель вернул товар службе доставки (после проверки), статус `failed_check`.
7. Товар приехал обратно к отправителю, статус `delivered_to_sender`. *Возможно, что нужно разделять, по чьей вине товар был отправлен обратно (товар не прошёл проверку, или покупатель его вообще не забрал), чтобы понимать, нужно ли возвращать деньги за доставку покупателю?*
8. Отправитель забрал товар (убедился, что товар не был повреждён), статус `refunded`.
9. Товар потерян или поврежден, статус `lost_or_damaged`.
10. Cроки доставки изменились, статус `delayed`.

## Предполагаемая архитектура

![Предполагаемая архитектура](./img/architecture.png)

### Компоненты

#### Delivery services registry

#### Delivery options service

#### Delivery service

#### Delivery status updater

#### Adapters to APIs of delivery services

## Мониторинг жизнеспособности системы

Критически важными процессами для работы системы являются:
1. Работоспособность API агрегатора: рассчёта вариантов доставки и создания отправлений.
Их недоступность можно мониторить по статусам (а также времени) ответов на API gateway.
2. Доступность и корректная работа API служб доставки. Проверкой их доступности занимается отдельный компонент — реестр служб доставки. Для мониторинга ошибок во взаимодействии с бизнес-логикой API служб доставки, нужно вести лог ошибок на стороне каждого компонента и оперативно реагировать на появление таких ошибок. Количество неудачных отправлений в статусе `failed` также говорит о проблеме на стороне службы доставки.
3. Работа компонента по обновлению статусов отправлений менее критичная, потому что происходит в фоне для пользователя, однако для корректной работы всей системы также необходима. Его можно контролировать по отсутствию событий смены статуса отправлений за длительный период времени.
4. Для разбирательств со службами доставки по поводу пропавших и повреждённых отправлений нужна "админка" для сотрудников поддержки, где можно будет посмотреть детальную информацию об отправлении. Возможно, эта админка будет вообще про заказы, где можно делать возврат средств, а информацию об отправлении она сможет получать через API чтения отправлений.

## Планирование разработки

Для разработки сервиса нужны бэкенд-разработчики в количестве от одного и более.
Кажется, что если на каждый компонент будет выделенный разработчик, это будет наиболее оптимально,
как с точки зрения скорости разработки, так и с точки зрения поддержки и развития сервиса.
Например, один разработчик занимается реестром служб доставок и разработкой унифицированного
интерфейса для доступа к API этих служб. Другой занимается написанием адаптеров общего интерфейса
к API конкретных служб доставок. Третий занимается API для рассчёта вариантов доставок. Четвёртый —
API создания и чтения отправлений. Пятый занимается компонентом обновления статусов отправлений.
Обладая такими ресурсами, разработку системы можно уложить примерно в два месяца. Ещё месяц уйдёт
на развёртывание тестового стенда, тестирование и отладку.

В условиях нехватки ресурсов, реестр служб доставки можно реализовать в виде статического файла,
который будет редактироваться "руками" и распространяться в код остальных компонентов. Также можно
реализовать сначала "PULL" подход для обновления статусов отправлений, и уже после запуска — "PUSH"
для тех служб, которые его поддерживают. Ещё можно отложить реализацию повторных попыток создания
отправлений в API служб доставок.

Кроме разработчиков потребуется девопс/админ с опытом администрирования Apache Kafka и построения
кластера PostgreSQL с шардингом и потоковой репликацией. В принципе, на первом этапе шардинг можно
не реализовывать, однако по мере роста количества отправлений, рано или поздно, момент
необходимости его внедрения настанет.

Этапы:
1. Проектирование типов данных для служб доставки, вариантов доставки, отправлений, истории изменений статусов отправлений.
2. Проектирование универсального интерфейса для доступа к API служб доставки после изучения нескольких служб с разными типами доставки, чтобы уловить всё разнообразие вариантов.
3. Разработка реестра служб доставки.
4. Разработка API рассчёта вариантов доставки и части адаптеров к API служб доставок, отвечающей за варианты доставки. Предварительно согласовать формат API с командой фронтенда.
5. Разработка API создания отправлений и части адаптеров к API служб доставок, отвечающей за создание отправлений. Необходимо согласовать взаимодействие с командой создания заказов. Возможно, нужно пересмотреть архитектуру в сторону асинхронного взаимодействия с ними через очередь.
6. Разработка компонента обновления статуса отправлений.
