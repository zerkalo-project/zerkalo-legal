# Диагностика ZERKALO — 1. База данных

Диагностический отчёт (только чтение, изменения в код не вносились).
Часть 1 из 4: схема БД, реестр таблиц, дубли/мёртвые места, готовые SQL-запросы для Adminer.

## 0. Контекст: какой бэкенд считать «настоящим»

| Контур | Где | Статус |
|---|---|---|
| **VPS Timeweb** (PostgREST :3000, Auth-proxy :4000, Functions :5000) | `deploy/timeweb/*.sql`, `deploy/functions/`, `deploy/postgrest/` | **Активный**: `BackendConfig.usePostgrest = bool.fromEnvironment('USE_POSTGREST', defaultValue: true)` — `lib/api/backend_config.dart:15-47`, URL по умолчанию `http://130.49.150.71:{3000,4000,5000}` (строки 9-11) |
| **Supabase** (миграции + legacy Edge Functions) | `supabase/migrations/*` (21 файл) | Исторический путь. Клиент умеет в него ходить (`BackendConfig.usePostgrest == false`), в коде сохранены обе ветки почти во всех сервисах |
| **Схема прода** | `deploy/timeweb/01_public_schema.sql` (639 строк) + `02_roles_rls.sql`, `03_auth.sql`, `04…07`, `99`, `100`, `365_questions.sql` | Финал всех миграций, переписанный без Supabase Auth |

Важно для аудита: в Supabase-ветке ещё существует история таблиц `spheres`, `sphere_templates`, `sphere_catalog` — они **удалены**: `supabase/migrations/20260913130000_drop_spheres.sql:285-287`. В схеме Timeweb их нет вовсе. То есть любые ссылки на «сферы» в БД — мёртвые; в клиенте `sphereId` остался как локальное поле модели (`lib/models/life_event.dart`, `user_metric.dart`, `day_task.dart`, `note_reminder.dart`).

---

## 1.1. Полный список таблиц — готовый запрос для Adminer

```sql
-- 1A. Все объекты схемы public: таблицы, вьюхи, сиквенсы — с приблизительным числом строк и размером
select c.relkind                                        as kind,   -- r=таблица, v=вьюха, S=сиквенс
       c.relname                                        as name,
       c.reltuples::bigint                              as approx_rows,
       pg_size_pretty(pg_total_relation_size(c.oid))     as total_size,
       coalesce(obj_description(c.oid, 'pg_class'), '')  as comment
from pg_class c
join pg_namespace n on n.oid = c.relnamespace
where n.nspname = 'public'
  and c.relkind in ('r', 'p', 'v', 'm', 'S')
order by c.relkind, c.relname;
```

```sql
-- 1B. Точное число строк по каждой таблице (один запрос, копировать целиком)
select 'users' as t, count(*) from public.users
union all select 'input_sources',      count(*) from public.input_sources
union all select 'events',             count(*) from public.events
union all select 'achievements',       count(*) from public.achievements
union all select 'metrics',            count(*) from public.metrics
union all select 'metric_values',      count(*) from public.metric_values
union all select 'note_folders',       count(*) from public.note_folders
union all select 'notes_reminders',    count(*) from public.notes_reminders
union all select 'calendar_events',    count(*) from public.calendar_events
union all select 'day_tasks',          count(*) from public.day_tasks
union all select 'input_transcripts',  count(*) from public.input_transcripts
union all select 'notifications',      count(*) from public.notifications
union all select 'daily_questions',    count(*) from public.daily_questions
union all select 'daily_polls',        count(*) from public.daily_polls
union all select 'daily_answers',      count(*) from public.daily_answers
union all select 'subscriptions',      count(*) from public.subscriptions
union all select 'payments',           count(*) from public.payments
union all select 'user_settings',      count(*) from public.user_settings
union all select 'fcm_devices',        count(*) from public.fcm_devices
union all select 'error_logs',         count(*) from public.error_logs
union all select 'legal_documents',    count(*) from public.legal_documents
union all select 'auth_users',         count(*) from public.auth_users
order by 2 desc, 1;
```

