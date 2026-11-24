# Handoff context: DLNA playback for YummyTV (read, then DELETE this file)

> Для следующего агента: прочитай целиком, затем удали этот файл (`git rm context.md`) и
> продолжай с раздела «Что делать дальше». Общайся с пользователем по-русски.

## Где код

- Ветка `dlna` в форке `https://github.com/Gidrex/YummyTV` (remote `fork`); upstream
  `Helandy/YummyTV` (remote `origin`) — туда НИЧЕГО не пушить.
- Клон: `git clone -b dlna https://github.com/Gidrex/YummyTV && cd YummyTV && git remote add origin-upstream https://github.com/Helandy/YummyTV` (имена remote на новом устройстве могут отличаться).

## Исходное ТЗ пользователя (дословно)

# Задача: добавить в YummyTV воспроизведение на Samsung TV по DLNA

Работаешь в /home/gidrex/git/YummyTV — это локальный клон чужого open-source приложения
(лицензия CC BY-NC 4.0). Создай ветку `dlna`. Ничего не пушь в upstream. Коммит только по
моей просьбе. Общайся со мной по-русски; весь текст в коде (комментарии, строки UI, KDoc) пиши
по-английски. Строго соблюдай AGENTS.md проекта (clean architecture, раскладка domain/data/ui,
строки в ресурсах).

## Цель
В плеере YummyTV, рядом с Chromecast, появляется кнопка «TV (DLNA)». По нажатию текущая серия
продолжает играть на ТВ с той же позиции. Плеер в приложении становится пультом: пауза,
перемотка, громкость, смена серии. Сначала поддержи источники, которые ТВ умеет играть почти
без обработки (ожидаю Kodik). Остальные добавляй по результатам фазы 1.

## Что уже есть и что переиспользовать
Мой рабочий DLNA-клиент для этого ТВ (Kotlin, Android) лежит в
/home/gidrex/Projects/dlna2tv/android/app/src/main/java/io/gitlab/gidrex/dlna2tv/:
- Dlna.kt — SOAP AVTransport/RenderingControl (SetAVTransportURI с DIDL, Play/Pause/Stop, громкость, mute).
- Core.kt — HTTP-сервер для ТВ (HEAD/GET, TimeSeekRange, close-delimited HTTP/1.0), опрос ТВ раз
  в секунду, команды на одном worker-потоке, расчёт позиции.
- Ts.kt — MPEG-TS муксер (H.264 + AAC/ADTS); Mp4.kt + Remux.kt — fMP4 → TS (пригодится для
  источников, которые отдают fMP4/MP4, а не TS).
- PlaybackService.kt — foreground service (mediaPlayback) + WakeLock/WifiLock.
- Util.kt — npt/parseNpt/parseClock/xmlEscape/tag.
Это мой код, копируй свободно. Перед правками прочитай /home/gidrex/Projects/dlna2tv/CLAUDE.md.

## Факты про ТВ (проверены, не перепроверяй с нуля)
- Samsung UE32K5500, Tizen 2.4. Рендерер: http://192.168.3.7:9197/dmr; контрол:
  http://192.168.3.7:9197/upnp/control/AVTransport1 и RenderingControl1. SSDP в этой сети не
  работает, IP фиксированный.
- Играет только MPEG-TS (`video/mpeg`) с H.264 + AAC, ≤1080p, close-delimited HTTP/1.0.
  protocolInfo/contentFeatures: `DLNA.ORG_OP=10;DLNA.ORG_CI=1;DLNA.ORG_FLAGS=01700000000000000000000000000000`.
- SetAVTransportURI отвечает, только когда ТВ уже открыл поток: таймаут 30 с.
- DLNA Seek отклоняется (701). Перемотка = новый URI с другим смещением (уникальный путь на
  каждый перезапуск), с дебаунсом 0.7 с.
- Порядок запросов ТВ: HEAD → пробный GET (ТВ его рвёт через 1–2 с) → HEAD → HEAD с
  `TimeSeekRange.dlna.org: npt=0-` → основной GET. Пробный и основной GET могут идти параллельно:
  статистику веди по последнему соединению.
- RelTime: после нашего нового URI считает с 0. Если ТВ переподключился сам (продолжение после
  паузы, перемотка с пульта) и прислал `npt=X-`, RelTime продолжает идти от X.
  Позиция = (начало первого кадра соединения − X) + RelTime.
