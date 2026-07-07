# Project Tylon

## О проекте

**Tylon** — web3 видеоигровой магазин, похожий на Steam, GOG или Epic Game Store. Игры продаются как NFT-ключи и хранятся on-chain на кошельке игрока.

**Блокчейн: TON**

1. Транзакции за доли секунды
2. Интеграция с Telegram (миллионы пользователей)
3. Удобство разработки (Acton + Tolk)

## Архитектура смарт-контрактов

В проекте два смарт-контракта:

### 1. GameCard (TEP-62 NFT Item)

NFT-ключ игры на кошельке игрока. Токен владения. Совместим с любым NFT-маркетплейсом (Getgems и др.).

Контракт имеет два состояния: **uninitialized** (только что создан коллекцией, без владельца) и **initialized** (владелец установлен). Инициализировать может только GameCollection.

**Storage:**

- `index: uint64` — порядковый номер в коллекции
- `collection_address: MsgAddress` — адрес GameCollection этой игры
- `owner_address: MsgAddress` — текущий владелец (`addr_none` если не инициализирован)
- `individual_content: Cell` — содержимое NFT в формате TEP-64 (содержит `game_id` и URL суффикс)

**Сообщения (входящие):**

- `transfer#5fcc3d14` — передача NFT другому владельцу (TEP-62)
- `get_static_data#2fcb26a2` — запрос статических данных (TEP-62)
- Инициализация от коллекции — установка `owner_address` и `individual_content` (только от collection_address)

**Сообщения (исходящие):**

- `ownership_assigned#05138d91` — уведомление нового владельца о получении NFT
- `report_static_data#8b771735` — ответ на `get_static_data` (содержит `index` и `collection_address`)
- `excesses#d53276db` — возврат излишков TON отправителю

**Геттеры (TEP-62):**

- `get_nft_data()` → `(init?, index, collection_address, owner_address, individual_content)`

---

### 2. GameCollection (TEP-62 Collection + TEP-66 Royalty + Payment Split)

NFT-коллекция одной игры. Деплой идёт на каждую игру. Создаёт GameCard при покупке. Поддерживает оплату в TON и USDT (TEP-74 Jetton).

**Storage:**

- `owner_address: MsgAddress` — владелец (платформа)
- `next_item_index: uint64` — следующий индекс для mint
- `collection_content: Cell` — содержимое коллекции (TEP-64 URI)
- `common_content: Cell` — общий префикс URI для айтемов (например, `https://example.com/items/`)
- `nft_item_code: Cell` — код контракта GameCard (необходим для развёртывания новых NFT)
- `platform_ton_wallet: MsgAddress` — TON-кошелёк платформы
- `developer_ton_wallet: MsgAddress` — TON-кошелёк разработчика
- `platform_fee: uint16` — комиссия платформы (например, 3000 = 30%)
- `game_prices: map<uint32, GamePrice>` — цены игр
- `royalty_numerator: uint16` — числитель royalty
- `royalty_denominator: uint16` — знаменатель royalty
- `royalty_destination: MsgAddress` — адрес получения royalty
- `usdt_master: MsgAddress` — адрес USDT Jetton master (для верификации Jetton wallet)

```tolk
struct GamePrice {
    price_ton: coins
    price_usdt: coins
}
```

**Сообщения (входящие):**

- `get_royalty_params#693d3950` — запрос royalty параметров (TEP-66)
- `buy{value}(game_id: uint32)` — покупка игры за TON. Контракт проверяет `value >= price`, делит payment split on-chain, mintит GameCard отправителю
- `transfer_notification#7362d09c` — уведомление от Jetton Wallet коллекции о получении USDT. Контракт проверяет `sender == Jetton wallet коллекции`, делит split, mintит GameCard
- `set_price(game_id, price_ton, price_usdt)` — обновить цену игры (owner-only)
- `mint(player_address, game_id)` — создание GameCard без оплаты (owner-only, для airdrop/giveaway)
- `change_owner(new_owner)` — передача владения коллекцией (owner-only)

**Сообщения (исходящие):**

- `report_royalty_params#a8cb00ad` — ответ на `get_royalty_params`
- Перевод TON платформе и разработчику (при покупке за TON)
- Команда Jetton wallet на перевод USDT платформе и разработчику (при покупке за USDT)
- Деплой GameCard с StateInit

**Геттеры (TEP-62):**

- `get_collection_data()` → `(next_item_index, collection_content, owner_address)`
- `get_nft_address_by_index(index)` → `address`
- `get_nft_content(index, individual_content)` → `full_content`

**Геттеры (TEP-66):**

- `royalty_params()` → `(numerator, denominator, destination)`

**Геттеры цен:**

- `get_game_price(game_id)` → `(price_ton, price_usdt)`

---

## Потоки покупки

