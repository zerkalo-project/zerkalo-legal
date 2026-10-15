# Диагностика ZERKALO — 2. Парсеры (клиент)

Диагностический отчёт (только чтение, изменения в код не вносились).
Часть 2 из 4: инвентарь парсеров `lib/utils/`, точки вызова, пересечения с ИИ.

## 2.1. Инвентарь: файл → назначение → публичный API

| Файл | Строк | Назначение | Публичные функции/классы |
|---|---|---|---|
| `metric_parse.dart` | 1319 | Числа → «Мои метрики»: каталог, стемы, денежные семьи (траты/доход/инвест/кредит), раскрытие сленга («косарь», «пятёра»), единицы, план/факт | `ParsedMetric` (:4), `metricsFromText` (:20), `matchUserMetrics` (:39), `mergeParsedMetrics` (:189), `confineParsedMetricsToCatalog` (:211), `looksLikeFinancePhrase` (:266), `convertToMetricUnit` (:669), `canonicalMetricUnit` (:754), `unitAliasesFor` (:757), `metricNameKey` (:788), `sameMetricName` (:791), `metricNameScore` (:795), `pickUserMetric` (:805), `expandSpokenAmounts` (:831), `metricClauses` (:1225), `looksLikeSportPhrase` (:1243) |
| `note_intent.dart` | 391 | Лид заметки («заметка:», «запиши в заметки», «напомни»), тело до конца фразы, нарезка тела на заметку/задачи/события | `VoiceNoteDraft` (:7), `splitNoteSentences` (:48), `isNoteLeading` (:77), `isPureNotePhrase` (:85), `hasNoteReminderLead` (:97), `stripNoteLeadSentences` (:102), `noteLeadBody` (:113), `NoteBodySplit` (:128), `splitNoteBody` (:223), `parseVoiceNote` (:295), `noteDraftFromHeard` (:342) |
| `task_intent.dart` | 470 | Дела на сегодня: лид «сегодня нужно / надо / не забыть / трекер», нарезка перечислений, раздача глагола, «сделал/выполнил», датированные задачи | `stripSpeechArtifacts` (:116), `looksLikeTaskList` (:123), `hasExplicitTaskLead` (:133), `looksLikeTrackerToday` (:139), `DatedTask` (:150), `looksLikeTaskDone` (:157), `expandTaskTitles` (:160), `looksLikePastFact` (:194), `parseActionTitles` (:212), `parseTaskList` (:232), `parseDatedTasks` (:242), `parseTaskDone` (:310) |
| `calendar_intent.dart` | 1336 | Календарь: лид, даты, перенос/отмена, дни рождения, «день N числа», заголовок-суть, привязка к существующим событиям | `CalendarDraft` (:9), `hasCalendarLead` (:202), `looksLikeCalendarCancel` (:204), `looksLikeCalendarReschedule` (:214), `looksLikeBirthdayList` (:221), `looksLikeCalendar` (:227), `parseRelativeClock` (:302), `parseCalendarIntent` (:334), `parseAllCalendarIntents`, `sameCalendarTitle` (:947), `phraseMentionsDay` (:1009), `dateWithLocalClock` (:1037), `birthdayPersonFrom` (:1260), `parseBirthdayDrafts` (:1223), `parseMonthDayDrafts` |
| `event_dates.dart` | 1298 | Разбор дат/времени (в т.ч. «на прошлой неделе», «12 числа», месяцы словами), общие предикаты «другой день / сегодня / вперёд», источник сегодняшних дел | `parseCalendarStartsAt` (:3), `EventDateParse` (:21), `CalendarMoveRange` (:29), `parseCalendarDate` (:44), `looksLikePastCalendarPhrase` (:125), `allNamedMonthDates` (:242), `allDayOfMonthDates` (:794), `splitCalendarClauses` (:912), `sameCalendarDay` (:928), `parseCalendarAnchorDate` (:938), `parseCalendarMoveRange` (:994), `hasTodayWord` (:1124), `isExplicitOtherDay` (:1129), `hasForwardDate` (:1137), `otherDayClauses` (:1144), `looksLikeSameTime` (:1162), `hasExplicitClock` (:1174), `stripTaskDateNoise` (:1196), `todayTaskSource` (:1231) |
| `calendar_recurrence.dart` | 227 | Повторы календаря и три времени напоминания (за день / за 3 часа / в момент) | `CalendarRecurrenceHint` (:5), `CalendarReminderTimes` (:38), `isAllDayClock` (:50), `looksLikeRecurringCalendar` (:95), `inferCalendarRecurrence` (:116), `recurrenceForCalendar` (:178), `stripRecurrenceWords` (:209), `calendarRepeatLabel` (:216) |
| `parse_confidence.dart` | 17 | Порог уверенности для окна «Я услышал…» | `ParseConfidence.threshold=0.7`, `high=0.92`, `medium=0.8`, `low=0.62`, `clamp`, `anyLow` (:4-16) |
| `ru_regex.dart` | 13 | Замена `\b` для кириллицы (в Dart `\w` = `[A-Za-z0-9_]`) | `ruWordEnd` (:11), `ruWordStart` (:13) |
| `ai_env.dart` | 52 | Секреты клиента; **AI-ключей на клиенте нет** (только ЮKassa) | `cleanEnvSecret`, `readEnvSecret`, `yookassaShopId`, `yookassaClientKey` |
| `date_formatters.dart`, `metric_formatters.dart`, `constants.dart`, `db_color.dart`, `id_generator.dart` | — | Форматирование/константы, не парсеры | — |

