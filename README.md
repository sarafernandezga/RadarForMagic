# Financial Speculation Radar for MTG

Project developed by:
- Sara Fernández García-Morales
- Laura Lopez Lindo
- Adrián Martínez Anchuelo

---

## Table of Contents

- [System Architecture](#system-architecture)
  - [Kafka Topics](#kafka-topics)
- [Node-RED Flow Overview](#node-red-flow-overview)
- [Telegram Bot Integration](#telegram-bot-integration)
  - [Automatic Price Notifications](#1-automatic-price-notifications)
  - [Interactive Card Lookup](#2-interactive-card-lookup-cards)
  - [Configuration](#3-configuration)
- [Infrastructure (`docker-compose.yaml`)](#infrastructure-docker-composeyaml)
  - [Environment Variables](#environment-variables)
- [Quick Command Reference](#quick-command-reference)
  - [Start the Environment](#start-the-environment)
  - [Check Status and Logs](#check-status-and-logs)
  - [Stop and Restart Services](#stop-and-restart-services)
  - [Clean Reset](#clean-reset)
- [Accessing & Running the Demo](#accessing--running-the-demo)

An event-driven distributed system for real-time market monitoring of **Magic: The Gathering (MGT)** cards, price anomaly detection (`>20%`), and live dashboard visualization.

## System Architecture

The project decouples data ingestion from stream processing and visualiztion using **Kafka**, allowing independent `consumer groups`to process events in parallel:

```
```

---

### Kafka Topics

- **`market.prices.mtg`**: Main event stream containing all price updates in euros (`EUR`) for the monitored cards. Partitioned using `card_id` as the message key (`msg.key`) to guarantee chronological ordering per card.

- **`market.anomalies.mtg`**: Filtered stream containing only events where a card experiences a price surge or drop exceeding **20%**, including previous/current prices, percentage change, and the card's image URL.

---

## Node-RED Flow Overview

The main Node-RED flow is organized into four functional blocks:

1. **Data Ingestion & Volatility Simulation (`API request`)**

    - **Initial Scryfall Query:** Sends a `POST`request to `https://api.scryfall.com/cards/collection`with the required headers (`User-Agent`, `Accept` and `Content-Type`) to fetch real market prices and artwork for the cards requested, storing them in global context (`global.cards`). The requeste cards has to passed as the `POST` payload.

    - **Market Simulator:** Since Scryfall updates real market prices every 24 hours, this module simulates intraday fluctuations (normal variations of `5 %` and occasional spikes of `30 %`) and publishes them to `market.prices.mtg`.

2. **Anomaly Detection (`Group: mtg-anomaly-detector`)**

    - Consumes price events in real time from `market.prices.mtg`.

    - Tracks the last known price of each card in memory and computes the percentage change. Whenever a variation exceeds the `20 %`threshold, it publishes a structured alert to `market.anomalies.mtg`.

3. **Price History & Visual Dashboard (`Group: mtg-price-history`)**

    - Consumes events from `market.prices.mtg`and maintains a sliding window of the last 50 price points per card.

    - Powers four **FlowFuse Dashboard 2.0** widgets:
        - **Top Rises (`ui-table`):** Dynamic leaderboard displaying the top 10 cards with the highest cumulative percentage gain in the sesion.

        - **Card Selection (`ui-dropdown`):** Interactive dropdown listing all tracked cards alphabetically.

        - **Card Evolution (`ui-chart`):** Time-series line chart showing the price evolution of the selected card.

        - **Card Image (`ui-templete`):** Real-time rendering of the selected card's official Scryfall artwork.

4. **Alert Manager (`Group: mtg-trend-analyzer`)**

    - Subscribed exclusively to `market.anomalies.mtg`to dispatch automated notifications via Telegram with the anomaly details and card image.


## Telegram Bot Integration

RadarForMagic integrates a Telegram bot that allows users to receive automatic price alerts and interactively check information about the cards in their collection.

### Automatic Price Notifications

The `Alert Manager` (`mtg-trend-analyzer`) consumes anomaly events from the Kafka topic `market.anomalies.mtg`.

When a price variation exceeding the 20% threshold is detected, the flow builds and sends a Telegram notification containing:

- **Price movement indicator:** 🔴 Red for price increases and 🟢 Green for price decreases.
- **Card name:** Name of the affected card.
- **Current price:** Latest simulated price in euros (EUR).
- **Percentage change:** Percentage of the price increase or decrease, accompanied by an upward or downward arrow.

Notifications are sent through the Telegram Bot API using an HTTP POST request to the `sendMessage` endpoint.

### Interactive Card Lookup (`/cards`)

The bot also provides an interactive catalogue that allows users to check the latest information about any card loaded into the system.

The interaction follows these steps:

1. **User command:** The user sends `/cards` to the Telegram bot.
2. **Message reception:** The Telegram Receiver node captures the incoming message and forwards it to the `show cards catalog` function.
3. **Catalogue generation:** The function retrieves the monitored cards from Node-RED's global context (`global.cards`) and creates an inline keyboard with one button per card. Each button displays the card name and set, and includes the card's unique identifier (`card_id`) as its `callback_data`.
4. **Catalogue delivery:** Node-RED sends an HTTP POST request to the Telegram Bot API's `sendMessage` endpoint, including the inline keyboard. Telegram displays the selectable card catalogue directly in the chat.
5. **Card selection:** When the user presses a button, Telegram generates a `callback_query` event containing the selected card's identifier. The `Button Event` node captures this event and forwards it to the `show card price and info` function.
6. **Card lookup:** The function extracts the `card_id` from the callback data and searches for the corresponding card in `global.cards`.
7. **Information delivery:** The bot retrieves the selected card's latest price, image, set and rarity, and sends the information back to the user through the Telegram Bot API. It uses the `sendPhoto` endpoint when an image is available, or `sendMessage` otherwise.

This implementation uses Telegram inline keyboards and callback queries to provide an interactive experience without requiring users to manually type card names or identifiers. Card information is retrieved from Node-RED's global context, rather than making a new Scryfall API request for every selection.

### Configuration

The Telegram integration requires two environment variables:

- **`TELEGRAM_TOKEN`**: Authentication token of the Telegram bot.
- **`TELEGRAM_CHAT_ID`**: Destination chat identifier used for automatic notifications.

These variables are stored in the `.env` file and accessed from Node-RED using `env.get()`. Sensitive credentials are not stored directly in the exported flows or committed to the repository.


---

## Infraestructure (`docker-compose.yaml`)

The entire stack is containerized using **Docker Compose**, allowing to keep the environment exclusively on the container.

- **`kafka`(`apache/kafka:latest`):** Runs apache Kafka in **KRaft** mode. Configures two listeners:

    - `kafka:19092`: Internal Docker network communication (used by Node-RED).

    - `localhost:9092`: External access from the host machine.

- **`nodered`(`nodered/node-red:latest`):** Runs the Node-RED flow engine on port `1880`with persistent storage via the `node_red_data` volume.

    - *Installed palette modules:* `node-red-contrib-kafka-suite` and `@flowfuse/node-red-dashboard`.

---

## Use of environment variables

We have declared as environment variables `TELEGRAM_TOKEN` and `TELEGRAM_CHAT_ID` so that they are not defined in the node-RED flow, and by fault public in the repository. These variables are declared in a `.env` file in the same directory as the `docker-comose.yaml`. In the repository there is an example of how to define them (`.env.example`). We access those variables in node-RED with: `env.get("TELEGRAM_VARIABLE");`.


---

## Quick Command Reference

Prepare a project directory containing `docker-comopes.yaml` before running any of the following commands. Docker Desktop must be running before executing the commands.

### Start the environment

```bash
# Start Kafka and Node-RED in detached mode (background) for the first time or after modifying the docker-compose.yaml
docker compose up -d
```

### Check status and logs

```bash
# Verify that both containers are running (Up)
docker compose ps

# Follow live logs for all services
docker compose logs -f

# Follow logs for a specific service
docker compose logs -f <service_name>
```

### Stop and restart services

```bash
# Stop containers while preserving all flows and stored data
docker compose stop

# Start existing stopped containers again
docker compose start

# Stop and remove containers (keeps data volumes intact)
docker compose down
```

### Clean reset (if Kafka throws a cluster erroror you want a fresh start)

```bash
# Removes containers and clears stores Kafka messages/volumes
docker compose down -v
docker compose up -d
```

---

## Accessing & Running the Demo

1. **Node-RED Editor:** <http://localhost:1880>
2. **Visual Dashboard:** <http://localhost:1880/dashboard>