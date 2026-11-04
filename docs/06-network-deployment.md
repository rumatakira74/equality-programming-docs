# 6. Развёртывание сети

Последовательность запуска новой сети протокола: конфигурация → genesis-пакет → узлы → посевной круг → приём участников → контроль. Все команды выполняются из корня основного репозитория; утилиты — `cmd/tnp-testnet` (конфигуратор и драйверы) и `cmd/tnp-node` (headless-узел). Ссылки на разделы спецификации — `docs/KNOWLEDGE-*.md` основного репозитория.

## 0. Предварительные условия

- Go-инструментарий и сборка ядра — см. [Сборка и тесты](02-build-and-test.md).
- Сборка утилит:

```sh
go build -o bin/tnp-testnet ./cmd/tnp-testnet
go build -o bin/tnp-node    ./cmd/tnp-node
```

- Для участников сети — приложение (APK), собранное под конкретную сеть: `equality-{global|slug}-{std|rec}-v{версия}.apk`, applicationId `io.equality.<sanitized-slug>.x<hash8>`, label `Eq-<hash8>` (раздел 2).
- Решение о параметрах (шаблон профиля, посевной круг, пороги) принимается по спецификации; конфигуратор лишь собирает пакет из валидной комбинации — genesis-валидация отклоняет несовместимые сочетания.

## 1. Конфигурация сети и genesis-пакет

Конфигуратор — режим `configure` утилиты `tnp-testnet` (раздел 5 спецификации):

```sh
# список шаблонов профилей (1..8; №8 subscriber-resource — recurring)
./bin/tnp-testnet configure -list

# сеть по шаблону профиля (-slug обязателен; -genesis-time — unix-секунды,
# 0 = сейчас; полный список флагов печатает -list/флаг без значения)
./bin/tnp-testnet configure -template 1 \
  -slug my-network \
  -network-id my-network \
  -seeds alice,bob,carol \
  -genesis-time 1762257600 \
  -display-name "Моя сеть" \
  -out net/
```

