# Модель данных

![UML данных](diagrams/04-data-model.png)

Логическая модель для PostgreSQL. PK обозначает первичный ключ, FK связь с другой сущностью, UQ уникальность. Схема показывает основные связи; правила целостности ниже обязательны при создании физической схемы

## Сущности

| Сущность | Ключи и основные данные |
|---|---|
| Trip | `id` PK, `title`, `owner_id`, `current_version`, `currency`, `source_context_revision`, `context_hash`, `context_verified_at`, `tracking_enabled`, `last_inbox_sequence` |
| TripMember | PK `(trip_id, user_id)`, `active`, `can_confirm`, `baggage_kg`, `route_entitlements`, `interests` |
| Flight | `id` PK, полный ключ UQ, исходное расписание, IANA-зоны, ожидаемые UTC-времена, `status`, назначенный источник, поток, ревизия, `state_version`, `current_event_id` nullable FK, `last_observed_at`, `last_verified_at` |
| FlightStatusEvent | `id` PK, `provider_id`, `event_id`, ключ рейса, `flight_id` nullable FK, поток/ревизия, `observed_at`, серверное `received_at`, разрешённый payload, хеш, outcome |
| PlanItem | `id` PK, `trip_id` FK, `kind`, `flight_id` nullable FK, `predecessor_id` nullable, UTC-окна, IANA-зона, MCT/запасы/переезд, ссылка на бронь, `booking_status` |
| Impact | `id` PK, `trip_id`, `item_id`, `source_event_id`, `source_state_version`, `plan_version`, `reason`, `state`, объяснение |
| Offer | `id` PK, `trip_id`, `impact_id`, `kind`, `replacement_item_id`, условия и изменение плана, основание допустимости, цена/валюта, `checked_at`, `expires_at`, fingerprint, версии плана и исходных рейсов |
| InboxMessage | `id` PK, `trip_id`, `user_id`, `kind`, текст, `semantic_key`, `sequence_no`, версия плана, `created_at` |
| OutboxTask | `id` PK, `kind`, `dedup_key`, ссылка на объект, типизированный payload/курсор, `next_attempt_at`, `attempts`, `lease_token`, `lease_until`, версии основания, state |
| PlanVersion | PK `(trip_id, version)`, неизменяемый snapshot, автор, причина, `created_at` |
| SelectionCommand | PK `(trip_id, actor_id, idempotency_key)`, хеш запроса, `offer_id`, сохранённый ответ и созданная версия |

`context_state` в API вычисляется из последней проверки и состояния синхронизации; время проверки хранится отдельно от времени изменения плана

Пользователи и аутентификация находятся в TravelTech, отдельная таблица аккаунтов Alfred не нужна. Все участники относятся ко всем элементам общего плана в рамках выбранного ограничения кейса

## Идентификация рейса

`Flight.id` является внутренним UUID. Он используется в FK, но поставщик не обязан знать этот ID

Сообщение сопоставляется по UQ из шести полей:

```text
operating_carrier + flight_number + departure_date
+ origin + destination + scheduled_departure_local
```

Используются код выполняющего перевозчика, нормализованный номер, местная дата и исходное плановое время вылета, IATA-коды аэропортов. `scheduled_departure_local` имеет формат HH:mm, 24:00 нормализуется в 00:00 следующей даты до сопоставления. Изменение ожидаемого времени не меняет ключ

Сообщения codeshare предварительно приводятся адаптером к выполняющему перевозчику по подтверждённому справочнику. Если однозначного соответствия нет, событие остаётся `unmatched`

## Целостность

- UQ `(provider_id, event_id)` для события; адаптер гарантирует уникальность event ID в пределах поставщика, включая смену потока
- UQ `(trip_id, id)` для PlanItem и составной FK `(trip_id, predecessor_id)` на PlanItem запрещают зависимость от чужой поездки
- У элемента один непосредственный предшественник. Это ограничение общего линейного маршрута; циклы запрещены проверкой приложения при записи плана
- `kind=flight` требует `flight_id`, остальные виды не содержат его. MCT, багаж, переезд и запасы неотрицательны
- Составные FK связывают Impact с элементом и версией той же поездки, Offer с последствием и заменяемым элементом той же поездки
- Trip.owner_id ссылается на участника этой поездки. Создание Trip, участников и версии 1 выполняется одной транзакцией с отложенной проверкой циклических FK
- InboxMessage и SelectionCommand ссылаются на `(trip_id, user_id)` участника. Удаление участника означает active=false, чтобы не разрушать ссылки истории; новые чтения, подтверждения и доставки запрещены. Эти FK не заменяют проверку доступа в API
- `current_version` ссылается на существующую PlanVersion этой поездки; snapshot содержит весь план, участников и ограничения
- UQ последствия: `(trip_id, item_id, source_event_id, source_state_version, plan_version, reason)`. Повтор состояния с другим event ID отсеивается до создания новой версии состояния
- UQ уведомления: `(trip_id, user_id, semantic_key)`. Ключи: `status:flight:state_version`, `disruption:flight:state_version:plan_version`, `plan:version`; сообщение о нарушении объединяет все последствия одного расчёта
- UQ `(trip_id, sequence_no)` и счётчик Trip.last_inbox_sequence задают порядок inbox. Номер назначается после блокировки Trip и сохраняется вместе с сообщением; курсор не строится по времени создания
- UQ OutboxTask.dedup_key исключает повторное создание одной задачи; выполненные задачи сохраняются до окончания окна повторной доставки. Для poll/sync ключ включает цикл запуска, для обработки состояния flight/state_version/trip, для доставки ID сообщения/канал
- Flight.current_event_id указывает на событие этого же рейса, создавшее текущий state_version; до первого подтверждения current_event_id=NULL и state_version=0
- Подключение поездки и новая версия плана создают пересчёт по уже известным состояниям рейсов, независимо от поступления нового события
- Задача process_impact содержит source_event_id и source_state_version. Устаревшее основание не применяется; задача актуального состояния создана атомарно с Flight
- Контекст TravelTech обновляется только при подтверждённой новой source_context_revision; одинаковая ревизия с другим context_hash требует сверки. План и фактическая бронь имеют разную ответственность
- Завершённый SelectionCommand хранит успешный ответ; повтор с тем же ключом и другим хешем отклоняется. Конкурентный повтор разрешается уникальным ключом и чтением сохранённого результата

## Время, разрешения и состояния

Моменты хранятся как `timestamptz`, локальные дата/время расписания и IANA-зоны отдельно. В API моменты передаются в UTC с Z. При неоднозначном местном времени требуется offset или уточнение у источника, угадывать запрещено

`route_entitlements` содержит разрешение/запрет/неизвестность для заданного маршрута, основание проверки и `valid_until`; паспортных реквизитов нет. Условия каждой пересадки и изменения аэропорта учитываются отдельно

| Объект | Состояния |
|---|---|
| Flight | scheduled, delayed, cancelled, departed, arrived |
| Event | received, unmatched, pending_reconciliation, applied, ignored_stale, conflict |
| Impact | open, needs_information, resolved, superseded |
| Поиск | searching, available, none_eligible, temporarily_unavailable |
| Бронирование элемента | booked, needs_booking, needs_change, not_required |
| OutboxTask | ready, processing, done, failed |

Индексы: Flight по времени и статусу; Event по flight/received_at; PlanItem по flight_id; InboxMessage по trip/user/sequence_no; OutboxTask по state/next_attempt_at. Запрет UPDATE/DELETE snapshot в PlanVersion обеспечивается правами приложения и триггером; откат плана создаёт новую версию

Физические DDL и миграции не входят в это решение кейса