```sql
-- 1C. Колонки всех таблиц (для сверки «используется ли колонка»)
select table_name, ordinal_position, column_name, data_type, is_nullable, column_default
from information_schema.columns
where table_schema = 'public'
order by table_name, ordinal_position;
```

```sql
-- 1D. «Какие индексы вообще дергали» — сразу видно непопуляризуемые
select relname        as table_name,
       indexrelname   as index_name,
       idx_scan       as scans,
       idx_tup_read   as tuples_read,
       pg_size_pretty(pg_relation_size(indexrelid)) as size
from pg_stat_user_indexes
where schemaname = 'public'
order by idx_scan asc, relname, index_name;
```

```sql
-- 1E. Таблицы: seq scan vs index scan, мёртвые кортежи, автовакуум
select relname, seq_scan, idx_scan, n_live_tup, n_dead_tup,
       last_autovacuum, last_analyze
from pg_stat_user_tables
order by seq_scan desc, relname;
```

---

## 1.2. Реестр таблиц: назначение и кто их использует

Обозначения: **К** = клиент Flutter (`lib/`), **С** = сервер (`deploy/functions/`, `deploy/postgrest/`).

### users — `01_public_schema.sql:59`
Профиль (email, name, birth_date, onboarding_completed). Комментариев в схеме нет, назначение восстанавливается по коду.
- К: `lib/services/auth_service.dart:247, 275, 307, 317` (upsert профиля), `lib/services/mirror_service.dart:260` и `lib/services/note_service.dart:143, 164` — `ensureProfile()` (upsert пустой строки, чтобы PostgREST/RLS видел пользователя).
- С: `delete_account.js:74` (удаление последним).
- Статус: **используется**. Это «якорь» для FK/RLS (в Timeweb-схеме нет FK на `auth.users`).

### input_sources — `:69` (индекс `input_sources_user_idx:78`)
Хранилище исходников ввода (kind text/audio, original_text, audio_path).
- К: **0 упоминаний** ни в `lib/`, ни где-либо ещё.
- С: только `delete_account.js:43` — `select audio_path … .not('audio_path','is',null)`, чтобы удалить файлы из Storage.
- INSERT-а нет вообще: `process_input.js:159` кладёт аудио в Supabase Storage через `uploadInputAudio` (`lib/shared/storage.js:49`), строку в `input_sources` не создаёт.
- Статус: **фактически мёртвая** (всегда пустая), но занимает место в RLS (`02_roles_rls.sql:114`), GRANT-ах и в списке удаления аккаунта.

### events — `:81` (индексы `:110-118`, 5 шт.)
Два смысла в одной таблице: (а) «жизненные события» со скорингом (`event_type`, `strength`, `score`, constraint `events_score_range`, триггер `derive_strength:504`) и (б) **лента истории** — нейтральные строки «что сказано и куда ушло» (`destination` in `'sphere','notes','tracker','calendar','metrics'`, `:106-108`).
- К: `lib/services/event_service.dart:29-124` (fetch/insert/update/delete), `lib/state/app_state.dart:343` (save), `:450` (load), `:849-884` (update/delete), `:1774-1790` (`_logActivity` → `eventService.create(... destination: …)`, затем `addEvent` + `_pushEventRemote`), `:3450-3457` (delta pull + push), `:3793-3799` (replay офлайн-очереди).
- С: `lib/shared/subscription.ts:55` — `countEventsThisMonth` (лимит бесплатного плана по `events.created_at`), `delete_account.js:11`.
- Нюанс: **каждый** голосовой/текстовый ввод пишет строку в `events` (см. `_logActivity`) — то есть на одну фразу появляются: запись в `notes_reminders`/`day_tasks`/`metric_values`/`calendar_events` **плюс** строка в `events`.
- `source_id` — в схеме есть (`:84`), но клиент его не пишет: `event_service.dart:66-81` (`_payload`) не содержит `source_id`. Читается в `LifeEvent.fromSupabase`. Статус: **используется, но `source_id` — мёртвая колонка на запись**.
- `destination`, `is_achievement`, `achievement_weight` — используются клиентом в payload (`:76-79`).