## 2.2. Где вызываются (файл:строка)

**`note_intent.dart`**
- `note_intent.dart:300-309` — `parseVoiceNote` внутри самого модуля (`isNoteLeading`, `noteLeadBody`, `splitNoteBody`).
- `note_intent.dart:85-94` — `isPureNotePhrase` сам вызывает `splitNoteBody`.
- `lib/state/app_state.dart:1248` — `parseVoiceNote(transcript, metrics: metrics)` (локальный разбор волны 1).
- `lib/state/app_state.dart:1253-1257` — `splitNoteBody(noteLeadBody(transcript), metrics:, reminderLead: hasNoteReminderLead(transcript))`.
- `lib/state/app_state.dart:1259` — `stripNoteLeadSentences(transcript)` (остаток для метрик).
- `lib/state/app_state.dart:1391-1396` — `noteDraftFromHeard(...)` в `_heardFromRemote` (ответ сервера).
- `lib/state/app_state.dart:1397` — `stripNoteLeadSentences(text)`.
- `lib/services/voice_service.dart:344-347` — `noteDraftFromHeard` в `_resultFromPayload`.
- `lib/services/voice_service.dart:476` — `parseVoiceNote` в `_localFallback` (офлайн/без сессии).
- `lib/utils/calendar_intent.dart:205, 215, 228` — `isPureNotePhrase` как «стоп» для календарных предикатов; `:387` — `isNoteLeading(clause)` при разборе клауз.
- `lib/utils/metric_parse.dart:135` — `looksLikePastFact` для фильтра плановых клауз.

**`task_intent.dart`**
- `app_state.dart:1318-1319` (`hasExplicitTaskLead`, `looksLikeTaskList`), `:1321` (`looksLikePastFact`), `:1322` (`parseDatedTasks`), `:1327` (`expandTaskTitles(parseTaskList(taskWork))`), `:1330-1331` (`looksLikeTaskDone`, `parseTaskDone`), `:1357`, `:1407-1413`, `:1468-1480`, `:1972` (`_addParsedTasks`).
- `voice_service.dart:394` (`expandTaskTitles`), `:509` (`parseTaskList`) — локальный фолбэк.
- `calendar_intent.dart:234, 245, 392` — `looksLikeTrackerToday`, `looksLikeTaskList`; `:415` — `parseActionTitles`.
- `task_intent.dart:165, 168` — самовызовы (`expandTaskTitles` → `parseTaskList`); `:233` — `todayTaskSource` из `event_dates.dart`.

