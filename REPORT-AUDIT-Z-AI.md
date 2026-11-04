# REPORT-AUDIT-Z-AI — независимый аудит equality-programming-docs

Аудитор: сторонний (мандат `audit-z-ai`). Дата: 04.11.2026.
База: ветка `audit-z-ai` = `origin/main` = `aa21c16` («docs 06:
развёртывание сети…»). Сверка с `/equality-trust-networks` = `0e2c342`
(та же дата; далее **CODE**), сам репозиторий — **DOC**.

Метод: инвентаризация DOC; построчная сверка команд, флагов и артефактов
с CODE (`cmd/tnp-testnet/configure.go`, `main.go`, `fleet.go`,
`export.go`; `cmd/tnp-node/main.go`; `go.mod`; CI; gradle-конфиги;
`core/*`, `cons/*`; `docs/KNOWLEDGE.md` v9.16). Файлы не изменялись.

## Сводка

| Критичность | Кол-во | Кратко |
|---|---|---|
| 🔴 критично | 2 | пример `configure` невыполним (-genesis-time не unix, нет обязательного -slug); выход конфигуратора описан неверно |
| 🟠 важно | 5 | «те же артефакты» у generate; Snowball α 0.7 vs 0.6 в коде; fee_min 0,01 vs 1; формула слота до mint-fix; R8+debug-подпись устарело |
| 🟡 желательно | 10 | CI-список, путь ранбука, «1..7» шаблонов, PoW-таргет 2^24, имя API моста, v9.13-семантика, акцент слотов, версия спеки v9.11, статус разработки, мелочи |

**Главный вывод:** docs-репозиторий структурно здоров и на удивление
точен в таблицах флагов `tnp-node`/`fleet` (12/12 флагов сверены),
но отстал от спеки на пять минорных версий (v9.11 → v9.16), содержит
один неработающий пример-флагман (configure) и неверное описание
артефактов конфигуратора, плюс транслирует два консенсусно-значимых
числовых расхождения из спеки/комментариев кода (α, fee_min).

---

## Находки

### 🔴 AUDIT-Z-DOC-01. Пример команды `configure` в docs 06 невыполним (два дефекта)

**Файлы:** `docs/06-network-deployment.md:27-33` против
`cmd/tnp-testnet/configure.go:35` и `configure.go:62-70` (CODE).

1. `-genesis-time 2026-11-04T12:00:00Z` — флаг объявлен как **Int64
   unix-секунды** (`fs.Int64("genesis-time", 0, "genesis-время, unix
   (0 = сейчас)")`); RFC3339-значение отваливается на парсинге флага.
2. В примере не передан **обязательный `-slug`**: для не-глобальной сети
   `if slugName == "" { return fmt.Errorf("нужен -slug …") }`; переданный
   `-network-id` — это «метка сети для детерминизма ключа приглашения»,
   не идентификатор сети.

Список параметров в `docs/06:35` не содержит также `-global`, `-recurring`,
`-providers`, `-provider-seed`, `-period-seconds`,
`-transfer-moratorium-days`, `-mint-challenge-window-hours`,
`-migrate-delay-days`, `-logo`, `-app-version`, `-signing-dir` (все есть в
`configure.go:25-47`).

**Воспроизведение:** скопировать пример из docs 06 → `flag provided but
not defined`/ошибка парсинга; убрать `-genesis-time` → `нужен -slug …`.

**Предложение:** переписать пример (unix-время + `-slug`) и дополнить
таблицу флагов; документ по развёртыванию обязан быть копипаста-запускаемым.

### 🔴 AUDIT-Z-DOC-02. Выход конфигуратора описан неверно

**Файл:** `docs/06-network-deployment.md:37` против
`cmd/tnp-testnet/configure.go:260-355`, `core/wire/genesis.go:461-463`
(CODE).

Docs утверждают: genesis.bin = «пакет: genesis-запись + манифест +
**подписанный seed-список**», плюс «таблица слотов челленджа посевного
круга» и invite «QR/ссылка». Факт:

- `genesis.bin` = ровно манифест + genesis-запись
  (`GenesisPackage` — 2 поля; подписанного seed-списка внутри нет —
  `seedlist.bin` пишет только режим `generate`, `main.go:492-498`);
- таблицу слотов печатают `generate`/`slots` (`main.go:515, 684`) —
  **configure не печатает**;