- На паузе ТВ рвёт соединение, при продолжении приходит заново с npt.
- Новое устройство ТВ встречает экраном «разрешить устройство»: до подтверждения он застревает в
  TRANSITIONING.
- Android 17: для обращения к ЛВС нужно runtime-разрешение ACCESS_LOCAL_NETWORK (запрашивай
  только на API 37+, minSdk у YummyTV 24).

## Фаза 0 — собрать YummyTV и проверить его как есть
1. Собери debug и поставь на телефон (Nothing A059P, Android 17, adb по Wi-Fi, IP 192.168.3.229;
   порт меняется — `adb mdns services`, затем `adb connect`). UI телефона води через mobile-mcp
   (`mobile_list_elements_on_screen`, батчи команд).
2. Проверь штатно: поиск аниме → тайтл → выбор озвучки/плеера → серия играет в приложении.
   Попробуй минимум Kodik, Alloha и один-два других источника. Запиши, что работает.

## Фаза 1 — разведка форматов (без ТВ)
1. Найди, где у плеера появляется итоговый поток: `PlayerStreamResolveResult.Stream`
   (url, headers, qualities, qualityHeaders, allohaAudioTracks) в feature/player
   (DefaultPlayerStreamRepository, PlayerSourceStreamHandler, PlayerStreamHandler). Временно
   залогируй url и headers для каждого источника.
2. Для каждого источника на ноутбуке с теми же заголовками:
   `ffprobe -headers $'Referer: …\r\n' <url>`, скачай 2–3 сегмента HLS. Выясни:
   - контейнер: TS-сегменты, fMP4 или прогрессивный MP4;
   - кодеки, разрешение, отдельные аудиодорожки;
   - непрерывность таймстемпов между сегментами, наличие EXT-X-DISCONTINUITY;
   - срок жизни ссылок и обязательные заголовки.
3. Сведи в таблицу «источник → формат → что нужно для ТВ»: ничего / склейка TS / перепаковка
   fMP4→TS / не поддержать. Покажи мне таблицу до начала фазы 2.

## Фаза 2 — DLNA-модуль
Отдельный модуль по их архитектуре (например `feature/dlna` с domain/data/presentation или
`core/dlna`, реши по AGENTS.md и объясни выбор). Внутри:
- Сервер на порту 49765 (как в Core.kt). Для HLS с TS-сегментами — стример: парсит m3u8
  (master → вариант по выбранному качеству ≤1080p), по npt находит сегмент через EXTINF, отдаёт
  сегменты подряд одним TS-потоком. Заголовки источника (Referer/Origin/UA) пробрасывай при
  каждом запросе сегмента. Ссылки протухают: умей перерезолвить поток.
- fMP4/MP4-источники — только если фаза 1 покажет, что они нужны; тогда переиспользуй Mp4.kt/Ts.kt.
- DLNA-контрол и опрос ТВ — порт Dlna.kt/Core.kt; позиция по правилу из «Фактов».
- Foreground service на время трансляции.
- IP телефона определяй на каждый запуск (UDP connect к ТВ), а не один раз.

## Фаза 3 — UI
- Кнопка «TV (DLNA)» в плеере (mobile и tv UI). По нажатию: локальный ExoPlayer на паузу,
  поток уходит на ТВ с текущей позиции. Контролы плеера управляют ТВ. Аппаратные кнопки
  громкости телефона меняют громкость ТВ.
- Выход из DLNA возвращает воспроизведение на телефон с позиции ТВ.
- Смена серии или озвучки во время DLNA переотправляет новый поток на ТВ.
- Прогресс просмотра YummyTV продолжает сохраняться по позиции ТВ.
- Ошибки (ТВ недоступен, ждёт «разрешить устройство», формат не поддержан) показывай понятным
  текстом из ресурсов.

## Тестирование
- Юнит-тесты: парсер m3u8 (master/media, EXTINF, относительные URL), выбор сегмента по времени,
  npt/parseClock, правило позиции с npt, пробрасывание заголовков. Плюс тесты, которые требуют
  AGENTS.md.
- Без ТВ: запусти стример на телефоне, скачай с ноутбука `curl http://192.168.3.229:49765/<path> -o x.ts`,
  проверь `ffprobe` и `ffmpeg -v error -i x.ts -f null -` (ошибок быть не должно, кроме последнего
  обрезанного кадра).