**`calendar_intent.dart` / `event_dates.dart` / `calendar_recurrence.dart`**
- `app_state.dart:1211-1214` (`looksLikeCalendarCancel/Reschedule/BirthdayList`, `parseCalendarMoveRange`), `:1280-1285`, `:1296-1305` (`parseAllCalendarIntents`, `parseCalendarIntent`), `:1424-1449` (`parseAllCalendarIntents`, `parseBirthdayDrafts`, `parseMonthDayDrafts`, `hasForwardDate`, `hasTodayWord`), `:1454` (`sameCalendarDay`), `:1516` (`otherDayClauses`), `:1522-1543` (`_keepRemoteCalendar`: `sameCalendarDay`, `hasTodayWord`, `phraseMentionsDay`, `hasForwardDate`, `looksLikeRecurringCalendar`, `hasCalendarLead`, `todayTaskSource`), `:1622-1627` (`parseCalendarMoveRange`, `looksLikeCalendarReschedule`), `:1661` (`sameCalendarDay`), `:1671` (`parseCalendarIntent`), `:1698` (`dateWithLocalClock`).
- `voice_service.dart:380` (`parseCalendarStartsAt`), `:498` (`parseCalendarIntent`) — локальный фолбэк.
- `calendar_intent.dart:76` — `recurrenceForCalendar` при сборке `CalendarItem`.
- `event_dates.dart:47` — `parseCalendarMoveRange` внутри `parseCalendarDate`; `:941`, `:994-1026` — самовызовы.
- `task_intent.dart:248` (`splitCalendarClauses`), `:252, 271, 290` (`parseCalendarDate`), `:265` (`sameCalendarDay`), `:278` (`hasTodayWord`), `:305`.

**`metric_parse.dart`**
- `app_state.dart:1278` (`matchUserMetrics(work, metrics)`), `:1464-1466` (`matchUserMetrics`, `confineParsedMetricsToCatalog`, `mergeParsedMetrics`), `:901` (`matchUserMetrics` для карточки события), `:3014-3020` (`matchUserMetrics`, `mergeParsedMetrics`, `confineParsedMetricsToCatalog` в `_saveStandaloneMetrics`), `:906, 3024, 3078` (`_matchExistingMetric` → `metricNameKey`/`sameMetricName`-подобные предикаты).
- `voice_service.dart:451-471` (`matchUserMetrics`, `metricsFromText`, `sameMetricName`, `pickUserMetric`), `:476` (`parseVoiceNote(metrics:)`).
- `note_intent.dart:258-261` — `matchUserMetrics(cleaned, metrics)` внутри `splitNoteBody` (куски-метрики вынимаются из заметки).
- `metric_parse.dart` самовызовы: `:113` (использует `event_dates.looksLikePastFact`), `:39-235` внутренняя логика.

**`parse_confidence.dart`**
- `app_state.dart:1274, 1354, 1357, 1482, 1551` — `ParseConfidence.high/medium` и пороги.
- `lib/models/heard_parse.dart:45` — `ParseConfidence.anyLow(confidences)` в `needsConfirmation`.
- `app_state.dart:1121` — `parsed.needsConfirmation(always: alwaysConfirmInput)`.

**`ru_regex.dart`** — используется в `note_intent.dart:157, 165` (вставка в регэкспы); остальные файлы пишут `(?![а-яёa-z0-9])` вручную.

## 2.3. Где парсер и ИИ разбирают один и тот же текст