Параметры: `-slug` (обязателен для не-глобальной сети), `-network-id` (метка для детерминизма ключа приглашения; по умолчанию — slug), `-seeds` (метки посевного круга), `-genesis-time` (unix-секунды, 0 = сейчас), `-issuance-amount` (выпуск за период), `-vouch-k` (поручительства на вход), `-epoch-seconds` (период; продакшен — 24 ч, тесты ускоряются), `-resource-id` (для recurring-профиля №8), `-display-name`, `-global` (глобальная сеть), `-recurring`, `-providers`/`-provider-seed` (fingerprint'ы поставщиков), `-period-seconds`, `-transfer-moratorium-days`, `-mint-challenge-window-hours`, `-migrate-delay-days`, `-logo`, `-app-version`, `-invite-key-hex` (ключ приглашения invite_only), `-signing-dir` (каталог боевых ключей подписи APK).

Выход в `-out` (аудит z-ai: описано по факту configure.go): `genesis.bin` (пакет: манифест + genesis-запись; seed-списка внутри нет), `manifest.json`, `invite.txt` (plain text: genesis-пакет, manifest_hash, invite_key; QR/ссылку `tnp://invite?…` формирует только `generate`), `guarantees.json` (уровень гарантий из манифеста, 3.3), `android.json` (applicationId/label/имя APK: `Eq-<hash8>`, `io.equality.<slug>.x<hash8>`), каталог `icon/`. Таблицу слотов челленджа печатают режимы `generate` и `slots` — не configure. Идентификация сети: `network_id = H(manifest ‖ genesis)`, slug, label `Eq-<hash8>`.

Базовый вариант без шаблона — режим `generate`: пишет `genesis.bin, manifest.json, seedlist.bin, invite.txt` и печатает таблицу слотов (guarantees.json/android.json/icon не создаёт — это только configure). Флаги: `-network-id`, `-seeds`, `-peer`, `-invite-key-hex`, `-out`, `-genesis-time`, `-epoch-seconds`, `-small-pow`, `-argon2-mib`, `-fresh-window`, `-anchor-epoch-zero`.

Важно: ключ приглашения хранится в genesis-записи и **не публикуется в манифесте** (spec/02 2.1). `genesis.bin` раздаётся узлам, `manifest.json` можно публиковать.

## 2. Запуск узлов

Узел — `tnp-node`. Минимум для полного узла:

```sh
./bin/tnp-node -genesis-file net/genesis.bin \
  -db node.db -status-file status.json
```

Узел печатает свой Peer ID и multiaddr-адреса — их указывают другим узлам через `-peer` (флаг повторяется). Ключевые флаги:

| Флаг | Назначение |
|------|------------|
| `-genesis-file` | genesis.bin из шага 1 (без него — встроенный тестовый genesis) |
| `-peer` | multiaddr пира для bootstrapping (повторяется) |
| `-relay` | multiaddr relay-узла (для узлов за CGNAT; повторяется) |
| `-force-public` | режим relay-узла: форсировать публичную достижимость |
| `-no-announce` | не анонсировать свои адреса (эмуляция CGNAT) |
| `-level light -own-wallet HEX` | лёгкий узел: проверка доказательств по финализированным чекпоинтам только для своего кошелька |
| `-wallet МЕТКА -wallet-domain seed\|wallet` | кошелёк узла для подписи чекпоинтов и веса голоса |
| `-db` | SQLite (без флага — без персистентности) |
| `-status-file` | JSON-статус (peers, journal_len, state_root, checkpoints) |
| `-link-groups` | несвязанность пиров по подсетям /24, /64 (№35): продакшен — true (по умолчанию), localhost-флот — false |
| `-import-db` | восстановление журнала из старой SQLite до старта сети |

Посевные узлы запускаются первыми; их адреса раздаются остальным через `-peer`. Для сетей, где часть узлов за CGNAT, поднимается relay-узел (`-force-public`), остальные подключаются к нему через `-relay`.

## 3. Посевной круг: выпуск и слоты

- Слоты челленджа посевного круга (окна, в которые разрешён выпуск): `./bin/tnp-testnet slots -genesis net/genesis.bin`.
- Выпуск равного периодического права: `./bin/tnp-testnet mint -genesis net/genesis.bin -seed alice -period 1 -peer <адрес>...` — посевной кошелёк публикует MINT завершённого периода.
- Отладка эквивокаций: `mint -save файл -no-submit` (сохранённая вторая ветка) → `publish` → `evidence -txa -txb`.

Все tx-режимы принимают `-peer` (пиры узлов), `-wait` (ожидание после публикации), `-db`, `-node-seed`.

## 4. Приём участников

Вход в закрытую сеть — invite_only с поручительством (invite_only_vouch):

1. Участник получает приглашение из `invite.txt` (QR/ссылка) и устанавливает APK данной сети.
2. REGISTER: кошелёк регистрируется с аппаратной аттестацией и усиленным PoW (в песочнице — синтетическая аттестация). Валидация аттестации (v9.12, этап 14) у каждого узла при apply: SPKI сертификата-листа обязан совпадать с регистрируемым ключом (`E_ATTEST_KEY_MISMATCH`), цепочка — до одного из двух корней Google, синтетические корни принимаются только при `sandbox = true` (не-sandbox сеть с синтетическим корнем не запускается — и headless-утилита тоже: test-only флаг `AllowSyntheticRoots` для тестов, прод-путь его не выставляет); срок листа проверяется на момент `timestamp_claim` регистрации: `./bin/tnp-testnet register -genesis net/genesis.bin -wallet dave -peer ...` (флаги `-guardians`, `-android-id`).
3. VOUCH: члены посевного круга поручаются за кандидата — нужно `vouch_k` поручительств: `./bin/tnp-testnet vouch -genesis net/genesis.bin -seed alice -candidate <hex> -peer ...`.

После регистрации участник — равный кошелёк; его узел может подписывать чекпоинты (`-wallet` с `-wallet-domain wallet`).

## 5. Контроль работающей сети

- Снимок реестра: `./bin/tnp-testnet state -genesis net/genesis.bin -peer ...`.
- Экспорт для анализа (оборот, предложение, неравенство Gini — из публичного реестра): `./bin/tnp-testnet export -genesis net/genesis.bin -format json`.
- Мониторинг узлов: `-status-file` (peers, journal_len, conflicts, state_root, checkpoints).
- Финальность — Snowball (k=10, α=0.7, β=4) и чекпоинты N-из-M; seed_sunset гасит посевной круг при 100 активных участниках ИЛИ 24 эпохах — далее чекпоинты подписывают только участники.

## Проверка перед стартом: флот на одном хосте

Полный прогон конфигурации без железа — режим `fleet` (48 persistent headless-узлов, <1 ГБ):

```sh
./bin/tnp-testnet fleet -genesis net/genesis.bin -out fleet/ -nodes 48 \
  -seeds alice,bob,carol -base-port 4100 -bin ./bin/tnp-node
```

На localhost: `-link-groups=false` у всех узлов и драйверов (все пиры в одной /24), `epoch_seconds` занижен. Ранбуки и прецеденты прогонов — `reports/stage8/TESTNET-RUNBOOK.md` и отчёты этапов 8/8.1 основного репозитория.

## Ограничения

- Ключи посевного круга и кошельков в `tnp-testnet` детерминированы от меток — это тестовая сеть; **боевые ключи генерируются на устройствах** (Keystore/KeyMint), закрытый ключ не покидает устройство.
- Sandbox-манифест → синтетические корни аттестации (только демо/песочница); продакшен-манифест → оба корня Google — RSA и ECDSA P-384 Remote Key Provisioning (A2, v9.12). Не-sandbox genesis с синтетическим корнем или нулевым PoW-таргетом не запускается (genesis-валидация, v9.12).
- Проверки на живых телефонах (Bluetooth-платёж/vouch, аттестация на реальном железе, посевной круг из нескольких людей и др.) — блок 8-live; без него пилоты не запускаются.

→ [Процесс разработки](04-development.md)