- A/B с ноутбука без приложения: отдай файл простым HTTP-сервером на порту 49765 (ufw для ТВ
  открыт) с DLNA-заголовками и вызови SetAVTransportURI/Play через curl. Так отделишь проблемы
  формата от проблем приложения.
- На ТВ: ПЕРЕД каждым тестом, который играет на ТВ, предупреди меня и дождись «готов»: ТВ
  должен быть включён, может понадобиться подтвердить «разрешить устройство». Сценарии:
  1. Старт серии.
  2. Перемотка кнопками и таймлайном.
  3. Пауза и продолжение: позиция не должна прыгать вперёд.
  4. Громкость кнопками телефона.
  5. Смена серии.
  6. Возврат на телефон с правильной позиции.
  7. 10+ минут с выключенным экраном телефона.
  8. Протухшая ссылка: долгая пауза, потом продолжение.
- Логи бери через `adb logcat` по своему тегу; буфер сбрасывается после переподключения adb.

## Отчёт
После каждой фазы — коротко: что сделано, что проверено и чем, что не проверено. Какие
источники заработали на ТВ, какие нет и почему.

Две оговорки:
- Где встраивать кнопку. В YummyTV Cast сделан через обёртку Media3 CastPlayer, и DLNA так красиво туда не ляжет, потому что ТВ не умеет перематывать по команде. Поэтому в промте это отдельная кнопка с режимом «пульта», а не ещё один Player.
- Не проверено, что Kodik отдаёт TS с H.264 + AAC. Это я понял по коду резолвера (/720.mp4:hls:manifest.m3u8). Именно поэтому фаза 1 стоит до написания кода.

## Уточнения пользователя по ходу

- Пушить в форк `Gidrex/YummyTV` пользователь разрешил явно (это не upstream). Коммиты — по-прежнему
  только по просьбе.
- Без nix: JDK ставится системно (`paru -S jdk21-openjdk`), уже стоит на старом устройстве.
- Пользователь сам иногда трогает телефон (например, открывал другое приложение) — это не сбой.

## Состояние на момент передачи (2026-11-24)

### Окружение (старое устройство)
- JDK 21 системный (`/usr/lib/jvm/java-21-openjdk`). Проекту нужен Java 21 (`build-logic/.../ConventionPlugins.kt`).
- `local.properties` (в .gitignore, в репо НЕТ): `sdk.dir=/home/gidrex/Android/Sdk`. На новом
  устройстве создай свой.
- Bash-хук переписывает команды через `rtk`; если auto mode ругается «hook changed this call's input»,
  повтори команду ровно в переписанном виде (с префиксом `rtk`).
- Первая сборка `:app:assembleDebug` шла >10 мин (качались зависимости), повторная — быстро.
  APK: `app/build/outputs/apk/debug/YummyTV-1.31.0-debug.apk`, пакет `su.afk.yummy.tv.debug`.
- Телефон: adb `192.168.3.229:41613` (порт меняется), mobile-mcp device id такой же.
- ВНИМАНИЕ: dlna2tv (`/home/gidrex/Projects/dlna2tv`, приватный gitlab.com/gidrex/dlna2tv,
  ветка `dlna2tv`) лежит только на старом устройстве. На новом — склонировать оттуда.
  ТВ и телефон должны быть в той же сети 192.168.3.0/24; ufw на ноутбуке должен пускать ТВ на 49765.

### Фаза 0 — частично
- [x] Ветка `dlna` создана от `14f5553cc`.
- [x] Debug собран и установлен на телефон, приложение стартует.
- [x] Онбординг пройден: выбран Mobile interface, настройки по умолчанию (Content language English,
  Video quality Best), тур по каталогу пропущен.
- [x] Поиск «Frieren» работает, открыта карточка «Frieren: Beyond Journey's End» (пропущен тур
  «Start watching»).
- [ ] Остановились здесь: телефон заблокировался (локскрин с отпечатком). Нужно попросить
  пользователя разблокировать. Было предложено поставить
  `adb shell settings put system screen_off_timeout 1800000` на время тестов (и потом вернуть
  прежнее значение — сначала прочитать его `settings get system screen_off_timeout`); ответа не было.
- [ ] Проверить штатное воспроизведение: Kodik, Alloha и 1–2 других источника. Записать результат.

