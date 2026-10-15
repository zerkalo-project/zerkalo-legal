# Диагностика ZERKALO — 3. ИИ (серверный конвейер)

Диагностический отчёт (только чтение, изменения в код не вносились).
Часть 3 из 4: схема вызовов, ветки провайдеров, wave1/wave2, серверные fallback-парсеры.

## 3.1. Схема вызовов (с точками входа)

```
[UI]
  voice_record_sheet.dart:149  → app.enqueueVoice(path)                      (микрофон)
  event_editor_sheet.dart:94   → app.enqueueText(text)                       (текст)
  metric_editor_sheet.dart:107 → voiceService.recognizeFile(path)            (только STT, без разбора)

[AppState]
  enqueueVoice            app_state.dart:922-970
    → voiceService.processAudioRemote(path, preview: true, metrics)   :933-939
    → _heardFromRemote(result.confineToWave(1), transcript, '')       :950-954
    → _runWave1(transcript, alreadyParsed: parsed)                    :955
    (+ обработка 402 SubscriptionRequiredException :956-962, VoiceException :963-964, прочее → recognizeFailed :965-968)

  enqueueText             app_state.dart:972-983
    → _runWave1(transcript, skipConfirm)                              :981

  _runWave1               app_state.dart:1110-1130
    → parsed = alreadyParsed ?? await _parseWave1(spoken)             :1120
    → низкая уверенность/«всегда подтверждать» → pendingHeardConfirmation, стоп :1121-1125
    → _commitWave1(parsed)                                            :1127

  _parseWave1             app_state.dart:1203-1242
    → _parseWave1Local (парсеры)                                      :1207
    → короткое замыкание на локальном ответе                          :1210-1222
    → voiceService.processText(preview: true)                         :1225-1232
    → _heardFromRemote                                                :1233-1237

  _commitWave1            app_state.dart:1566-1619
    → addNote (notes_reminders) :1570-1581
    → _saveStandaloneMetrics (metric_values) :1583-1590 / :3009-3076
    → _applyCalendarDraft (calendar_events) :1592-1598 / :1606-1719
    → _addParsedTasks (day_tasks) :1599-1614 / :1972
    → InputKind-набор → «Записано: …» :1728-1744

[VoiceService]
  processAudioRemote      voice_service.dart:146-205   multipart: preview, client_now, metrics
  transcribeAudioRemote   voice_service.dart:208-246   transcribe_only=true
  processText             voice_service.dart:247-276   JSON: text, metrics, preview, wave=1
  _processInput           voice_service.dart:278-333   «wave»: 1 (сервер игнорирует)
  _resultFromPayload      voice_service.dart:327-413   разбор ответа в VoiceResult
  _localFallback          voice_service.dart:446-525   офлайн-разбор теми же приоритетами
  processRecording        voice_service.dart:120-140   preview=false — НИКЕМ не вызывается

[FunctionsClient]
  functions_client.dart:41,77  POST {FUNCTIONS_URL}/functions/v1/process_input
  BackendConfig.functionsUrl   lib/api/backend_config.dart:36-47  (по умолчанию :5000)

[Server]
  src/index.js:85-96     requireUser → multer(processInputUpload) → processInputHandler
  routes/process_input.js:36
     48-66   loadEntitlements → 402 { error: 'subscription_required' }
     68-115  парсинг body/multipart (text, client_now, preview, transcribe_only, speech_provider, metrics)
     117-156  ветка transcribe_only → сразу { original_text, ... } без разбора
     159-170  uploadInputAudio в Storage (только если есть файл и !preview)
     171-200  каталог метрик: из БД + mergeMetricHints(clientMetrics, dbHints)
     192-211  analysis = confineToWave(await analyzeLife({ text, audio, speechProvider, metrics, now, wave: 1 }), 1)
     210-232  ветка preview → ответ со всеми массивами, БЕЗ записи в БД
     234-247  preview=false → saveNotes + saveStandaloneMetrics
     249-268  финальный JSON
     269-283  catch → empty_input (400) / process_failed (500)

[ai.ts]
  analyzeLife            540-600
     транскрипция при необходимости: runStt (540-560), empty_input (566-569)
     expandSpokenAmounts (572-580)   ← metrics.ts:57
     buildWave1Prompt (582)          ← prompts.ts:22
     runText (583-590)               ← providers/run.ts:300-490
     extractJson (591)               ← ai.ts:135-184 (normalizeNote/Metric/Calendar/Task)
     applyIntentHeuristics (592-595) ← ai.ts:602-735
     confineToWave (592) / result.ai (598)
```

## 3.2. Ветки: модель ответила / упала / офлайн

**Текстовый провайдер** (`providers/run.ts`):
- Цепочка: `AI_TEXT_CHAIN`, по умолчанию `gigachat,yandexgpt` (`registry.ts:19, 41-49`).
- Пул слотов: `AI_TEXT_SLOT_<ID>` (по умолчанию 1), FIFO без обгона, таймаут очереди 30 с (`run.ts:11, 41-45, 163-181`); при переполнении → «ИИ перегружен. Попробуйте ещё раз.» (`:376-379`).
- Ни один не настроен → «ИИ не ответил: GigaChat недоступен, YandexGPT не настроен…» (`:370-379`).
- Модель ответила, но JSON битый → следующий провайдер (`:428-447`); ошибка провайдера → следующий (`:448-475`); если следующих нет → «ИИ не ответил вовремя (…). Попробуйте ещё раз.» (`:444-446, 472-474`).
- Успех → `{ raw, provider }`, лог `[TEXT] <id> ok` (`:413-425`).