- `invite.txt` у configure — plain text (пакет, manifest_hash,
  invite_key; `configure.go:295-305`); `tnp://invite?…`-ссылку формирует
  только `generate` (`main.go:505-510`);
- не упомянуты фактические артефакты `guarantees.json`
  (`configure.go:279-294`), `android.json` (`configure.go:329-355`),
  каталог `icon/`.

**Предложение:** переписать секцию выхода по фактическому списку;
развести описание `configure` (конструктор сети) и `generate`
(генерация с seed-списком) — сейчас они смешаны.

### 🟠 AUDIT-Z-DOC-03. «Генерация без шаблона — те же выходные артефакты» — неверно

**Файл:** `docs/06-network-deployment.md:39` против
`cmd/tnp-testnet/main.go:354-518` (CODE).

`generate` пишет `genesis.bin, manifest.json, seedlist.bin, invite.txt`
и печатает таблицу слотов — **не** создаёт `guarantees.json/android.json/icon`
(это только configure). Список флагов неполон: у `generate` ещё `-out`,
`-genesis-time`, `-epoch-seconds`, `-small-pow`, `-argon2-mib`,
`-fresh-window`, `-anchor-epoch-zero` (`main.go:355-369`).

### 🟠 AUDIT-Z-DOC-04. Snowball α: docs 0.7, код 0.6 (комментарий кода тоже 0.7)

**Файлы:** `docs/03-protocol.md` («k=10, α=0.7, β=4») и
`docs/06-network-deployment.md` («Snowball (k=10, α=0.7, β=4)») против
`cons/params.go:50` (CODE) — `SnowballAlpha: 0.6`, при том что
собственный комментарий `cons/params.go:21-22` и симуляция
(`sim/sim.go:138`) — 0.7, и `docs/development-plan.md` называет
«α = 0.7» утверждённым решением.

Docs соответствуют комментарию, но не фактическому runtime-значению.
Консенсусный параметр безопасности — требуется решение владельца (см.
QUESTIONS) и синхронная правка либо кода, либо документации обоих
репозиториев.

### 🟠 AUDIT-Z-DOC-05. `fee_min`: docs 0,01 — код 1

**Файл:** `docs/03-protocol.md` («`fee_min = 0,01`»; наследует
`CODE/docs/KNOWLEDGE-4.md:89`) против `core/params/params.go:118` и
`core/profiles/profiles.go:42` (CODE) — `FeeMin: 1`.

Единицы профиля целые (IssuanceAmount 1000, DemurrageFloor 3000 = «3
месячных выпуска»), т. е. fee_min = 1 целая единица, а не 0,01.
Расхождение есть в самой спеке; docs транслируют без оговорки.

### 🟠 AUDIT-Z-DOC-06. Формула слота челленджа — семантика до mint-fix (v9.14)

**Файл:** `docs/03-protocol.md` (слот = `H(address ‖ period ‖ hash(чекпоинта
предыдущей эпохи))`) против `core/ledger/validate.go:176-188, 263-292`
(CODE).

Код считает `H(address ‖ period)` (`hashThree`); включение хеша чекпоинта
— открытый вопрос, PENDING за владельцем (комментарий в коде). Плюс с
v9.14 якорь слота — **текущий период выпуска** (`mint_slot_mode`,
`MintWindowAnchor`, `protocolVersion = 3`: `core/params/params.go:210`).
Docs описывают модель до mint-fix.

### 🟠 AUDIT-Z-DOC-07. «Release: R8 + debug-подпись» устарело (после A8/fix-release-keys)

**Файл:** `docs/02-build-and-test.md` против
`app/android/app/build.gradle.kts:28-184, 217-265` (CODE).

Release-сборка требует боевой signing-конфиг: без
`TNP_SIGNING_PROPERTIES`/`TNP_KEY_KIND=std|rec` — GradleException;
задача `verifySigning` сверяет SHA-256 файла ключа с
`TNP_SIGNING_KEY_SHA256` и права ≤0600; release-артефакт без боевой
подписи отвергается. Ни TNP_SIGNING_*, ни verifySigning, ни
`docs/RELEASE-KEYS.md`, ни `~/.tnp-signing` в DOC не упомянуты.

### 🟡 AUDIT-Z-DOC-08..15. Устаревшее и неточности (сводно)

