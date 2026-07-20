# 🛒 Physical Gem Shop

A purchasing system built on **ABP Framework** + **EF Core** that lets users spend Gems on real-world physical products — with inventory tracking, shipping, campaign-based scheduling, and order management.

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

Uses the existing tables, plus two new ones (`GemShopCampaigns`, `GemShopCampaignItems`) that replace the scheduling/limit fields previously stored directly on `GemShopItems`:

| Table | Purpose |
|---|---|
| `GemShopItems` | Product catalog |
| `GemShopCampaigns` | 🆕 A scheduled sales campaign (name, description, active window) |
| `GemShopCampaignItems` | 🆕 Join table — which items belong to a campaign, and each item's per-campaign purchase limit |
| `GemShopOrders` | Orders |
| `GemShopOrderItems` | Line items within an order |
| `GemShopShippingInfos` | Shipping/tracking per order |
| `GemShopInventoryItems` | Current inventory levels |
| `GemShopInventoryTransactions` | Full audit trail of inventory changes |
| `UserShippingInfos` | User's saved addresses |
| `UserGemHistory` | Ledger of gem balance changes |

### Removed from `GemShopItems`

`Quantity`, `StartDate`, `EndDate`, `LimitPerStudent`, and `LimitType` are no longer stored on the item. Inventory now lives entirely in `GemShopInventoryItems`/`GemShopInventoryTransactions`, and scheduling/limits now live in `GemShopCampaigns`/`GemShopCampaignItems`.

### Relationship model

A campaign is **many-to-many** with items: one campaign (e.g. "Tết 2025 Sale") can bundle several `GemShopItems` together under a single active window, and the same item can appear in multiple campaigns over time (never two *overlapping* active campaigns for the same item, enforced at the application layer). `LimitPerStudent` is set **per item, per campaign** on `GemShopCampaignItems`, not on the campaign itself — two items in the same campaign can have different limits.

```sql
-- =============================================================================
-- Part 6: GEMSHOPCAMPAIGNS – campaigns for physical items
-- =============================================================================

IF NOT EXISTS (SELECT * FROM sysobjects WHERE name = 'GemShopCampaigns' AND xtype = 'U')
BEGIN
    CREATE TABLE [dbo].[GemShopCampaigns] (
        [Id]                    INT             IDENTITY(1,1) NOT NULL,
        [Name]                  NVARCHAR(255)   NOT NULL,
        [Description]           NVARCHAR(1000)  NULL,
        [StartDate]             DATETIME2       NOT NULL,
        [EndDate]               DATETIME2       NOT NULL,
        [CreationTime]          DATETIME2       NOT NULL    DEFAULT GETUTCDATE(),
        [LastModificationTime]  DATETIME2       NULL,
        [IsDeleted]             BIT             NOT NULL    DEFAULT 0,
        CONSTRAINT [PK_GemShopCampaigns] PRIMARY KEY ([Id])
    );

    PRINT 'Created table GemShopCampaigns.';
END
ELSE
BEGIN
    PRINT 'Table GemShopCampaigns already exists – skipped.';
END

IF NOT EXISTS (SELECT * FROM sysobjects WHERE name = 'GemShopCampaignItems' AND xtype = 'U')
BEGIN
    CREATE TABLE [dbo].[GemShopCampaignItems] (
        [Id]                    INT             IDENTITY(1,1) NOT NULL,
        [CampaignId]            INT             NOT NULL,
        [GemShopItemId]         INT             NOT NULL,
        [LimitPerStudent]       INT             NOT NULL    DEFAULT 0,
        [SortOrder]             INT             NOT NULL    DEFAULT 0,
        [CreationTime]          DATETIME2       NOT NULL    DEFAULT GETUTCDATE(),
        [LastModificationTime]  DATETIME2       NULL,
        [IsDeleted]             BIT             NOT NULL    DEFAULT 0,
        CONSTRAINT [PK_GemShopCampaignItems] PRIMARY KEY ([Id]),
        CONSTRAINT [FK_GemShopCampaignItems_Campaign] FOREIGN KEY ([CampaignId]) REFERENCES [dbo].[GemShopCampaigns] ([Id]),
        CONSTRAINT [FK_GemShopCampaignItems_GemShopItem] FOREIGN KEY ([GemShopItemId]) REFERENCES [dbo].[GemShopItems] ([Id]),
        CONSTRAINT [UQ_GemShopCampaignItems_CampaignId_GemShopItemId] UNIQUE ([CampaignId], [GemShopItemId])
    );

    PRINT 'Created table GemShopCampaignItems.';
END
ELSE
BEGIN
    PRINT 'Table GemShopCampaignItems already exists – skipped.';
END
```

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
| `CreationTime` / `CreatorId` | — | ABP audit fields |

> `Quantity`, `StartDate`, `EndDate`, `LimitPerStudent`, `LimitType` have been removed — see `GemShopCampaigns` / `GemShopCampaignItems` below.

### `GemShopCampaigns` 🆕

