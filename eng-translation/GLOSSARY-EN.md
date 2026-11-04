# Glossary (RU → EN)

Fixed terminology for the `eng-translation/` mirror of this repository. Terms follow the established English texts of the platform specification (`docs/KNOWLEDGE-ENG-1..5.md`, `README-ENG.md` in `equality-trust-networks`); do not invent synonyms. The same glossary is applied uniformly in `equality-user-docs/eng-translation/` (with a reduced, user-facing subset).

Code identifiers, field names, flags, file names, paths, version labels (`protocolVersion`, `v9.16`, `Eq-<hash8>`, `K = 3`) are never translated.

| Русский | English (established) |
|---------|----------------------|
| поручительство | vouching (the mechanism); `vouch` (transaction); pl. vouches |
| поручитель | vouching member; voucher (in the reward context) |
| поручаемый | vouched-for member |
| гардиан | guardian (personal guardians / public guardians) |
| смена гардианов | guardian re-election |
| слот поручительства | (vouching) slot; the pool of slots — slot pool S |
| челлендж выпуска | issuance challenge (informally: mint challenge) |
| выпуск | issuance (process/amount); tx `mint`; the total — emission |
| период | period |
| эпоха | epoch |
| чекпоинт | checkpoint |
| финальность / финализация | finality / finalization |
| мораторий | moratorium |
| демередж | demurrage |
| комиссия (сжигаемая) | burn fee (the fee that is burned; `fee_mode = burn`) |
| посевной круг | seed circle |
| сид-участник / сид-кошелёк | seed member / seed wallet |
| генезис | genesis (genesis package, `genesis_time`) |
| реестр | (public) registry |
| кошелёк | wallet |
| узел | node; полный узел — full node; лёгкий узел — light node |
| аттестация | attestation (hardware attestation; attestation chain; attestation leaf) |
| цепочка записей | chain of signed records / wallet chain |
| эквивокация | equivocation |
| Flagged (метка) | the `flagged` mark |
| режим-приманка | decoy mode |
| тревожный PIN | duress PIN |
| миграция | migration (tx `migrate`) |
| ферма кошельков | wallet farm / farm |
| вычислительный барьер | cost barrier |
| отложенный перевод | delayed transfer (threshold P, delay D = 48 h) |
| регистрация | registration (tx `register`) |
| профиль сети | network profile |
| поставщик | provider |
| покупной вход / вход по приглашению | purchased entry / entry by an invitation key |
| единица | unit |
| окно накопления | accrual window |
| якорь слота | slot anchor |
| «отравленный» claim | "poisoned" claim |
| генератор псевдослучайных чисел (ГПЧ) | pseudorandom number generator (PRNG) |
| сид прогона | run seed |
| прогон | run |
| сетка чувствительности | sensitivity grid |
| мост (Kotlin ↔ Go) | bridge |
| снимок состояния | state snapshot (state commitment) |
| заголовок чекпоинта | checkpoint header |
| чёрный список фингерпринтов | fingerprint blacklist |
| отпечаток кошелька | wallet fingerprint |
| манифест | manifest |
| обход (атаки) | bypass |
| согласие поручителя | (implicit in) vouching consent — phrased descriptively |
| сток | sink |
| организатор (сети) | organizer |

Numbers and formatting: American English (decimal point, thousands separator from 10,000, en dash in ranges like "7–30 days", "48 h" with a space). Dates: "October 18, 2026".