- **CI-список неполон** (`docs/02`): перечислены android.yml/cons.yml;
  фактически 5 workflows: + core.yml (go test -race ./core/...), net.yml
  (./net/... + vet cmd/), flutter.yml (analyze/test + сборка APK,
  flutter 3.47.9).
- **Путь ранбука** (`docs/06:104`, `docs/02:43`): файл в
  `CODE/reports/stage8/TESTNET-RUNBOOK.md`, а не `reports/TESTNET-RUNBOOK.md`.
- **«Список шаблонов (1..7)»** (`docs/06:24`): шаблонов 8 (профиль №8
  subscriber-resource добавлен этапом 12; `configure.go:430`).
- **PoW-таргет** (`docs/01:49`): «лёгкий SHA-256 (таргет 2^24…)» —
  путаница таргета с числом итераций: таргет 2^40 ≈ 2^24 итераций
  (`core/params/params.go:180`, правка A4).
- **Имя мостового API** (`docs/01:25`): окно челленджа для UI —
  `MintWindowJSON` (`mobile/query.go:903+`), а не
  `CheckMintChallengeWindow` (метод ядра; через мост недоступен).
- **v9.13-семантика времени отсутствует** (`docs/03`): docs описывают
  «допуск ±30 мин от сетевого времени»; фактически валидация ядра — от
  `txNow = timestamp_claim`, C9 — слой допуска сети к часам узла
  (`core/ledger/validate.go`, комментарий Apply; `net/gossip.go:369+`);
  protocolVersion 3 нигде не назван.
- **«Слоты 3/5/10»** (`docs/03`): акцент — 3 (≥90 дн.) и 5 (≥365 дн.)
  регулярные тиры, 10 — только посевному кругу в первый год; оговорка в
  docs есть, но читается как три регулярных тира.
- **Версия спеки v9.11** (`README.md:7,31`): актуально v9.16
  (`CODE/docs/KNOWLEDGE.md:8`); 9.12 (аттестация A1/A2/A12), 9.13
  (txNow), 9.14 (mint-fix), 9.15, 9.16 — не отражены.
- **Статус разработки** (`README.md:31`): обрывается на этапе 12; в
  плане/коде уже выполнены этапы 13-16, темы A3/A5/A6/A8/A10,
  audit-frontier-pass, fix-f5-f16, fuzz-debts, mint-fix,
  fix-cp-threshold, fix-mint-window, fix-release-keys, pre-live-audit.
- **Мелочи:** имена промптов «stage-N-prompt.md» — схема приблизительна
  (факт: `stage-11-prep-prompt.md`, `a8-keys-prompt.md`, …);
  «чек-листы в reports/stage6/, stage7/» — там замеры и скриншоты,
  выделенных чек-листов 8-live нет; локальный каталог сим-репо
  (`~/projects/anti-farm-simulation`) отличается от имени GitHub-репо
  (equality-anti-farm-simulation) — читатель может спутать.

### 🟡 AUDIT-Z-DOC-16. Пробелы покрытия (важные темы без документации)

- **MIGRATE** (миграция ключей): только name-drop; нет процесса
  propose → согласия гардианов → окно `migrate_delay_days`, nonce,
  режимов `migrate -mode propose|cancel`.
- **Восстановление/recover**: карточка восстановления, сценарий
  «кража → восстановление», подписи гардианов — не описаны.
- **Герметичность release-сборки**: см. AUDIT-Z-DOC-07 (полный пробел).
- **PoW-детали**: нет RepeatAndroidIDTarget, правил песочницы
  (нулевой таргет вне sandbox — невалидный genesis), `-small-pow`/
  `-argon2-mib`.
- **Eclipse/DoS**: link-groups упомянуты, но нет `-no-announce`/
  `-force-public` семантики, net/dos.go, eclipse-режима tnp-testnet.
- **Режимы tnp-testnet**: transfer, cancel, migrate, redeem, findlabels,
  flood, eclipse — не документированы (для «документа развёртывания»
  отсутствие transfer/cancel/redeem заметно).
- **`cmd/tnp-sandbox-peer`** — не упомянут.
- **Модули pool/redeem/contribution** — типы перечислены, семантика,
  флаги и лимиты (этап 10) нет.