### achievements — `:121` (индекс `:133`, триггер `sync_event_achievement:476`)
«Достижения/гордость» (title, weight in 10/25/50/75/100, occurred_at).
- К: **0 упоминаний** (`achievements` встречается только в `lib/models/life_event.dart` как *поле* `isAchievement`, не как таблица).
- С: только в списке удаления `delete_account.js:11`.
- Статус: **не используется**, при этом триггер продолжает синхронизировать `events.is_achievement/achievement_weight` — то есть таблица дублирует две колонки `events`.

### metrics — `:136` (индексы `:150, 153`)
Каталог «Мои метрики»: name, unit, ai_hint, color, sort_order, is_archived, goal.
- К: `lib/services/metric_service.dart:92` (локальный ключ стора), `:112-118` (pull через PostgREST/Supabase), `:189-211, 359-388` (insert/update), `lib/state/app_state.dart:3371-3385` (delta pull), `:2860-2926` (очередь операций).
- С: `process_input.js:171` (каталог для промпта, `is_archived=false`, `order created_at desc`), `lib/shared/metrics.ts:85, 98` (выбор карточки), `:115, 646` (вставка замеров).
- Статус: **ключевая таблица**, используется с обеих сторон.

### metric_values — `:156` (индексы `:167, 169`)
Замеры: metric_id, user_id, event_id, value, date.
- К: `metric_service.dart:643-661` (load), `:249-296` (`insertValueRemote`, upsert `onConflict: 'id'`), `app_state.dart:3009-3076` (`_saveStandaloneMetrics`), `:3210-3244` (replay очереди), `:3371-3385` (delta).
- С: `metrics.ts:598-676` (`saveStandaloneMetrics` — вставка для голосового/текстового пути, **только при `preview=false`**), `:646`, `:70-160` (`saveEventMetric`).
- **Дублей по ключу нет**: UNIQUE(`metric_id`,`date`) отсутствует → отсюда разовые скрипты `deploy/timeweb/99_metric_values_dedup.sql` и `100_metric_values_dedup_safe.sql`. Клиент схлопывает только по `id`/`event_id+metric_id` (`metric_service.dart:456-508`).
- `event_id` почти всегда пустой: в голосовом пути `app_state.dart:3028-3035` создаёт `MetricSample(..., eventId: eventId)` где параметр `eventId = ''` по умолчанию (`:3012`) и вызов `:1583-1588` его не передаёт.

### note_folders — `:172` (индексы `:182, 184, 186`)
Папки заметок + ручной порядок `position`.
- К: `note_service.dart:85` (table), `:358-411` (insert/update/delete), `app_state.dart:3364, 3518-3538`.
- С: нет (кроме удаления аккаунта).
- Статус: используется. `position` появился позже всех (миграция `20261009120000_note_folder_position.sql`), в Timeweb-схеме уже есть (`:176`), индекс `note_folders_position_idx:186`.

### notes_reminders — `:189` (индексы `:201-207`, + trgm)
Заметки и напоминания: text, reminder_datetime, repeat_type (enum `none/once/daily/weekly/monthly`, `:28-29`), is_active, folder_id.
- К: `note_service.dart:44` (table), `:227, 246, 319-348` (load/insert/update/delete), `app_state.dart:3485-3538`.
- С: `process_input.js:323-327` — INSERT заметки, но достигается **только при `preview=false`** (ветка `if (preview)` на `:210` возвращает ответ раньше). `delete_account.js:12`.
- Тонкость: `lib/models/note_reminder.dart:68-77` читает `sphere_id` из строки — колонки в схеме Timeweb **нет** (удалена вместе со сферами) → всегда `null`, безвредный no-op.
- Статус: используется. Серверный путь записи — мёртвый код (см. 3.4).

