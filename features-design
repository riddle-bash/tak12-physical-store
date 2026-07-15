# 🛒 Physical Gem Shop

A purchasing system built on **ABP Framework** + **EF Core** that lets users spend Gems on real-world physical products — with inventory tracking, shipping, and order management.

> Extends the existing Gem Shop to support a new `Physical` item type, while keeping the door open for future types (`AICredit`, `PROSubscription`, `CouponCode`, etc.)

**Design goals:** SOLID · Extensible · Concurrency-safe · Transactional · Consistent with existing ABP conventions

---

## Table of Contents

1. [Database](#database)
2. [Supported Item Types](#supported-item-types)
3. [Architecture](#architecture)
4. [Core Components](#core-components)
5. [Physical Item Configuration](#physical-item-configuration)
6. [Inventory Management](#inventory-management)
7. [Purchase Flow](#purchase-flow)
8. [Fulfillment Flow](#fulfillment-flow)
9. [Admin Features](#admin-features)
10. [Client Features](#client-features)
11. [APIs](#apis)
12. [Concurrency & Transactions](#concurrency--transactions)
13. [Extensibility](#extensibility)

---

## Database

**No schema changes required.** Uses the existing tables:

| Table | Purpose |
|---|---|
| `GemShopItems` | Product catalog |
| `GemShopOrders` | Orders |
| `GemShopOrderItems` | Line items within an order |
| `GemShopShippingInfos` | Shipping/tracking per order |
| `GemShopInventoryItems` | Current inventory levels |
| `GemShopInventoryTransactions` | Full audit trail of inventory changes |
| `UserShippingInfos` | User's saved addresses |

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
      ├── GemDeductionService
      ├── OrderService
      ├── InventoryService
      └── ShippingService
```

**Rule of thumb:** adding a new `GemShopItemType` = write a new handler. Never modify an existing one.

---

## Core Components

### 🧩 Purchase Handler
Orchestrates the full Physical purchase workflow:
`Validate → Deduct Gems → Create Order → Reserve Inventory → Create Inventory Transaction → Create Shipping Info`

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
- Creates `GemShopOrder` + `GemShopOrderItems`
- Updates order status
- Shared/reusable across future purchase handlers

### 🚚 Shipping Service
- Creates shipping info, updates shipping status, maintains tracking
- **Shipping status is independent of order status**

### 💎 Gem Deduction Service
- Deducts Gems from the user's balance
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
   Deduct Gems
         │
         ▼
   Create Order → Create Order Item
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
Full CRUD on `GemShopOrders` with pagination, search, sort, status filter. Order detail view shows order info, purchased items (product, quantity, gem cost, status), and shipping info. Admins can update **Order Status** and **Shipping Status** independently.

### 🏠 User Shipping Address Management
CRUD on `UserShippingInfos`, grouped by user. List shows user, address count, default address. Supports create/edit/delete/set default.

### 🚚 Shipping Management
CRUD on `GemShopShippingInfos` with pagination, search, sort, status filter. Displays order, user, receiver, phone, address, tracking number, shipping status. Admins can update tracking number, shipping status, and notes.

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
5. Execute server-side purchase flow (deduct gems → create order/item → reserve inventory → create inventory transaction → create shipping info)

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
- `GetMyOrders`
- `GetMyOrderDetail`
- CRUD `UserShippingInfos`

**Admin APIs**
- CRUD `GemShopItems`
- CRUD `GemShopOrders`
- CRUD `GemShopShippingInfos`
- CRUD `InventoryTransactions`
- `InventoryAdjustment`

> Client and Admin APIs live in **separate Application Services**.

---

## Concurrency & Transactions

- Inventory operations are concurrency-safe.
- The entire purchase — validation, gem deduction, order creation, inventory reservation, inventory transaction, shipping info — runs inside **one transaction**.
- Any failure rolls back the whole operation.

---

## Extensibility

To add a new `GemShopItemType`:

1. Implement a new purchase handler that follows the purchase strategy interface.
2. Register the handler.
3. Configure the resolver.

No existing purchase handlers or application services need to change. Future types like `CouponCode`, `PROSubscription`, and `AICredit` can all reuse this same infrastructure.

---

## Design Principles

- SOLID principles
- Strategy Pattern for purchase handling
- Reusable domain services
- Transactional inventory management
- Concurrency-safe reservation
- Clean separation of Admin vs. Client APIs
- ABP Framework + EF Core best practices
