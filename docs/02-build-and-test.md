# 2. Сборка и тесты

## Инструментарий

| Компонент | Версия/инструмент |
|-----------|-------------------|
| Ядро | Go, `gomobile bind` → AAR (Go 1.27: флаг `-checklinkname=0` — linkname-директивы anet) |
| Android-сборка | Gradle 8.11.1, AGP 8.9.1 (CI выровнен на неё на этапе 9) |
| UI | Flutter/Dart, платформенный слой Kotlin |
| Сборка приложения | Release: R8 + боевая подпись (тема A8/fix-release-keys: signing-конфиг генерирует configure — `TNP_SIGNING_PROPERTIES`, `TNP_KEY_KIND=std\|rec`, сверка отпечатка `TNP_SIGNING_KEY_SHA256` и прав файлов ≤0600 задачей verifySigning; release без боевой подписи отвергается — см. docs/RELEASE-KEYS.md основного репозитория); имя `equality-{global|slug}-{std|rec}-v{версия}.apk`, applicationId `io.equality.<sanitized-slug>.x<hash8>` (глобальная — `io.equality.global.x<hash8>`), label `Eq-<hash8>` |
| Эмуляторы | AVD x86_64 + KVM (arm64 падал в NDK-трансляции — используйте x86_64) |

## Команды проверки (ядро)

- `go test ./...` — полный набор, должен быть зелёным.
- `go test -race ./core/... ./cons/... ./mobile/... ./net/...` — гонки; этап 9 закрыл три высокие (Snowball, cons.Clock, cheapCheck/StateRoot), сейчас чисто.
- Race обязателен для `net/query` после любых правок read-слоя.

## Проверки приложения

- `flutter analyze` — чисто.
- `flutter test` — виджет-тесты (включая отложенные переводы и PIN-правила этапа 9).
- Instrumented-тесты на эмуляторе (API 36, KVM): аттестация, Keystore, биометрия — 5/5 на этапе 6.
- Тесты приманки/тревожного PIN: блокировка, приманка, возврат к реальному кошельку.

## CI

- `android.yml` — сборка Android (AGP 8.9.1, gomobile bind `-checklinkname=0`).
- `cons.yml` — консенсус.
- `core.yml` — `go test -race ./core/...`.
- `net.yml` — `go test ./net/...` + `go vet ./cmd/...`.
- `flutter.yml` — `flutter analyze`/`flutter test` + сборка APK (Flutter 3.47.9).
- CI не гоняет эмуляторные прогоны — они выполняются вручную по ранбукам.

## Известный флак

`TestSideChannelPublishPropagates` (net) — таймаут 30 с при работающем эмуляторе (CPU-нагрузка swiftshader). На чистых прогонах без эмулятора — зелёный. При падении перезапустить без эмулятора; регрессом без повтора не считать. Аналогичный timing-фактор под нагрузкой у `TestSyncAnchorContinuation` (net) — долг 8-live; в одиночном прогоне зелёный.

## Тестовая сеть и симуляции

- `cmd/tnp-testnet` — генерация genesis-пакета, манифеста, seed-списков, приглашений (invite_only) и tx-драйверы; режим «флота» — 48 persistent-узлов на одном хосте.
- `cmd/tnp-node` — headless-узел (~12 МБ RSS; 48 узлов <1 ГБ).
- Эпохи для тестов ускоряются параметром `epoch_seconds` (продакшен — 24 ч); PoW-таргеты в тестах занижены.
- Сдвиг часов эмулятора: `adb shell cmd alarm set-time`. Сдвиг часов хоста для длинных флотов — контролируемая операция с остановкой timesyncd и **обязательным возвратом** (см. [Процесс](04-development.md)).
- Память для полного прогона: 2 эмулятора (~3,8 ГБ каждый) + флот узлов + CLI ≈ 10–13 ГБ RAM; при нехватке сокращать headless-узлы, а не эмуляторы. Gradle-демоны перед прогоном останавливать.
- Ранбуки и результаты — `reports/` основного репозитория (`reports/stage8/TESTNET-RUNBOOK.md`, отчёты этапов, артефакты прогонов 50 узлов).

## Блок 8-live (только живой телефон)

Bluetooth-платёж и Bluetooth-vouch, аттестация на реальном железе, SQLite на устройстве, установка release-APK, посевной круг из нескольких людей, тест принуждения, замеры ресурсов (батарея/трафик), NSD/mDNS в реальном Wi-Fi. Чек-листы — в `reports/stage6/`, `reports/stage7/`.

→ [Конспект протокола](03-protocol.md)