### subscriptions — `:210`
Подписка пользователя: plan_type enum, status enum, started_at, expires_at, is_trial, yookassa_payment_method_id, intro_used.
- К: `lib/services/subscriptions_remote_service.dart` (2 обращения), UI `subscription_screen.dart`, `app_state.dart` (15 упоминаний).
- С: `subscription.ts:67, 79` (`loadEntitlements`, авто-перевод в `expired`), `create_payment.js:63`, `renew_subscriptions.js:32, 85, 95`, `yookassa_webhook.js:119, 130, 155`.
- Дубли внутри: `plan_type='trial'` + `status='trialing'` + `is_trial` + `intro_used` описывают одно и то же состояние разными полями.

### payments — `:224`
Платежи ЮKassa: yookassa_payment_id (unique), amount_rub, status enum, is_intro, payload jsonb, confirmed_at.
- К: `payments` в `lib/services/payment_service.dart` — только через функции (`VpsFunctionsClient`), таблицу напрямую не читает/не пишет.
- С: `create_payment.js:123`, `renew_subscriptions.js:70`, `yookassa_webhook.js:43, 80`.
- Статус: **серверная таблица**, клиенту недоступна (RLS-политики есть, но запросов нет). `is_intro` дублирует `subscriptions.intro_used`.

### user_settings — `:240`
Тема, дни/часы напоминаний, language, timezone, morning/evening.
- К: `app_state.dart:3704, 3716` (`_pullSettings` через PostgREST / Supabase), запись темы/напоминаний через `SettingsService`.
- С: `send_reminders.js:179` (raw SQL, cron), `delete_account.js`.
- Статус: используется.

### fcm_devices — `:260`
Токены push (provider check `rustore`/`fcm`, unique(user_id, token), last_seen_at).
- К: напрямую нет — только через функцию `register_device` (`lib/services/push_service.dart`).
- С: `register_device.js:27` (upsert), `send_reminders.js:371-386` (выбор устройств), `delete_account.js`.
- Статус: используется.

### error_logs — `:272` (индекс `:281`)
Журнал ошибок сервера (source, message, details jsonb).
- К: **0 упоминаний** (клиент ошибки никуда не шлёт).
- С: `lib/shared/log_error.js` → INSERT в `public.error_logs`; вызывается из `send_reminders.js:359, 406`.
- Статус: используется только сервером; на клиенте логов нет.

### legal_documents — `:284`
Политика/согласия (slug, title, body).
- К: **0 упоминаний**. Тексты на клиенте/лендинге — статические файлы: `docs/politika.html`, `docs/soglashenie.html`, `docs/soglasie.html`, `deploy/legal/*`.
- С: ни одного запроса.
- Статус: **мёртвая** (кроме сида `:591` и RLS `02:122`).

### calendar_events — `:291` (индексы `:309-313`)
Календарь: title, starts_at, три напоминания (reminder_at / reminder_soon_at / reminder_at_event), is_annual, is_monthly, is_all_day, color, repeat_type text `once|yearly|monthly|weekly` (`:302-303`).
- К: `lib/services/calendar_service.dart:57, 100-147` (fetch/insert/update/delete), `app_state.dart:3547-3573`.
- С: `send_reminders.js:255, 306` (raw SQL по `reminder_at/reminder_soon_at/reminder_at_event` и repeat_type), `delete_account.js`.
- Дубли: три колонки `reminder_*` под одно «напомнить»; два словаря повторяемости (text здесь vs enum в `notes_reminders`) плюс клиентский `RepeatRule {none, once, daily, weekly, monthly, yearly}` (`lib/models/enums.dart:10`) — **в `calendar_events.repeat_type` нет `none`/`daily`**, а в `notes_reminders.repeat_type` нет `yearly` (`:28-29`).

### day_tasks — `:316` (индексы `:326, 328`)
Задачи дня: title, day, done.
- К: `lib/services/task_service.dart` (create/rollOver/prune), `app_state.dart:3582-3604`.
- С: `send_reminders.js:249` (напоминание о задачах на день), `delete_account.js`.
- Статус: используется.

### input_transcripts — `:331` (индекс `:339`)
Сырые фразы пользователя (text, spoken_at).
- К: `app_state.dart:987-1064` (`_rememberTranscript` + `_uploadTranscript`), `:1066-1104` (`_syncTranscriptsRemote`, лимит `_transcriptLimit`).
- С: только удаление аккаунта.
- Статус: используется.

