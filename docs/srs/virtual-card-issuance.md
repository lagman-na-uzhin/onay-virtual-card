# Спецификация процесса: Эмиссия виртуальных карт (Virtual Card Issuance)

## 1. Обзор (Overview)
Документ описывает логику и техническую реализацию процесса выпуска новой виртуальной карты для клиента. Процесс включает в себя проверку существующих продуктов пользователя.

## 2. Бизнес-правила
* **Условие выпуска**: Виртуальная карта выпускается только при отсутствии у пользователя активной карты выбранного типа.

## 3. Сценарий взаимодействия (Sequence Diagram)

```mermaid
sequenceDiagram
    autonumber
    
    actor User as Клиент
    participant App as Mobile App
    participant GW as API Gateway
    participant CardSrv as Card Service
    participant InvSrv as Inventory System
    participant BalSrv as Wallet Service
    participant DB as Card Database

    Note over User, App: [PRE-CONDITION]: Данные получены при Init.<br/>У юзера нет активных вирт. карт. Кнопка активна.

    User->>App: Нажатие "Открыть виртуальную карту"
    activate App
    
    Note right of App: POST /v1/cards/virtual
    App->>GW: Request Issue
    activate GW
    
    GW->>CardSrv: Forward Request (Auth Context)
    activate CardSrv
    
    rect rgb(240, 240, 240)
        Note over CardSrv, DB: Бизнес-валидация (Double Check)
        CardSrv->>DB: GetActiveCards(userId)
        DB-->>CardSrv: null (no active cards)
    end

    critical Регистрация и Эмиссия
        CardSrv->>InvSrv: RequestNewCardID(type="virtual")
        activate InvSrv
        InvSrv-->>CardSrv: Success (CardID: 900234...)
        deactivate InvSrv
        
        CardSrv->>BalSrv: InitializeWallet(cardId, currency="KZT")
        activate BalSrv
        BalSrv-->>CardSrv: WalletCreated (Balance: 0)
        deactivate BalSrv

        CardSrv->>DB: SaveEntity(userId, cardId, status="ACTIVE")
    end

    CardSrv-->>GW: 201 Created (Card Data Object)
    deactivate CardSrv
    
    GW-->>App: JSON Response (cardId, maskedId, balance)
    deactivate GW

    App-->>User: Success UI (Отображение карты)
    deactivate App
```
## 4. Обработка исключительных ситуаций (Error Handling)
401 Unauthorized	Ошибка авторизации	Токен не передан, просрочен или подделан.
409 Conflict	Карта уже существует	В БД найдена активная карта того же типа для данного userId.