| Property | Type | Notes |
|---|---|---|
| `Id` | `int` (identity) | PK |
| `Name` | `string` | e.g. "Tết 2025 Sale" |
| `Description` | `string?` | |
| `StartDate` | `DateTime2` | Campaign active window start |
| `EndDate` | `DateTime2` | Campaign active window end |
| `CreationTime` / `LastModificationTime` / `IsDeleted` | — | ABP audit fields (soft delete) |

### `GemShopCampaignItems` 🆕

Join table between `GemShopCampaigns` and `GemShopItems`.

| Property | Type | Notes |
|---|---|---|
| `Id` | `int` (identity) | PK |
| `CampaignId` | `int` | FK → `GemShopCampaigns` |
| `GemShopItemId` | `int` | FK → `GemShopItems` |
| `LimitPerStudent` | `int` | Max purchases of **this item**, **within this campaign** |
| `SortOrder` | `int` | Display order within the campaign |
| `CreationTime` / `LastModificationTime` / `IsDeleted` | — | ABP audit fields (soft delete) |

Unique constraint on `(CampaignId, GemShopItemId)` — an item can only appear once per campaign.

> ⚠️ **Type note:** the FKs here are declared `INT`, while `GemShopItems.Id` above is documented as `Guid` elsewhere in this system. Confirm whether `GemShopItems`/`GemShopOrders`/etc. actually use `int` identity keys, or whether `GemShopCampaignItems.GemShopItemId` needs to be `Guid`/`uniqueidentifier` to match. This doc assumes the SQL as given is authoritative and existing tables are `int`-keyed; update if that's not the case.

### `GemShopOrders`

| Property | Type | Notes |
|---|---|---|
| `Id` | `Guid` | PK |
| `OrderNumber` | `string` | Human-readable order reference |
| `UserId` | `Guid` | FK → user |
| `TotalGems` | `int` | Total gems deducted for the order |
| `Status` | `enum GemShopOrderStatus` | e.g. `Pending`, `Processing`, `Completed`, `Cancelled` |
| `EmailConfirmCode` | `string` | Code generated when the order is created and sent in the confirmation email; used to confirm the purchase and move the order from `Pending` to `Processing` |
| `ConfirmedTime` | `DateTime?` | Set once `EmailConfirmCode` is validated |
| `CreationTime` / `CreatorId` | — | ABP audit fields |

### `GemShopOrderItems`

| Property | Type | Notes |
|---|---|---|
| `Id` | `Guid` | PK |
| `OrderId` | `Guid` | FK → `GemShopOrders` |
| `GemShopItemId` | `Guid` | FK → `GemShopItems` |
| `CampaignId` | `int?` | 🆕 FK → `GemShopCampaigns`. Set for Physical items purchased under a campaign; `null` otherwise. Needed to scope purchase-limit counts to the correct campaign |
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
| `OrderId` | `Guid?` | Nullable — FK → `GemShopOrders`. Set when the gem change is tied to a shop order; `null` for non-order sources such as admin grants or achievement rewards |
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
      ├── CampaignResolutionService  ──▶ resolves active GemShopCampaignItem for the item
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
`Resolve Active Campaign → Validate → Deduct Gems → Create Order (with EmailConfirmCode) → Send Confirmation Email → Confirm → Reserve Inventory → Create Inventory Transaction → Create Shipping Info`

### 📅 Campaign Resolution Service 🆕
- Given a `GemShopItemId`, finds the `GemShopCampaignItem` (joined to its `GemShopCampaign`) where the current time falls within the campaign's `StartDate`/`EndDate`.
- If no such row exists, the item is not currently purchasable.
- If more than one exists (an item accidentally in two overlapping active campaigns), this is treated as a data/config error to be prevented at campaign-authoring time, not resolved at purchase time.

### ✅ Purchase Validation Service
Checks three things before allowing a purchase:

| Check | Rule |
|---|---|
| **Active campaign** | Item must belong to a `GemShopCampaignItem` whose campaign's `StartDate <= CurrentTime <= EndDate` |
| **Inventory** | Must be available before reservation |
| **Purchase limit** | Count of the user's completed purchases of this item **within this campaign** must be `< LimitPerStudent` (from `GemShopCampaignItems`) |

> The previous `AllTime` / `TimeRange` distinction (`LimitType`) is not represented in the new schema — limits are always scoped to the active campaign. Flag if item-wide, cross-campaign limits are still needed; that would require either restoring a flag on `GemShopCampaignItems` or a separate lifetime-limit field on `GemShopItems`.

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
- Creates `GemShopOrder` (including a freshly generated `EmailConfirmCode`) + `GemShopOrderItems` (with `CampaignId` set for Physical items)
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

Scheduling and limits for a Physical item are no longer stored on `GemShopItems` — they're defined by whichever `GemShopCampaign` currently includes the item:

| Property | Where it lives now |
|---|---|
| Initial stock | `GemShopInventoryTransactions` (`StockIn`), seeding `GemShopInventoryItems` |
| Active window (`StartDate`/`EndDate`) | `GemShopCampaigns` |
| `LimitPerStudent` | `GemShopCampaignItems` (per item, per campaign) |