### Покупка за TON (on-chain)

1. Игрок отправляет `buy{value=10 TON}(game_id=1)` на GameCollection
2. Контракт проверяет: `value >= game_prices[game_id].price_ton`
3. Контракт отправляет `platform_fee%` от полученных TON → `platform_ton_wallet`
4. Контракт отправляет остаток → `developer_ton_wallet`
5. Контракт развёртывает GameCard с владельцем `sender`
6. Если на любом шаге ошибка — вся транзакция откатывается (bounce)

### Покупка за USDT (on-chain, TEP-74)

1. Игрок отправляет USDT `transfer` на Jetton Wallet коллекции с `forward_payload`, содержащим `game_id`
2. Jetton Wallet шлёт `transfer_notification#7362d09c(amount, sender, forward_payload)` на GameCollection
3. **Контракт проверяет:** `in.senderAddress == Jetton wallet коллекции` (защита от фейковых уведомлений)
4. Контракт проверяет: `amount >= game_prices[game_id].price_usdt`
5. Контракт шлёт команду своему Jetton wallet: перевести `platform_fee%` USDT → Jetton wallet платформы
6. Контракт шлёт команду своему Jetton wallet: перевести остаток → Jetton wallet разработчика
7. Контракт развёртывает GameCard с владельцем `sender`

### Airdrop / Giveaway

1. Платформа (owner) вызывает `mint(player_address, game_id)`
2. Контракт проверяет `sender == owner_address`
3. Развёртывает GameCard без списания средств

---

## Payment split

При каждой продаже контракт автоматически разделяет средства в одной транзакции:

- `platform_fee%` → кошелёк платформы
- `(100% - platform_fee%)` → кошелёк разработчика

Разработчик получает доход сразу, без ожидания выплат от платформы.

---

## Royalty (TEP-66)

При вторичной продаже GameCard на маркетплейсе (Getgems и др.) маркетплейс автоматически:

1. Вызывает `royalty_params()` на GameCollection
2. Отправляет `numerator/denominator` % продажной цены разработчику

---

## Безопасность

### Защита от фейкового transfer_notification

Любой может отправить контракту сообщение `transfer_notification#7362d09c` с произвольной суммой. Если не проверять отправителя — злоумышленник получит NFT бесплатно.

**Защита:** контракт хранит `usdt_master` и `jetton_wallet_code`, вычисляет адрес своего Jetton wallet и проверяет что отправитель совпадает:

```tolk
val expectedWallet = calculateJettonWalletAddress(
    myAddress(), usdtMaster, jettonWalletCode
);
assert(in.senderAddress == expectedWallet) throw Errors.FakeJetton;
```

### Прямые переводы TON

| Ситуация | Поведение |
|---|---|
| Пустой body | Игнорируется (баланс копится) |
| Неизвестный op | `throw(0xFFFF)` — транзакция откатывается, TON возвращаются |
| Bounced message | Игнорируется (`flags & 1`) |

### Инициализация GameCard

- GameCard создаётся коллекцией в **uninitialized** состоянии (только `index` + `collection_address`)
- Инициализировать (`owner_address` + `individual_content`) может только **GameCollection**
- Повторная инициализация невозможна — после установки `owner_address` контракт переходит в initialized состояние

### Payment split safety

- Перед отправкой проверяется `rest_amount >= 0`
- Отправка с mode `SEND_MODE_BOUNCE_ON_ACTION_FAIL` — при ошибке вся транзакция откатывается

---

## Совместимость

- **TEP-62** — GameCard работает на любом NFT-маркетплейсе
- **TEP-66** — Royalty автоматически выплачивается при вторичных продажах
- **TEP-64** — Метаданные NFT в стандартном off-chain формате
- **TEP-74** — Оплата USDT через Jetton стандарт

## Роли

| Роль | Ответственность |
|------|----------------|
| **Платформа** | Деплой GameCollection, управление ценами, mint, хранение ключей |
| **Разработчик** | Загружает игру через бэкенд, получает доход on-chain с каждой продажи |
| **Игрок** | Оплачивает (TON/USDT), получает NFT на кошелёк, использует как ключ доступа |

## Бизнес-логика (off-chain)

Каталог игр, описания, скриншоты, скидки, промо — хранятся в API/БД. Это бизнес-логика, привязанная к сервису. Контракты — только технология (NFT, royalty, payment split).

## Аккаунт пользователя

Каждый пользователь — TON-кошелёк. Вход/регистрация через TON Connect.

## Порядок разработки

1. **GameCard** — контракт, тесты, развёртывание
2. **GameCollection (MVP, только TON)** — `buy`, payment split, mint
3. **GameCollection (USDT)** — интеграция с Jetton wallet, `transfer_notification`
4. **GameCollection (Royalty + marketplace)** — TEP-66
