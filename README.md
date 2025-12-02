# ⚖️ NovaFin Worker: Trade Settlement

> **The Clearing House Engine.**
> Asynchronous batching, validation, and committal of financial transactions to the NovaEco Immutable Ledger.

[](https://www.google.com/search?q=https://github.com/novaeco-tech/novafin-worker-trade-settlement/actions)
[](https://opensource.org/licenses/MIT)
[](https://www.google.com/search?q=https://settle.finance.novaeco.tech)

**TradeSettlement** is a headless, event-driven worker belonging to the **[NovaFin](https://www.google.com/search?q=https://finance.novaeco.tech)** enabler.

While **NovaTrade** matches buyers and sellers (the "Front Office"), this worker acts as the "Back Office." It processes the queue of matched trades, verifies funds, locks assets, and atomically commits the transfer to the financial ledger. It creates the "Proof of Payment" required for physical goods to move.

-----

## 🎯 Value Proposition

In a circular economy, value exchange is complex. We trade Euros, Carbon Credits, and Material Rights simultaneously. Real-time synchronous settlement is risky and slow. This worker solves this by:

1.  **Decoupling:** Allowing the Marketplace (`NovaTrade`) to handle high-volume matching without waiting for the Ledger (`NovaFin`) to write to disk.
2.  **Atomicity:** Ensuring "Delivery-vs-Payment" (DvP). Money only moves if the asset moves.
3.  **Gas Efficiency:** Batching micro-transactions (e.g., 100 plastic bottle deposits) into single ledger commits to reduce computational overhead.

-----

## 🏗️ Architecture (The Clearing Flow)

This worker consumes the `queue.finance.settlements` queue managed by NovaFin.

```mermaid
graph LR
    subgraph "Marketplace (NovaTrade)"
        Trade[Matching Engine] -->|1. Trade Matched| MQ[(RabbitMQ)]
    end

    subgraph "The Worker (NovaFin)"
        MQ -->|2. Consume Event| Worker[TradeSettlement Worker]
        Worker -->|3. Verify Balance| DB[(Ledger DB)]
        Worker -->|4. Atomic Commit| DB
    end

    subgraph "Notification"
        Worker -->|5. Callback: 'PAID'| Trade
        Worker -->|6. Notify User| App[Mission Control]
    end
```

### The Settlement Lifecycle

1.  **Ingest:** Reads a `TradeExecutedEvent` (JSON) containing Buyer ID, Seller ID, Amount, Asset Type, and Trade Hash.
2.  **Validation:** Checks if the Buyer has sufficient available balance (not just total balance, but *unlocked* balance).
3.  **Locking:** Temporarily locks the funds to prevent double-spending.
4.  **Execution:** Moves funds from Buyer -\> Seller.
5.  **Receipt:** Generates a cryptographically signed transaction receipt.

-----

## ✨ Key Features

### 1\. Multi-Asset Settlement

The circular economy runs on multiple currencies. This worker handles the conversion logic:

  * **Fiat:** Euros/USD (via Stripe Connect integration).
  * **Tokens:** NovaPoints, Carbon Credits, Biodiversity Offsets.
  * **Rights:** Ownership tokens for "Product-as-a-Service" leases.

### 2\. High-Frequency Batching

To handle thousands of micro-transactions (e.g., automated bottle returns in `NovaPack`), the worker uses a **"Nagle's Algorithm"** approach:

  * It buffers transactions for 500ms or until 100 items are queued.
  * It commits them as a single database transaction block.

### 3\. Idempotency & Safety

Financial systems cannot fail partially.

  * **Idempotency Keys:** Every trade message has a unique UUID. If the worker crashes and restarts, it checks the Ledger to ensure it doesn't process the same payment twice.
  * **Dead Letter Queues (DLQ):** Failed settlements (e.g., "Insufficient Funds") are routed to a specialized queue for manual accountant review.

-----

## 🚀 Getting Started

### Prerequisites

  * Docker Desktop
  * Python 3.11+
  * Access to a RabbitMQ instance (or local mock)

### Installation

1.  **Clone the repo:**
    ```bash
    git clone https://github.com/novaeco-tech/novafin-worker-trade-settlement.git
    cd novafin-worker-trade-settlement
    ```
2.  **Start the Dev Environment:**
    ```bash
    make dev
    ```
      * Starts the worker container.
      * Starts a local RabbitMQ and Postgres instance.
      * **Health Check:** http://localhost:8080/health

### Configuration (`.env`)

```ini
# Queue Connection
RABBITMQ_URI=amqp://guest:guest@rabbitmq:5672/
QUEUE_NAME=queue.finance.settlements

# Database (The Ledger)
DATABASE_URL=postgresql://user:pass@novafin-db:5432/ledger

# Worker Settings
BATCH_SIZE=100
BATCH_TIMEOUT_MS=500
```

-----

## 📂 Repository Structure

```text
novafin-worker-trade-settlement/
├── src/
│   ├── main.py             # Entry point (Consumer Loop)
│   ├── handlers/           # Logic for different asset types (Fiat vs Token)
│   ├── ledger/             # SQL Alchemy models for the Ledger
│   └── utils/              # Idempotency checks and Math (Decimal)
├── tests/                  # Pytest suite
├── Dockerfile              # Production build definition
└── requirements.txt        # Python dependencies
```

-----

## 🧪 Testing

We use strict **Property-Based Testing** (via Hypothesis) to ensure financial accuracy.

  * **Unit Tests:** `make test`
      * Verifies that `1.00 + 2.00` always equals `3.00` (Decimal precision checks).
  * **Load Tests:** `make test-load`
      * Floods the local queue with 10,000 dummy trades to verify batching performance.

-----

## 🤝 Contributing

**Security Notice:** This repo handles financial logic. All PRs require **two** maintainer reviews. Code must adhere to OWASP guidelines for preventing race conditions.

**Maintainers:** `@novaeco-tech/maintainers-enabler-novafin`