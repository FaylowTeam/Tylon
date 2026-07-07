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

**Storage:**

- `index: uint256` — порядковый номер в коллекции
- `collection_address: MsgAddress` — адрес GameCollection этой игры
- `owner: MsgAddress` — текущий владелец (игрок)
- `game_id: uint32` — идентификатор игры

**Обязательные сообщения (TEP-62):**

- `transfer#5fcc3d14` — передача NFT другому владельцу

**Обязательные геттеры (TEP-62):**

- `get_nft_data()` → `(init?, index, collection_address, owner_address, individual_content)`

### 2. GameCollection (TEP-62 Collection + TEP-66 Royalty + Payment Split)

NFT-коллекция одной игры. Деплой идёт на каждую игру. Создаёт GameCard при покупке. Поддерживает оплату в TON и USDT (TEP-74 Jetton).

**Storage:**

- `owner: MsgAddress` — владелец (платформа)
- `platform_ton_wallet: MsgAddress` — TON-кошелёк платформы
- `platform_usdt_wallet: MsgAddress` — USDT-кошелёк платформы (Jetton)
- `developer_ton_wallet: MsgAddress` — TON-кошелёк разработчика
- `developer_usdt_wallet: MsgAddress` — USDT-кошелёк разработчика (Jetton)
- `platform_fee: uint16` — комиссия платформы (например, 3000 = 30%)
- `game_prices_ton: dict<uint32, uint64>` — цены игр в nanoTON
- `game_prices_usdt: dict<uint32, uint64>` — цены игр в nanoUSDT
- `next_item_index: uint256` — следующий индекс для mint
- `game_content_uri: ^Cell` — URI метаданных игры (TEP-64)
- `royalty_numerator: uint16` — числитель royalty
- `royalty_denominator: uint16` — знаменатель royalty
- `royalty_destination: MsgAddress` — адрес получения royalty

**Обязательные сообщения:**

- `buy{value}(game_id)` — покупка игры за TON. Контракт проверяет цену, делит payment split, mintит GameCard
- `transfer_notification` — уведомление от Jetton Wallet о переводе USDT. Контракт проверяет сумму, делит split, mintит GameCard
- `set_price(game_id, price_ton, price_usdt)` — обновить цену игры (owner-only)

**Обязательные геттеры (TEP-62):**

- `get_collection_data()` → `(next_item_index, collection_content, owner_address)`
- `get_nft_address_by_index(index)` → `address`
- `get_nft_content(index, individual_content)` → `full_content`

**Обязательные геттеры (TEP-66):**

- `royalty_params()` → `(numerator, denominator, destination)`

**Геттеры цен:**

- `get_game_price_ton(game_id)` → `uint64`
- `get_game_price_usdt(game_id)` → `uint64`

## Контроль доступа к mint

`mint()` доступен только для `owner` (платформы). Игрок не может mintить напрямую.

Поток покупки за TON:

1. Игрок оплачивает через бэкенд (карта/крипто/off-chain)
2. Бэкенд отправляет `GameCollection.mint(player_address, game_id)`
3. Контракт проверяет `sender == owner` → mintит GameCard

Поток покупки за USDT:

1. Игрок отправляет USDT через Jetton Wallet → `GameCollection.transfer_notification`
2. Контракт проверяет сумму и отправителя
3. Mintит GameCard

## Payment split

При каждой продаже контракт автоматически разделяет средства:

- `platform_fee%` → кошелёк платформы
- `(100% - platform_fee%)` → кошелёк разработчика

Split происходит on-chain в одной транзакции. Разработчик получает доход сразу.

## Royalty (TEP-66)

При вторичной продаже GameCard на маркетплейсе (Getgems и др.) маркетплейс автоматически:

1. Вызывает `royalty_params()` на GameCollection
2. Отправляет `numerator/denominator` % продажной цены разработчику

## Совместимость

- **TEP-62** — GameCard работает на любом NFT-маркетплейсе
- **TEP-66** — Royalty автоматически выплачивается при вторичных продажах
- **TEP-74** — Оплата USDT через Jetton стандарт

## Роли

| Роль | Ответственность |
|------|----------------|
| **Платформа** | Деплой GameCollection, mint NFT, хранение ключей, управление ценами |
| **Разработчик** | Загружает игру через бэкенд, получает доход on-chain с каждой продажи |
| **Игрок** | Оплачивает (TON/USDT), получает NFT на кошелёк, использует как ключ доступа |

## Бизнес-логика (off-chain)

Каталог игр, описания, скриншоты, скидки, промо — хранятся в API/БД. Это бизнес-логика, привязанная к сервису. Контракты — только технология (NFT, royalty, payment split).

## Аккаунт пользователя

Каждый пользователь — TON-кошелёк. Вход/регистрация через TON Connect.

## Порядок разработки

1. **GameCard** — фундамент, TEP-62
2. **GameCollection** — зависит от GameCard, TEP-62 + TEP-66 + payment split + USDT