### notifications — `:342` (индексы `:356, 358`, unique(user_id, source_key))
Встроенные уведомления (дублируют push): kind, title, body, read_at, deleted_at.
- К: `lib/services/inbox_service.dart:54, 70, 86` (fetch/upsert `onConflict: user_id,source_key`), `app_state.dart:3430`.
- С: `send_reminders.js:391-404` (INSERT … ON CONFLICT DO NOTHING, `sourceKey` формируется в `:334-337`, формат `cal:day:<id>:<date>` / `cal:<kind>:<id>:<stamp>`).
- Замечание: в `delete_account.js` таблица **не** входит в `USER_TABLES` (`:9-24`), но строки удалятся каскадом при `delete from public.users` (FK `on delete cascade`), т.к. `users` удаляется на `:74`.

### daily_questions — `:362`
Вопросы дня (365 шт.: `deploy/timeweb/365_questions.sql`), CHECK по `type`/`category`.
- К: `mirror_service.dart:144` (fetch, `order sort_order`).
- С: нет (кроме функции `daily_question_stats`, читающей её).
- Статус: используется (read-only).

### daily_polls — `:373` (индекс `:389`, unique(user_id, date, slot))
Опросы «Зеркала»: slot `morning|noon|evening|night`, sleep, mood, energy, stress, sport (числа!).
- К: `mirror_service.dart:122, 158-176` (upsert `onConflict: user_id,date,slot`), `app_state.dart:3431, 3614-3641`.
- С: нет.
- Дубли: `mood/energy/stress/sleep/sport` — те же показатели, что живут в `metrics`/`metric_values`.

### daily_answers — `:392`
Ответ на вопрос дня: **unique(user_id, date)** — один ответ в день на 365 вопросов.
- К: `mirror_service.dart:133, 191-208` (upsert `onConflict: user_id,date`), `app_state.dart:3437, 3643-3665`.
- Серверная функция `public.daily_question_stats` (`05_daily_question_stats.sql:4`) читает эту таблицу, считает среднее/распределение по другим пользователям; вызывается из `daily_question_stats.js:28`, а клиент дергает `mirror_service.dart:241` → `VpsFunctionsClient.invoke('daily_question_stats')`.

### auth_users — `03_auth.sql:19`
Собственная таблица логина (email, password hash) для Auth-proxy на :4000.
- С: `deploy/postgrest/auth/src/index.js:181` (`select id,email from public.auth_users where id=$1`), `delete_account.js:86`.
- Статус: используется (только VPS-контур).

### metric_values_backup_dedup — `100_metric_values_dedup_safe.sql:22`
Разовый бэкап перед чисткой дублей (`create table if not exists … as select * from public.metric_values`).
- Никем не читается. Статус: **технический мусор**, если чистка уже проведена (удалять вручную).

### Вьюхи и функции
| Объект | Где | Используется? |
|---|---|---|
| `v_metric_totals` (view) | `01_public_schema.sql:551` | **Нет нигде.** Клиент считает итоги в Dart (`metric_service.dart` — `dayTotal/weekTotal`). Кандидат «мёртвого» объекта. |
| `set_updated_at()` | `:410` | Да (все триггеры `*_updated_at`) |
| `sync_event_achievement()` | `:476` | Формально да, но только для мёртвой таблицы `achievements` |
| `derive_strength()` | `:504` | Да (`events_derive_strength`) |
| `age_years`, `age_bucket` | `:528, 539` | Только в SQL, из кода не вызываются |
| `requesting_user_id()` | `02_roles_rls.sql:18` | Да — все RLS-политики в Supabase-ветке |
| `verify_password`, `register_user`, `login_user` | `03_auth.sql:40, 70, 116` | Да — Auth-proxy |
| `daily_question_stats(uuid, uuid)` | `05_daily_question_stats.sql:4` | Да — маршрут `/functions/v1/daily_question_stats` |

## 1.3. Признаки дублей и «мёртвых» мест

**(а) Похожие по назначению сущности**

