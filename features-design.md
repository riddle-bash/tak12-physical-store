# 🛒 Physical Gem Shop

A purchasing system built on **ABP Framework** + **EF Core** that lets users spend Gems on real-world physical products — with inventory tracking, shipping, and order management.

> Extends the existing Gem Shop to support a new `Physical` item type, while keeping the door open for future types (`AICredit`, `PROSubscription`, `CouponCode`, etc.)

**Design goals:** SOLID · Extensible · Concurrency-safe · Transactional · Consistent with existing ABP conventions

---

## Table of Contents

1. [Database](#database)
2. [Table Properties](#table-properties)
3. [Supported Item Types](#supported-item-types)
4. [Architecture](#architecture)
5. [Core Components](#core-components)
6. [Physical Item Configuration](#physical-item-configuration)
7. [Inventory Management](#inventory-management)
8. [Purchase Flow](#purchase-flow)
9. [Fulfillment Flow](#fulfillment-flow)
10. [Admin Features](#admin-features)
11. [Client Features](#client-features)
12. [APIs](#apis)
13. [Concurrency & Transactions](#concurrency--transactions)
14. [Extensibility](#extensibility)

---

## Database

**No schema changes required** beyond the two additive fields called out below (`GemShopOrders.EmailConfirmCode` and `UserGemHistory.OrderId`). Uses the existing tables:

| Table | Purpose |
|---|---|
| `GemShopItems` | Product catalog |
| `GemShopOrders` | Orders |
| `GemShopOrderItems` | Line items within an order |
| `GemShopShippingInfos` | Shipping/tracking per order |
| `GemShopInventoryItems` | Current inventory levels |
| `GemShopInventoryTransactions` | Full audit trail of inventory changes |
| `UserShippingInfos` | User's saved addresses |
| `UserGemHistory` | Ledger of gem balance changes |

---

## Table Properties

### `GemShopItems`

| Property | Type | Notes |
|---|---|---|
| `Id` | `Guid` | PK |
| `Name` | `string` | Display name |
| `Description` | `string` | |
| `ImageUrl` | `string` | |
| `Type` | `enum GemShopItemType` | `Physical`, `AICredit`, `PROSubscription`, `CouponCode`, virtual types |
| `GemCost` | `int` | |
| `IsActive` | `bool` | |
| `Quantity` | `int?` | **Physical only** — used at creation to seed `GemShopInventoryItems`; not updated afterward |
| `StartDate` | `DateTime?` | **Physical only** |
| `EndDate` | `DateTime?` | **Physical only** |
| `LimitPerStudent` | `int?` | **Physical only** |
| `LimitType` | `enum LimitType` | **Physical only** — `0 = AllTime`, `1 = TimeRange` |
| `CreationTime` / `CreatorId` | — | ABP audit fields |

### `GemShopOrders`

| Property | Type | Notes |
|---|---|---|
| `Id` | `Guid` | PK |
| `OrderNumber` | `string` | Human-readable order reference |
| `UserId` | `Guid` | FK → user |
| `TotalGems` | `int` | Total gems deducted for the order |
| `Status` | `enum GemShopOrderStatus` | e.g. `Pending`, `Processing`, `Completed`, `Cancelled` |
| `EmailConfirmCode` | `string` | 🆕 Code generated when the order is created and sent in the confirmation email; used to confirm the purchase and move the order from `Pending` to `Processing` |
| `ConfirmedTime` | `DateTime?` | Set once `EmailConfirmCode` is validated |
| `CreationTime` / `CreatorId` | — | ABP audit fields |

### `GemShopOrderItems`

| Property | Type | Notes |
|---|---|---|
| `Id` | `Guid` | PK |
| `OrderId` | `Guid` | FK → `GemShopOrders` |
| `GemShopItemId` | `Guid` | FK → `GemShopItems` |
| `Quantity` | `int` | |
| `GemCost` | `int` | Snapshot of cost at purchase time |
| `Status` | `enum GemShopOrderItemStatus` | |

### `GemShopShippingInfos`

| Property | Type | Notes |
|---|---|---|
| `Id` | `Guid` | PK |
| `OrderId` | `Guid` | FK → `GemShopOrders` |
| `UserId` | `Guid` | FK → user |
| `ReceiverName` | `string` | |
| `Phone` | `string` | |
| `Address` | `string` | |
| `TrackingNumber` | `string?` | |
| `ShippingStatus` | `enum GemShopShippingStatus` | Independent of `GemShopOrders.Status` |
| `Notes` | `string?` | |

### `GemShopInventoryItems`

| Property | Type | Notes |
|---|---|---|
| `Id` | `Guid` | PK |
| `GemShopItemId` | `Guid` | FK → `GemShopItems`, unique |
| `QuantityOnHand` | `int` | Derived/maintained from `GemShopInventoryTransactions` |
| `QuantityReserved` | `int` | Sum of open `Reserved` transactions |
| `QuantityAvailable` | `int` | `QuantityOnHand - QuantityReserved` |
| `RowVersion` | `byte[]` | Concurrency token for safe reservation |

### `GemShopInventoryTransactions`

| Property | Type | Notes |
|---|---|---|
| `Id` | `Guid` | PK |
| `GemShopItemId` | `Guid` | FK → `GemShopItems` |
| `TransactionType` | `enum InventoryTransactionType` | `0 = StockIn`, `1 = StockOut`, `2 = Reserved`, `3 = Released`, `4 = Adjustment` |
| `Quantity` | `int` | Signed or paired with type, per convention |
| `Note` | `string?` | Required for `Adjustment` |
| `ReferenceOrderId` | `Guid?` | Set for `Reserved` / `Released` / `StockOut` transactions tied to an order |
| `CreationTime` / `CreatorId` | — | ABP audit fields |

### `UserShippingInfos`

| Property | Type | Notes |
|---|---|---|
| `Id` | `Guid` | PK |
| `UserId` | `Guid` | FK → user |
| `ReceiverName` | `string` | |
| `Phone` | `string` | |
| `Address` | `string` | |
| `IsDefault` | `bool` | One default per user |

### `UserGemHistory`

| Property | Type | Notes |
|---|---|---|
| `Id` | `Guid` | PK |
| `UserId` | `Guid` | FK → user |
| `Amount` | `int` | Positive for credit, negative for debit (or paired with a `Type` field, per existing convention) |
| `Type` | `enum GemHistoryType` | e.g. `Purchase`, `Refund`, `AdminGrant`, `AchievementReward` |
| `OrderId` | `Guid?` | 🆕 **Nullable** — FK → `GemShopOrders`. Set when the gem change is tied to a shop order (e.g. purchase deduction, cancellation refund); `null` for non-order sources such as admin grants or achievement rewards |
| `BalanceAfter` | `int` | Running balance snapshot |
| `CreationTime` | `DateTime` | |

---

## Supported Item Types

| Type | In this build? |
|---|---|
| **Physical** | ✅ Implemented here |
| AICredit | 🔜 Framework-ready |
| PROSubscription | 🔜 Framework-ready |
| CouponCode | 🔜 Framework-ready |
| Existing virtual items | Already supported |

---

## Architecture

Built with the **Strategy Pattern** — one handler per item type — so new item types never require touching existing purchase logic.

```
Purchase Request
      │
      ▼
Purchase Application Service
      │
      ▼
IPurchaseHandlerResolver
      │
      ▼
PhysicalPurchaseHandler
      │
      ├── PurchaseValidationService
      ├── GemDeductionService  ──▶ writes UserGemHistory (OrderId set)
      ├── OrderService          ──▶ generates EmailConfirmCode
      ├── InventoryService
      └── ShippingService
```

**Rule of thumb:** adding a new `GemShopItemType` = write a new handler. Never modify an existing one.

---

## Core Components

### 🧩 Purchase Handler
Orchestrates the full Physical purchase workflow:
`Validate → Deduct Gems → Create Order (with EmailConfirmCode) → Send Confirmation Email → Confirm → Reserve Inventory → Create Inventory Transaction → Create Shipping Info`

### ✅ Purchase Validation Service
Checks three things before allowing a purchase:

| Check | Rule |
|---|---|
| **Active period** | `StartDate <= CurrentTime <= EndDate` |
| **Inventory** | Must be available before reservation |
| **Purchase limit** | See limit types below |

**Purchase limit types:**
- **AllTime** — counts *all* completed purchases of the item, ever
- **TimeRange** — counts completed purchases only within the item's active period

### 📦 Inventory Service
- Inventory is **never modified directly** — every change flows through `GemShopInventoryTransactions`
- All operations are transactional and concurrency-safe

Supported operations: `Reserve` · `Release` · `Stock In` · `Stock Out` · `Adjustment`

**Inventory transaction types:**

| Value | Type | Meaning |
|---|---|---|
| 0 | StockIn | Inventory received |
| 1 | StockOut | Inventory leaves warehouse (after successful delivery) |
| 2 | Reserved | Held during an in-progress purchase |
| 3 | Released | Returned after a cancellation |
| 4 | Adjustment | Manual correction |

### 🧾 Order Service
- Creates `GemShopOrder` (including a freshly generated `EmailConfirmCode`) + `GemShopOrderItems`
- Updates order status
- Validates `EmailConfirmCode` during the confirm-purchase step
- Shared/reusable across future purchase handlers

### 🚚 Shipping Service
- Creates shipping info, updates shipping status, maintains tracking
- **Shipping status is independent of order status**

### 💎 Gem Deduction Service
- Deducts Gems from the user's balance
- Writes a `UserGemHistory` row with `OrderId` set to the new order
- Runs **before** inventory reservation and order creation

---

## Physical Item Configuration

When `GemShopItem.Type == Physical`, these extra fields apply:

| Property | Description |
|---|---|
| `Quantity` | Initial inventory (set at item creation only) |
| `StartDate` | Purchase window start |
| `EndDate` | Purchase window end |
| `LimitPerStudent` | Max purchases allowed per user |
| `LimitType` | `0 = AllTime`, `1 = TimeRange` |

---

## Inventory Management

Inventory quantity is derived from transaction history — **admins never edit it directly**.

| Event | Transaction Created |
|---|---|
| Receiving stock | `StockIn` |
| Manual correction | `Adjustment` |
| Customer purchase | `Reserved` |
| Successful delivery | `StockOut` |
| Cancelled purchase | `Released` |

---

## Purchase Flow

```
User → Browse Shop → Select Physical Item
         │
         ▼
   Validate (active period, purchase limit, inventory)
         │
         ▼
   Select shipping address → Confirm purchase
         │
         ▼
   Deduct Gems → Write UserGemHistory (OrderId set)
         │
         ▼
   Create Order (generate EmailConfirmCode) → Create Order Item
         │
         ▼
   Send confirmation email (contains EmailConfirmCode)
         │
         ▼
   Confirm purchase (validate EmailConfirmCode) → Order Status = Processing
         │
         ▼
   Reserve Inventory → Create Inventory Transaction (Reserved)
         │
         ▼
   Create Shipping Information
         │
         ▼
   ✅ Success
```

---

## Fulfillment Flow

```
Admin → Prepare Shipment → Update Shipping Status
          │
          ▼
       Ship Package → Deliver Package
          │
          ▼
   Create Inventory Transaction (StockOut)
          │
          ▼
       Complete Order
```

> Shipping status and order status are tracked and updated **independently**.

---

## Admin Features

### 🗂️ Gem Shop Item Management
Reuses `CreateEditGemShopItemModal.tsx`. When Item Type = `Physical`, show: `Quantity` (creation-only), `StartDate`, `EndDate`, `LimitPerStudent`, `LimitType`.

### 📦 Inventory Management
An Inventory Transaction modal supporting **Stock In** and **Adjustment**, with fields: `Transaction Type`, `Quantity`, `Note`.
Saving always: creates a `GemShopInventoryTransaction` → updates `GemShopInventoryItems` → never edits inventory directly.

### 🧾 Order Management
Full CRUD on `GemShopOrders` with pagination, search, sort, status filter. Order detail view shows order info (including confirmation status derived from `EmailConfirmCode`), purchased items (product, quantity, gem cost, status), and shipping info. Admins can update **Order Status** and **Shipping Status** independently.

### 🏠 User Shipping Address Management
CRUD on `UserShippingInfos`, grouped by user. List shows user, address count, default address. Supports create/edit/delete/set default.

### 🚚 Shipping Management
CRUD on `GemShopShippingInfos` with pagination, search, sort, status filter. Displays order, user, receiver, phone, address, tracking number, shipping status. Admins can update tracking number, shipping status, and notes.

### 💎 Gem History
Read-only view of `UserGemHistory` per user, showing `Amount`, `Type`, `BalanceAfter`, and (when present) a link to the related `GemShopOrders.OrderId`. Entries with no `OrderId` represent non-order gem changes (grants, rewards, refunds not tied to a shop order, etc.).

---

## Client Features

### 🛍️ Shop — `/shop`
Search, item-type filter, status filter, sort, pagination.

Each item card shows: image, name, gem cost, remaining inventory, purchase limit, active time, and availability state:
`Available` · `Out of Stock` · `Purchase Limit Reached` · `Outside Active Period`

### 💳 Purchase Dialog
1. Validate purchase
2. Load user's saved shipping addresses
3. Select or create an address
4. Confirm purchase
5. Execute server-side purchase flow (deduct gems → write gem history → create order with `EmailConfirmCode` → create order item → send confirmation email → confirm → reserve inventory → create inventory transaction → create shipping info)

### 📜 Order History — `/shop/orders`
Search (by item name), time range, sort, status filter, pagination.

List shows: order number, purchase time, total gems, order status, shipping status.
Detail view shows: purchased items, shipping info, tracking number, timeline, current status.

### 📍 Shipping Address — `/profile`
Create / edit / delete addresses, set default. Default address is auto-preselected at checkout.

---

## APIs

**Client APIs**
- `GetShopItems`
- `GetShopItemDetail`
- `PurchasePhysicalItem`
- `ConfirmPurchase` (validates `EmailConfirmCode`)
- `GetMyOrders`
- `GetMyOrderDetail`
- `GetMyGemHistory`
- CRUD `UserShippingInfos`

**Admin APIs**
- CRUD `GemShopItems`
- CRUD `GemShopOrders`
- CRUD `GemShopShippingInfos`
- CRUD `InventoryTransactions`
- `InventoryAdjustment`
- `GetUserGemHistory`

> Client and Admin APIs live in **separate Application Services**.

---

## Concurrency & Transactions

- Inventory operations are concurrency-safe (`RowVersion` on `GemShopInventoryItems`).
- The entire purchase — validation, gem deduction, gem history write, order creation, inventory reservation, inventory transaction, shipping info — runs inside **one transaction**.
- Any failure rolls back the whole operation.

---

## Extensibility

To add a new `GemShopItemType`:

1. Implement a new purchase handler that follows the purchase strategy interface.
2. Register the handler.
3. Configure the resolver.

No existing purchase handlers or application services need to change. Future types like `CouponCode`, `PROSubscription`, and `AICredit` can all reuse this same infrastructure, including the shared `UserGemHistory` ledger (with `OrderId` populated only when the type produces a `GemShopOrder`).

---

## Design Principles

- SOLID principles
- Strategy Pattern for purchase handling
- Reusable domain services
- Transactional inventory management
- Concurrency-safe reservation
- Clean separation of Admin vs. Client APIs
- ABP Framework + EF Core best practices
