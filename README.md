# Music Capsule

Офлайн-трекер музыки и аналог Spotify Wrapped для локальной FLAC/MP3 медиатеки.
Приложение в фоне перехватывает воспроизведение из любых плееров (Poweramp, AIMP,
Musicolet, Symfonium и др.), собирает аналитику в Room и генерирует анимированные
Stories-капсулы за месяц/год с фото артистов из Spotify и текстами от Gemini.

## Стек

- Kotlin (JVM 17), Jetpack Compose + Material 3 (Material You)
- Clean Architecture + MVVM: UI -> ViewModel -> Repository -> Room / Remote
- Room (SQLite) + Flow/Coroutines, DataStore Preferences
- OkHttp + Kotlinx Serialization, Coil, AndroidX Palette, Navigation Compose
- Min SDK 26, Target SDK 34

## Структура

```
app/src/main/java/com/musiccapsule/app/
├── MusicCapsuleApp.kt            # Application + ручной DI
├── MainActivity.kt
├── core/
│   ├── di/AppContainer.kt        # граф зависимостей
│   └── util/                     # нормализация артистов, слоты времени, даты
├── data/
│   ├── local/                    # Room: entities, DAO, database
│   ├── remote/spotify/           # Client Credentials Flow, /v1/search
│   ├── remote/gemini/            # generateContent: алиасы, roast, архетип
│   ├── repository/               # StatsRepository (вся аналитика), ArtistRepository
│   └── settings/                 # DataStore: ключи API, чёрный список плееров
├── domain/model/                 # MonthlyCapsule, TopTrack, TopArtist, ...
├── tracking/
│   ├── MusicListenerService.kt   # NotificationListenerService + MediaSessionManager
│   ├── PlaybackStateMachine.kt   # дедупликация, паузы, порог 30с/50%, анти-сон
│   └── CoverCache.kt             # обложки из метаданных -> filesDir/covers/{md5}.jpg
└── ui/
    ├── dashboard/                # стрик, тепловая карта 365 дней, минуты
    ├── history/                  # LazyColumn + stickyHeader + свайпы
    ├── capsule/                  # HorizontalPager, 5 слайдов, Palette-градиент, шеринг PNG
    ├── settings/                 # ключи API, доступ к уведомлениям, список плееров
    └── components/               # Heatmap, StoryProgressBar
```

## Сборка

1. Открыть папку проекта в Android Studio (Hedgehog+) — Gradle-обёртку IDE создаст сама,
   либо выполнить `gradle wrapper --gradle-version 8.9`.
2. `./gradlew :app:assembleDebug`

## Разрешение на перехват

Нужен доступ к уведомлениям: кнопка «Выдать» в настройках приложения, либо через adb:

```
adb shell settings put secure enabled_notification_listeners \
  com.musiccapsule.app/com.musiccapsule.app.tracking.MusicListenerService
```

## Логика засчитывания

- Пауза не считается: счётчик накапливает только время PLAYING.
- Запись фиксируется при смене трека, если прослушано >= 30 сек или >= 50% длительности.
- Анти-сон: один трек по кругу в 02:00–06:00 при погашенном экране более 3 часов —
  событие помечается `is_sleep_session` и не портит статистику.
- Стрик дней: >= 15 минут суммарного прослушивания в день.
- FLAC Flex: объём = суммарное время x 950 кбит/с (средний битрейт FLAC).