| # | Что с чем | Суть дублирования |
|---|---|---|
| 1 | `events` ↔ `calendar_events` + `day_tasks` + `notes_reminders` + `metric_values` | `events.destination` (`:106-108`) хранит адрес записи, который и так известен по таблице-получателю. Плюс сама по себе `events` — это лента истории (см. `destination='notes'/'tracker'` в `_logActivity`). |
| 2 | `achievements` ↔ `events.is_achievement` + `events.achievement_weight` | Два места под одно «достижение»; `events`-колонки заполняются триггером из `achievements`, которую никто не пишет. |
| 3 | `events.source_id` ↔ `input_sources` | Два способа сослаться на исходный ввод; ни один не заполняется (см. выше). |
| 4 | `metric_values.event_id` ↔ `events` | Второй путь «замер ↔ событие»; в голосовом пути всегда `''`/null (`app_state.dart:3012, 3028`). |
| 5 | `daily_polls.sleep/mood/energy/stress/sport` ↔ `metrics`/`metric_values` | Одни и те же показатели в двух системах; опросы в `daily_polls` не попадают в метрики. |
| 6 | `subscriptions.{plan_type='trial', status='trialing', is_trial, intro_used}` ↔ `payments.is_intro` | Четыре поля вместо одного признака «вводный период». |
| 7 | `notifications.{user_id, source_key}` ↔ `fcm_devices.{user_id, token}` | Два разных механизма дедупликации одного напоминания (ключ — `send_reminders.js:334-337` vs уникальность токена). |
| 8 | `notes_reminders.repeat_type` (enum, `:28-29`) ↔ `calendar_events.repeat_type` (text CHECK, `:302-303`) ↔ клиентский `RepeatRule` | Три словаря: enum без `yearly`, text без `none/daily`, клиент с `yearly`. Плюс клиент шлёт `repeat.name` в обоих случаях (`note_service.dart:305`, `calendar_service.dart:72`) — рассинхрон возможен. |
| 9 | `calendar_events.reminder_at` / `reminder_soon_at` / `reminder_at_event` ↔ `notes_reminders.reminder_datetime` | Одна сущность «напомнить» в четырёх колонках. |
| 10 | `metric_values` без UNIQUE(metric_id,date) | Источник копий — отсюда `99_metric_values_dedup.sql` и `100_metric_values_dedup_safe.sql` (в последнем прямо сказано: клиент вставлял замеры без id и дельта-синк лил историю заново). |

**(б) Неиспользуемые колонки**
- `events.source_id` — не пишется (`event_service.dart:66-81`).
- `notes_reminders.sphere_id` — читается клиентом (`note_reminder.dart:68-77`), но колонки в схеме Timeweb нет (удалена вместе со сферами) → всегда null.
- `metrics.sphere_id` — в схеме нет, но остался в типах/сигнатурах: `deploy/functions/lib/shared/ai.ts:16` (`MetricHint.sphere_id?`), `metrics.ts` (5 упоминаний), `lib/models/user_metric.dart`. Мёртвый параметр.
- `payments.payload` jsonb — пишется вебхуком (`yookassa_webhook.js:80`), не читается никем.
- `daily_questions.created_at`, `legal_documents.updated_at` — служебные, ок.
- `error_logs.details` — пишется `log_error.js`, читается только вручную.

**(в) Индексы, похожие на «без запросов»** (проверить `idx_scan` запросом 1D):
- `events_text_trgm_idx` (`:116`), `events_parsed_trgm_idx` (`:118`), `notes_text_trgm_idx` (`:207`) — создавались под серверный поиск (`20260903130000_search_and_intro.sql`). Единственный `ilike` в серверном коде — `metrics.ts` (3 шт.); в клиенте поиск идёт по локальным спискам (`note_reminder.dart:23-28`, `life_event.dart:93-98`). Похоже, trgm нигде не нужен.
- `age_years`/`age_bucket` — в SQL есть, вызовов нет.
- Что **точно нужно** (используется дельта-синком через `updated_at`: `lib/services/offline_sync.dart:14-37`): `events_updated_idx`, `notes_updated_idx`, `calendar_updated_idx`, `tasks_updated_idx` (имя индекса для `day_tasks` — `:328`!), `metrics_updated_idx`, `metric_values_updated_idx`.
- Дубли индексов: `day_tasks_user_day_idx` + `tasks_updated_idx` (пересечение по user_id), `metrics_user_active_idx` (partial) + `metrics_updated_idx`.