### Фаза 1 — подготовка сделана
- Итоговый поток появляется в
  `feature/player/presentation/.../handler/PlayerStreamHandler.kt` (ветка
  `is PlayerStreamResolveResult.Stream`). Там стоит ВРЕМЕННЫЙ лог (тег `DlnaProbe`, помечен
  `TEMP(dlna phase 1)`): src (iframeUrl), url, headers, qualities, qualityHeaders, audio, subs.
  Убрать перед фазой 2.
- Снять логи: `adb logcat -c` перед тестом, потом `adb logcat -d -s DlnaProbe`.
  Длинные сообщения logcat режет ~4 КБ — если qualities не влезут, разбить лог.
- Источники (модули `source/*`): ailiberty, aksor, alloha, amedia, aniboom, anidub, anilibria,
  anime4k, animebesst, animeon, animevost, anistar, ashdi, cvh, dreamerscast, fenix, kodik, mikai,
  moon, rutube, sameband, sibnet, studiomir, torrent, vk, zedfilm (+ availability).
  Экстракторы: `source/<name>/data/.../extractor/<Name>Extractor.kt`.
- Гипотеза по Kodik: URL вида `.../720.mp4:hls:manifest.m3u8`
  (`source/kodik/data/.../KodikExtractor.kt:63,318`). Вероятно, по `.../720.mp4` лежит и
  прогрессивный MP4 — проверить ffprobe; если это обычный (не fragmented) MP4, текущий `Mp4.kt`
  dlna2tv, судя по названию Remux.kt («fMP4 -> TS»), может не подойти — посмотреть Mp4.kt.
  HLS-вариант Kodik, скорее всего, TS-сегменты → «склейка TS» без перепаковки.

### Прочитанный код dlna2tv (для фазы 2)
- `Dlna.kt`: SOAP через OkHttp (quick 3 с / slow 30 с для SetAVTransportURI), `play()` = Stop →
  SetAVTransportURI(DIDL с `protocolInfo=http-get:*:video/mpeg:FEATURES`, duration) → Play.
  setVolume/setMute/volume().
- `Core.kt`: ServerSocket на 49765, поток на соединение; `Playing(path уникален на рестарт)`,
  `Stats(start,out,bytes)` по последнему conn; poller раз в 1 с (GetTransportInfo, GetVolume,
  GetPositionInfo.RelTime); `position = stats.start + tv.rel`; при npt=X: `start = t0 − X`.
  Команды — single-thread executor. `localIp()` через UDP connect к ТВ:9197 на каждый play.
  Ответ — `HTTP/1.0 200 OK`, `Connection: close`, `Content-Type: video/mpeg`,
  `transferMode.dlna.org: Streaming`, `contentFeatures.dlna.org`, при seek — `TimeSeekRange.dlna.org: npt=a-L/L`.
- `Ts.kt`: TsMuxer (PAT/PMT, PID 0x100 video/0x101 audio, PCR с задержкой 0.7 с).
- `Remux.kt`: fMP4 (DASH) → TS со старта с ближайшего ключевого кадра, `Adts`, `annexB`.
- `PlaybackService.kt`: FGS mediaPlayback + PARTIAL_WAKE_LOCK + WifiLock LOW_LATENCY.
- `Util.kt`: parseClock, parseNpt, npt, xmlEscape, tag, chain, parseRange.
- Не прочитаны: Mp4.kt, Net.kt (RangeStream), Player.kt, Headphones.kt, MainActivity.kt.

## Что делать дальше

1. Удалить этот файл.
2. Подготовить окружение на новом устройстве (JDK 21, Android SDK, `local.properties`, adb к
   телефону, клон dlna2tv), собрать `./gradlew :app:assembleDebug`, установить.
   Если устройство не в сети 192.168.3.x — фазы 0/1 на телефоне и все тесты с ТВ невозможны, сказать пользователю.
3. Довести фазу 0: разблокированный телефон → Frieren → серия → проверить Kodik, Alloha, ещё 1–2.
4. Фаза 1: собрать `DlnaProbe`-логи для каждого источника, на ноутбуке ffprobe/скачать сегменты
   с теми же заголовками, свести таблицу «источник → формат → что нужно для ТВ», показать
   пользователю и ЖДАТЬ его решения до фазы 2.
5. Отчитываться коротко после каждой фазы.