An item with no active `GemShopCampaignItem` row is simply not purchasable, regardless of inventory.

---

## Inventory Management

Inventory quantity is derived from transaction history — **admins never edit it directly**. Inventory is tracked per item and is **shared across all campaigns** that item is ever part of.

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
   Resolve active GemShopCampaignItem for the item
         │
         ▼
   Validate (active campaign, purchase limit, inventory)
         │
         ▼
   Select shipping address → Confirm purchase
         │
         ▼
   Deduct Gems → Write UserGemHistory (OrderId set)
         │
         ▼
   Create Order (generate EmailConfirmCode) → Create Order Item (with CampaignId)
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
Reuses `CreateEditGemShopItemModal.tsx`. `Quantity`, `StartDate`, `EndDate`, `LimitPerStudent`, `LimitType` inputs are removed from this modal — initial stock is entered as an Inventory Transaction (`StockIn`) after creation, and scheduling/limits are set up via Campaign Management below.

### 📅 Campaign Management 🆕
New CRUD screens for `GemShopCampaigns` and `GemShopCampaignItems`:

- **Campaign list** — `Name`, `StartDate`, `EndDate`, item count, status (Upcoming / Active / Ended).
- **Campaign detail** — manage the set of items included (`GemShopCampaignItems`): add/remove items, set `LimitPerStudent` and `SortOrder` per item.
- Saving must enforce no overlapping active campaigns for the same `GemShopItemId`.

### 📦 Inventory Management
An Inventory Transaction modal supporting **Stock In** and **Adjustment**, with fields: `Transaction Type`, `Quantity`, `Note`.
Saving always: creates a `GemShopInventoryTransaction` → updates `GemShopInventoryItems` → never edits inventory directly.

### 🧾 Order Management
Full CRUD on `GemShopOrders` with pagination, search, sort, status filter. Order detail view shows order info (including confirmation status derived from `EmailConfirmCode`), purchased items (product, campaign, quantity, gem cost, status), and shipping info. Admins can update **Order Status** and **Shipping Status** independently.

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

Each item card shows: image, name, gem cost, remaining inventory, purchase limit (from active campaign), active time (from active campaign), and availability state:
`Available` · `Out of Stock` · `Purchase Limit Reached` · `Outside Active Period` (no active campaign)

### 💳 Purchase Dialog
1. Resolve the item's active `GemShopCampaignItem`
2. Validate purchase (campaign window, limit, inventory)
3. Load user's saved shipping addresses
4. Select or create an address
5. Confirm purchase
6. Execute server-side purchase flow (deduct gems → write gem history → create order with `EmailConfirmCode` → create order item with `CampaignId` → send confirmation email → confirm → reserve inventory → create inventory transaction → create shipping info)

### 📜 Order History — `/shop/orders`
Search (by item name), time range, sort, status filter, pagination.

List shows: order number, purchase time, total gems, order status, shipping status.
Detail view shows: purchased items (including which campaign, if any), shipping info, tracking number, timeline, current status.

### 📍 Shipping Address — `/profile`
Create / edit / delete addresses, set default. Default address is auto-preselected at checkout.

---

## APIs

**Client APIs**
- `GetShopItems`
- `GetShopItemDetail` (includes resolved active campaign, if any)
- `PurchasePhysicalItem`
- `ConfirmPurchase` (validates `EmailConfirmCode`)
- `GetMyOrders`
- `GetMyOrderDetail`
- `GetMyGemHistory`
- CRUD `UserShippingInfos`

**Admin APIs**
- CRUD `GemShopItems`
- CRUD `GemShopCampaigns`
- CRUD `GemShopCampaignItems`
- CRUD `GemShopOrders`
- CRUD `GemShopShippingInfos`
- CRUD `InventoryTransactions`
- `InventoryAdjustment`
- `GetUserGemHistory`

> Client and Admin APIs live in **separate Application Services**.

---

## Concurrency & Transactions

- The entire purchase — campaign resolution, validation, gem deduction, gem history write, order creation, inventory reservation, inventory transaction, shipping info — runs inside **one transaction**.
- Any failure rolls back the whole operation.

---

## Extensibility

To add a new `GemShopItemType`:

1. Implement a new purchase handler that follows the purchase strategy interface.
2. Register the handler.
3. Configure the resolver.

No existing purchase handlers or application services need to change. Future types like `CouponCode`, `PROSubscription`, and `AICredit` can all reuse this same infrastructure, including the shared `UserGemHistory` ledger (with `OrderId` populated only when the type produces a `GemShopOrder`) and, if useful, the `GemShopCampaigns`/`GemShopCampaignItems` scheduling model.

---

## Design Principles

- SOLID principles
- Strategy Pattern for purchase handling
- Reusable domain services
- Transactional inventory management
- Concurrency-safe reservation
- Clean separation of Admin vs. Client APIs
- ABP Framework + EF Core best practices