## 1.4. SQL-запросы для проверки (Adminer, «SQL-запрос», по шагам)

```sql
-- Q1. Дубли замеров метрик: точные копии «метрика+значение+дата»
select metric_id, value, date,
       count(*) as copies,
       min(created_at) as first_at,
       max(created_at) as last_at
from public.metric_values
group by metric_id, value, date
having count(*) > 1
order by copies desc, last_at desc
limit 100;
```

```sql
-- Q2. Дубли по «надёжному» ключу: одно событие даёт один замер на метрику
select event_id, metric_id, count(*) as copies
from public.metric_values
where event_id is not null
group by event_id, metric_id
having count(*) > 1
order by copies desc;
```

```sql
-- Q3. Дубли в пользовательских сущностях (заметки/календарь/задачи/карточки метрик)
select 'notes' as kind, user_id::text as owner, text as title,
       coalesce(reminder_datetime::text,'') as when_at, count(*) as copies
from public.notes_reminders
group by 1,2,3,4 having count(*) > 1
union all
select 'calendar', user_id::text, lower(title), starts_at::text, count(*)
from public.calendar_events
group by 1,2,3,4 having count(*) > 1
union all
select 'task', user_id::text, lower(title), day::text, count(*)
from public.day_tasks
group by 1,2,3,4 having count(*) > 1
union all
select 'metric_card', user_id::text, lower(name) || ' (' || unit || ')', '', count(*)
from public.metrics
group by 1,2,3,4 having count(*) > 1
order by copies desc, kind;
```

```sql
-- Q4. «Мёртвые» записи и внутренние противоречия
select 'input_sources_total'   as check_name, count(*)::text as value from public.input_sources
union all select 'achievements_total',            count(*)::text from public.achievements
union all select 'legal_documents_total',         count(*)::text from public.legal_documents
union all select 'notes_repeat_none_with_date',   count(*)::text from public.notes_reminders
        where repeat_type = 'none' and reminder_datetime is not null
union all select 'notes_repeat_set_without_date', count(*)::text from public.notes_reminders
        where repeat_type <> 'none' and reminder_datetime is null
union all select 'calendar_repeat_conflict',      count(*)::text from public.calendar_events
        where (repeat_type = 'once' and (is_annual or is_monthly))
           or (repeat_type <> 'once' and not (is_annual or is_monthly))
union all select 'metric_values_on_archived',     count(*)::text from public.metric_values v
        join public.metrics m on m.id = v.metric_id where m.is_archived
union all select 'users_without_subscription',    count(*)::text from public.users u
        left join public.subscriptions s on s.user_id = u.id where s.id is null
union all select 'orphan_metric_values',          count(*)::text from public.metric_values v
        left join public.metrics m on m.id = v.metric_id where m.id is null
union all select 'events_by_destination',         string_agg(destination || '=' || cnt, ', ' order by destination)
        from (select destination, count(*) cnt from public.events group by 1) x;
```

```sql
-- Q5. Что реально пишется на сервере, а что только на клиенте (быстрая проверка гипотезы «серверный путь мёртв»)
select 'notes_from_server_preview' as check_name,
       count(*)::text as value
from public.notes_reminders n
left join public.notes_reminders n2 on false
where false;  -- заглушка: серверный INSERT отличается от клиентского только наличием folder_id = null и repeat_type из модели ИИ
-- Практический вариант: заметки, созданные сервером (нет folder_id и reminder_datetime часто = момент вставки)
select count(*) as server_style_notes
from public.notes_reminders
where folder_id is null and repeat_type = 'none' and reminder_datetime is null;
```

Примечание к Q5: надёжно отличить серверную вставку нельзя — поэтому основной вывод о мёртвом серверном пути делается по коду (`preview=true` во всех клиентских вызовах), а не по данным.