- **Лёгкий узел** — одна строка; нет протокола доказательств
  (StateProof/SMT) и границ доверия.
- **Два Android-артефакта**: корневой `android/` (Kotlin-shell,
  `platform.tnp.android`) и Flutter `app/` (applicationId из
  `TNP_APPLICATION_ID`) — слои описаны, различие APK-сборки нет.

---

## Проверено и чисто (docs ↔ код совпадают)

- **Инструментарий**: Go 1.27 + `-checklinkname=0`; Gradle 8.11.1 /
  AGP 8.9.1; пути сборки `go build -o bin/tnp-testnet ./cmd/tnp-testnet`,
  `./cmd/tnp-node`; `go test ./...` и race-набор соответствуют CI.
- **Флаги `tnp-node` — 12/12 сверены** с `cmd/tnp-node/main.go:50-69`
  (genesis-file/peer/relay/force-public/no-announce/level light + own-wallet/
  wallet+wallet-domain/db/status-file/link-groups/import-db); поля
  status-file (peers/journal_len/state_root/checkpoints) верны.
- **Флаги fleet** (`-genesis -out -nodes 48 -seeds -base-port -bin`) —
  валидны (`fleet.go:225-238`), дефолт 48 узлов.
- **Посевной круг/приём/контроль**: slots/mint/publish/evidence,
  register/vouch, state/export — команды и флаги совпадают с usage
  `main.go`.
- **network_id = H(manifest ‖ genesis)**, глобальная сеть-константа,
  slug-метка, Eq-метка — совпадают с `core/wire/genesis.go` и
  `core/crypto/address.go`.
- **Fingerprint-манифест (v9.11)**: SeedMember = только PublicKey,
  провайдеры — fingerprint'ы, `-provider-seed` — совпадают.
- **Recurring (v9.10-9.11)**: периодная метка, сгорание после границы
  P+1 (`E_BURNED`), без сброса баланса, ограничения unit/transfer_mode,
  genesis-валидация — совпадают.
- **quota_reset/окна M5/M6**, **чекпоинты** (эпоха 24 ч, M=21/N=14, SMT-корень,
  режимы + переходный посевной, seed_sunset 100/24, BuilderQueue №42),
  **Snowball k=10/β=4** (кроме α), **подписанный ответ голосования** —
  совпадают.
- **update_guardians**: окно 48 ч, лимит 150, ≤3 личных, отмена PIN-only —
  совпадают.
- **Экономика**: rate ∈ [0.5; 2]%, адаптив 15% за K=6, шаг 0,05 п.п.,
  демередж 1%/floor 3000, P=3000/P_max=30000/D=48 ч, R=10%×12, K=3,
  возраст поручителя ≥90 дн. — совпадают.
- **Bootstrap-каскад**, invite_only, BT = передача подписанной tx —
  совпадают.
- **Sandbox-корни vs корни Google (A2)**, детерминированные ключи
  tnp-testnet, allowBackup=false, экраны приложения — совпадают.
- **Имена APK/applicationId/label** (`io.equality.<slug>.x<hash8>`,
  `Eq-<hash8>`) — совпадают с `cmd/tnp-testnet/android.go` и `wire.EqLabel`.
- **docs 05 (отказ от слияния сетей)**: дизайн-файл и ветка `audit`
  существуют; изоляция сетей подтверждена кодом.
- **docs 04**: все файлы-источники процесса существуют; правило
  «чекпоинты сессии CLI не включать в промпты» соответствует решению
  01.11.2026.

## QUESTIONS

1. **α: 0.6 или 0.7?** (совместно с находкой AUDIT-Z-04 отчёта tnp)
2. **fee_min: 1 единица или 0,01?** (совместно с tnp QUESTIONS-4)
3. **Слот челленджа: хеш чекпоинта в формуле** — PENDING в коде; канонизировать.
4. **Доки 06 копипаста-запускаемы?** — прямая правка, но зафиксировать
   правило «каждый пример конфигуратора проверяется прогоном» в docs 04.
5. **Чек-листы 8-live**: где канонические файлы (docs 02 ссылается на
   несуществующие чек-листы)?
6. **Какой API считать публичным контрактом моста** — `MintWindowJSON`
   (синхронизировать формулировку в docs 01).

---

*Отчёт в ветке `audit-z-ai`; файлы репозитория не изменялись.*