**STT** (`run.ts:205-290`): `AI_SPEECH_CHAIN`, по умолчанию `nexara,speechkit` (`registry.ts:18`). Nexara — свой пул ключей `NEXARA_API_KEY[_N]` с очередью 60 с (`providers/stt/nexara.ts:5-106`), таймаут 9 с (`:5`), 402 → «недостаточно средств (нужно минимум 200 ₽)» (`:121-129`). SpeechKit — `YANDEX_API_KEY` + `YANDEX_FOLDER_ID`, таймаут 10 с, из WAV вырезается PCM (`speechkit.ts:7-31, 43-62`). Ошибка первого → второй (`run.ts:270-282`); последний не настроен → явное сообщение (`:283-289`).

**Внутри `analyzeLife`**: транскрипция падает и текста нет → проброс наверх (`ai.ts:560-562`); текст есть → продолжаем на присланном тексте (`:558-561`). Пустой текст → `throw new Error('empty_input')` (`:566-569`) → на сервере 400 «Не расслышал, повторите» (`process_input.js:274-281`).

**На клиенте**:
- 402 → `pendingSubscriptionRequired` + текст про подписку (`app_state.dart:956-962`, `voice_service.dart` обрабатывает `SubscriptionRequiredException`).
- `400 empty_input` → `VoiceException.notHeard` («Не расслышал, повторите», `voice_service.dart:24, 183`);
- прочие ошибки функции → `VoiceException.recognizeFailed` («Не удалось распознать, проверьте интернет», `:25-26`);
- ошибка разбора в `_parseWave1` → тихий откат на локальный разбор (`app_state.dart:1238-1241`);
- нет сессии: голос — сразу `recognizeFailed` (`app_state.dart:926-928`), текст — `_localFallback` (`voice_service.dart:275`);
- таймаут: `AppConstants.inputTimeout` (`app_state.dart:939, 1232`, `_runInputJob` `:1820-1828` → «ИИ слишком долго отвечает. Попробуйте ещё раз.»);
- офлайн-CRUD: очередь `OfflineOp` и delta-курсоры (`lib/services/offline_sync.dart`, `app_state.dart:3787-3870`).

## 3.3. wave1 / wave2

- **Wave 2 не существует.** Явно: `ai.ts:93-94` — `@deprecated Wave 2 removed; any value is treated as wave 1`; `confineToWave(result, _wave?)` игнорирует аргумент (`:110-129`).
- Всё, что «волна 1», — это отсечение лишнего: `confineToWave` обнуляет `events` и `sphere_updates` и пересчитывает `intent`; плюс клиентский дубль — `VoiceResult.asWave1()` (`voice_result.dart:67-82`) и алиас `confineToWave(int?)` (`:85-88`).
- Клиент **всегда** шлёт `'wave': 1` (`voice_service.dart:294-295` с комментарием «Always wave 1; server ignores other values»), сервер всё равно передаёт `wave: 1` в `analyzeLife` (`process_input.js:199`).
- Логи-маркеры: `[AI] волна 1 модель вернула` / `волна 1 после отсечения` (`ai.ts:98-107`), `[AI] волна 1 вернула: metrics=… calendar=… tasks=… notes=… events=…` (`voice_service.dart:425-441`), `[AI] волна 1: игнорирую events=N` (`voice_service.dart:438-440`, `app_state.dart:1380-1384`).

## 3.4. Где на сервере подключаются fallback-парсеры

Всё в `applyIntentHeuristics` (`ai.ts:602-735`), который выполняется **после** ответа модели:

| Шаг | Строки | Функция |
|---|---|---|
| Лид заметки | 631-636 | `extractNoteSentence`, `stripNoteLeadSentences`, `noteLeadBody`; `workText = leadBody` |
| «Чистая заметка» | 638-650 | `noteOnly` → только заметка |
| Метрики | 652-656 | `matchUserMetricsFromText` (`metrics.ts:316`), `mergeAnalyzedMetrics`, `confineAnalyzedMetricsToCatalog` |
| Факт vs задача | 658-663 | `looksLikeLifeEvent:1688`, `TASK_LEADING:740`, `TASK_LEAD_ANYWHERE:745`, `resolveTasks:968` |
| Календарь | 664-695 | `fallbackCalendar:1389`, `keepServerCalendar:949`, `sameCalendarEntry:840`, `collapseMovedCalendar:1500`, `dropUnmentionedToday:1526` |
| Заметка из лида | 697-702 | `noteFromLead:1955` / `resolvedNote:1783` |
| Прочее | — | `splitNoteBodyClauses:1899`, `clauseMentionsTitle:1888`, `fallbackNote:1975`, `fallbackNoteFromBody:1795`, `parseNoteRemindAt:1988`, `monthDayCalendarDrafts:1448`, `calendarFromClause:1615`, `todayTaskSource:880`, `otherDayClauses:874`, `looksLikeTrackerToday:902`, `isExplicitOtherDay:855` |

Второй серверный слой — в маршруте, но он **недостижим с текущим клиентом**:
- `process_input.js:286-293` `notesToSave` → `isNoteLeading` / `fallbackNote`;
- `process_input.js:234-247` `saveNotes` (`:295-332`) и `saveStandaloneMetrics` (`metrics.ts:598`);
- оба вызываются только при `preview=false`, а клиент всегда передаёт `preview: true` (`app_state.dart:936, 1230`; `voice_service.dart:167`); единственный путь с `preview: false` — `voiceService.processRecording` (`voice_service.dart:120-140`), и у него **нет ни одного вызова** ни в `lib/`, ни в `test/`.