| Место | Что делает парсер | Что делает ИИ | Как сводятся |
|---|---|---|---|
| `app_state.dart:1203-1242` (`_parseWave1`) | `_parseWave1Local(transcript)` — полный локальный разбор **той же строки**, что уходит в `processText` (`:1225-1232`) | сервер: `analyzeLife(expandedText)` (`ai.ts:540-600`) по тому же тексту | `_heardFromRemote` (`:1375-1502`) мержит: заметка (`noteDraftFromHeard`), метрики (`mergeParsedMetrics`), календарь (правила приоритета), задачи (`expandTaskTitles`) |
| Метрики | `metric_parse.matchUserMetrics` (`metric_parse.dart:39`) | `metrics.ts:316 matchUserMetricsFromText` + модель в промпте (карточки передаются в `process_input.js:171-200`) | Обе стороны: `mergeAnalyzedMetrics` + `confineAnalyzedMetricsToCatalog` — и на клиенте (`app_state.dart:1464-1466`), и на сервере (`ai.ts:652-656`) |
| Заметка | `parseVoiceNote` / `splitNoteBody` | `noteFromLead` (`ai.ts:1955`) при наличии лида, иначе ответ модели (`extractJson:135`) | клиент: `local.note != null` → **локально и без ИИ** (`app_state.dart:1208-1210`); сервер: лид побеждает модель (`ai.ts:697-702`) |
| Календарь | `parseAllCalendarIntents` + `fallbackCalendar`-аналог | `analyzeLife` → `parsed.calendar` | приоритеты: move/cancel → локально (`app_state.dart:1211-1217`), несколько дней недели → локально (`:1442-1444`, `_preferLocalWeekdayCalendar:1505`), `hasForwardDate && !hasTodayWord` → локально (`:1445-1449`), иначе merge по `sameCalendarDay+sameCalendarTitle` (`:1450-1459`); фильтр модели — `keepServerCalendar` (`ai.ts:949`) / `_keepRemoteCalendar` (`app_state.dart:1522`) |
| Задачи | `parseTaskList`/`parseDatedTasks` | `parsed.tasks` | `resolveTasks` (`ai.ts:968-999`): если локальных пунктов больше — локальные; на клиенте — `parseDatedTasks` важнее, иначе `expandTaskTitles(result.tasks)` (`:1411-1414`) |
| STT-текст | — | `runStt` (`providers/run.ts:205`) | клиент использует `original_text` как источник (`voice_service.dart:340-343`, `noteDraftFromHeard(remoteTranscript:)`) |

## 2.4. Есть ли флаг «ИИ ответил → парсеры не трогают»?

**Нет.** Такого флага нет ни на клиенте, ни на сервере. Есть обратная логика — «локальный ответ → ИИ не вызывается вообще»:

Клиент, `_parseWave1` (`app_state.dart:1203-1242`):
1. `:1210` — `if (local.note != null) return local;` — явная заметка **не ходит в ИИ** (комментарий на `:1208-1209`: «модель и отсечение волны 1 успевали обнулить notes»).
2. `:1211-1217` — отмена/перенос/список ДР/«N числа» + непустой локальный календарь → локальный ответ.
3. `:1218-1221` — только календарь без метрик/задач (`calendarOnly`) → локальный.
4. `:1222` — нет удалённой сессии → чистый локальный разбор.
5. `:1238-1241` — исключение от ИИ → локальный ответ (без сообщения об ошибке).
6. Иначе — ответ ИИ (preview) и merge в `_heardFromRemote` (`:1375`).

Ближайший аналог «флага» на уровне парсеров — «стоп-сигнал по заметке»: `calendar_intent.dart:205, 215, 228` начинаются с `if (isPureNotePhrase(text)) return false;` — календарный парсер сам отступает, если фраза целиком заметка. Плюс «чистая заметка» на сервере: `noteOnly` (`ai.ts:639-650`).

Сервер: флага нет вовсе — всегда `applyIntentHeuristics` (`ai.ts:602-735`), т.е. модель **и** локальные фолбэки применяются вместе; приоритет задан правилами, не флагом.
