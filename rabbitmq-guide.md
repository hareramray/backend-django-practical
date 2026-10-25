# RabbitMQ: Complete Step-by-Step Guide

**RabbitMQ** is an open-source **message broker**. Programs (producers) send messages to RabbitMQ, RabbitMQ stores and routes them, and other programs (consumers) receive them. The sender and the receiver never talk to each other directly — they only talk to the broker.

Why put a broker between two programs?

| Problem                                                   | How RabbitMQ solves it                                                          |
|-----------------------------------------------------------|---------------------------------------------------------------------------------|
| A web request has to do slow work (email, PDF, video)     | Put a message on a queue, return the response, let a worker do the work later   |
| The receiving service is down or restarting               | Messages wait safely in the queue until it comes back                          |
| Traffic spikes (1 000 orders in one minute)               | The queue absorbs the spike; workers process at their own speed                 |
| One event must reach many services (order → email, stock, analytics) | One message is copied to many queues by an **exchange**              |
| You want to add more workers to go faster                 | Start more consumers on the same queue — RabbitMQ spreads the messages         |
| Services are written in different languages               | AMQP is a wire protocol; there are clients for Python, Java, Go, Node, .NET…   |
| You must not lose messages                                | Durable queues, persistent messages, acknowledgements, publisher confirms      |

What RabbitMQ gives you:

| Feature                         | What it does                                                                      |
|---------------------------------|-----------------------------------------------------------------------------------|
| **Queues**                      | Ordered buffers that store messages until a consumer takes them                   |
| **Exchanges**                   | Routers: decide which queue(s) a message goes to                                  |
| **Bindings & routing keys**     | Rules that connect exchanges to queues                                            |
| **Acknowledgements**            | A message is only removed after the consumer says "done"                          |
| **Publisher confirms**          | The broker tells the producer "I have safely stored your message"                 |
| **Durability / persistence**    | Queues and messages survive a broker restart                                     |
| **Prefetch (QoS)**              | Limits how many unacknowledged messages a consumer holds — fair work distribution |
| **TTL, length limits, DLX**     | Expire messages, cap queue size, move failed messages to a dead-letter queue      |
| **Queue types**                 | Classic, **Quorum** (replicated, safe), **Streams** (append-only log, replay)     |
| **Management UI + HTTP API**    | Web dashboard on port 15672 to see/create everything                              |
| **CLI tools**                   | `rabbitmqctl`, `rabbitmq-diagnostics`, `rabbitmq-plugins`, `rabbitmq-queues`      |
| **Clustering**                  | Several nodes act as one logical broker                                           |
| **Plugins**                     | Shovel, Federation, MQTT, STOMP, Prometheus, delayed messages…                   |
| **Multiple protocols**          | AMQP 0-9-1 (main), AMQP 1.0, MQTT, STOMP, Stream protocol                        |

This guide goes from zero to production-level topics. Parts 1–4 are plain Python scripts (with the `pika` library) in a practice folder; Part 5 connects RabbitMQ to the `blog` Django project from [django-setup-guide.md](django-setup-guide.md) and [django-celery-guide.md](django-celery-guide.md). Commands are for **Windows** (PowerShell); macOS/Linux differences are noted.

> Versions used: **RabbitMQ 4.x**, **Erlang/OTP 26/27**, **Python 3.10+**, **pika 1.3**, **aio-pika 9**, **Celery 5.4+**.

---

## Table of Contents

- **Part 1 — Concepts**
  - [Step 0: The Big Picture](#step-0-the-big-picture)
  - [Step 1: Every Term Explained](#step-1-every-term-explained)
  - [Step 2: The Four Exchange Types](#step-2-the-four-exchange-types)
- **Part 2 — Install and Administer**
  - [Step 3: Install RabbitMQ](#step-3-install-rabbitmq)
  - [Step 4: The Management UI (Tour)](#step-4-the-management-ui-tour)
  - [Step 5: Users, Virtual Hosts and Permissions](#step-5-users-virtual-hosts-and-permissions)
  - [Step 6: The CLI Tools](#step-6-the-cli-tools)
- **Part 3 — Coding With Python (pika)**
  - [Step 7: Set Up the Practice Project](#step-7-set-up-the-practice-project)
  - [Step 8: Hello World (Default Exchange)](#step-8-hello-world-default-exchange)
  - [Step 9: Work Queues (Acks, Prefetch, Durability)](#step-9-work-queues-acks-prefetch-durability)
  - [Step 10: Publish/Subscribe (Fanout Exchange)](#step-10-publishsubscribe-fanout-exchange)
  - [Step 11: Routing (Direct Exchange)](#step-11-routing-direct-exchange)
  - [Step 12: Topics (Topic Exchange)](#step-12-topics-topic-exchange)
  - [Step 13: Headers Exchange](#step-13-headers-exchange)
  - [Step 14: RPC (Request/Reply)](#step-14-rpc-requestreply)
  - [Step 15: Message Properties](#step-15-message-properties)
- **Part 4 — Reliability and Advanced Features**
  - [Step 16: Consumer Acknowledgements in Depth](#step-16-consumer-acknowledgements-in-depth)
  - [Step 17: Publisher Confirms and Unroutable Messages](#step-17-publisher-confirms-and-unroutable-messages)
  - [Step 18: Queue Arguments (TTL, Length Limits, Expiry)](#step-18-queue-arguments-ttl-length-limits-expiry)
  - [Step 19: Dead Letter Exchanges and Retries](#step-19-dead-letter-exchanges-and-retries)
  - [Step 20: Delayed Messages](#step-20-delayed-messages)
  - [Step 21: Priority Queues and Single Active Consumer](#step-21-priority-queues-and-single-active-consumer)
  - [Step 22: Alternate Exchanges and Exchange-to-Exchange Bindings](#step-22-alternate-exchanges-and-exchange-to-exchange-bindings)
  - [Step 23: Queue Types — Classic, Quorum, Stream](#step-23-queue-types--classic-quorum-stream)
  - [Step 24: Policies](#step-24-policies)
  - [Step 25: Connections, Channels, Heartbeats and Reconnecting](#step-25-connections-channels-heartbeats-and-reconnecting)
  - [Step 26: Threads and Long-Running Work With pika](#step-26-threads-and-long-running-work-with-pika)
  - [Step 27: Async Python With aio-pika](#step-27-async-python-with-aio-pika)
- **Part 5 — Django Integration**
  - [Step 28: Use RabbitMQ as the Celery Broker](#step-28-use-rabbitmq-as-the-celery-broker)
  - [Step 29: Publish Domain Events From Django (pika)](#step-29-publish-domain-events-from-django-pika)
  - [Step 30: A Consumer as a Django Management Command](#step-30-a-consumer-as-a-django-management-command)
- **Part 6 — Operations**
  - [Step 31: Configuration Files and Ports](#step-31-configuration-files-and-ports)
  - [Step 32: Memory, Disk Alarms and Flow Control](#step-32-memory-disk-alarms-and-flow-control)
  - [Step 33: Definitions — Export and Import Your Setup](#step-33-definitions--export-and-import-your-setup)
  - [Step 34: Monitoring (UI, HTTP API, Prometheus)](#step-34-monitoring-ui-http-api-prometheus)
  - [Step 35: Security (Users, TLS)](#step-35-security-users-tls)
  - [Step 36: Clustering](#step-36-clustering)
  - [Step 37: Shovel and Federation](#step-37-shovel-and-federation)
  - [Step 38: Best Practices Checklist](#step-38-best-practices-checklist)
  - [Step 39: Troubleshooting](#step-39-troubleshooting)
  - [Step 40: RabbitMQ vs Redis vs Kafka](#step-40-rabbitmq-vs-redis-vs-kafka)
  - [Step 41: Cheat Sheet](#step-41-cheat-sheet)

---

# Part 1 — Concepts

## Step 0: The Big Picture

RabbitMQ's main protocol is **AMQP 0-9-1** (Advanced Message Queuing Protocol). Its model looks like this:

```
                         ┌───────────────────────────── RabbitMQ broker (node) ─────────────────────────────┐
                         │                         virtual host "blog"                                      │
 ┌────────────┐          │   ┌────────────┐   binding             ┌──────────────┐                          │        ┌────────────┐
 │ PRODUCER   │ publish  │   │            │   routing key =       │   QUEUE      │  deliver                 │  ack   │ CONSUMER   │
 │ (Django    │ ───────▶ │   │  EXCHANGE  │ ──"email"───────────▶ │  "emails"    │ ───────────────────────▶ │ ◀───── │ (worker 1) │
 │  view)     │ message: │   │  "tasks"   │                       └──────────────┘                          │        └────────────┘
 └────────────┘ exchange │   │  (direct)  │   routing key =       ┌──────────────┐                          │        ┌────────────┐
                + routing│   │            │ ──"pdf"─────────────▶ │   QUEUE      │ ───────────────────────▶ │ ◀───── │ CONSUMER   │
                  key    │   └────────────┘                       │  "pdfs"      │                          │        │ (worker 2) │
                + body   │                                        └──────────────┘                          │        └────────────┘
                + props  └──────────────────────────────────────────────────────────────────────────────────┘
```

The life of one message:

1. The **producer** opens a TCP **connection** to RabbitMQ, then opens a **channel** inside it.
2. It **publishes** a message to an **exchange**, giving a **routing key** (a string like `"email"`).
3. The exchange looks at its **bindings** (rules) and its **type**, and decides which **queue(s)** get a copy.
   - No matching queue → the message is **dropped** (or returned / sent to an alternate exchange, if configured).
4. The message sits in the queue (in memory and/or on disk).
5. A **consumer** that subscribed to the queue receives the message (RabbitMQ **pushes** it).
6. The consumer does the work and sends an **ack** (acknowledgement).
7. Only after the ack does RabbitMQ delete the message. If the consumer dies before acking, the message is **re-queued** and given to another consumer.

Key idea: **producers never send directly to a queue. They always send to an exchange.** (Even the "send to queue X" shortcut goes through the hidden *default exchange* — see Step 8.)

---

## Step 1: Every Term Explained

| Term                         | Meaning                                                                                                     |
|------------------------------|-------------------------------------------------------------------------------------------------------------|
| **Broker**                   | The RabbitMQ server software                                                                                |
| **Node**                     | One running RabbitMQ server (an Erlang VM). Named like `rabbit@hostname`                                    |
| **Cluster**                  | Several nodes joined together, sharing users, vhosts, exchanges, queue definitions                         |
| **Producer / Publisher**     | Any program that sends messages                                                                             |
| **Consumer / Subscriber**    | Any program that receives messages                                                                          |
| **Message**                  | Payload (**body**, bytes) + **properties** (content type, headers, delivery mode…) + **routing key**       |
| **Body**                     | Raw bytes. RabbitMQ does not look inside. You choose the format (JSON, text, protobuf…)                     |
| **Connection**               | A long-lived TCP connection between your app and the broker (port 5672). Expensive to open                 |
| **Channel**                  | A lightweight virtual connection *inside* a TCP connection. All work (declare, publish, consume) happens on a channel. One connection can have many channels |
| **Exchange**                 | Receives messages from producers and routes them to queues. Has a **type**: direct, fanout, topic, headers |
| **Queue**                    | Stores messages in order (FIFO) until consumed                                                              |
| **Binding**                  | A link "exchange → queue" (or exchange → exchange), optionally with a **binding key** / arguments           |
| **Routing key**              | A string the producer attaches to a message; exchanges compare it to binding keys                          |
| **Binding key**              | The pattern on a binding (`"email"`, `"blog.*.created"`, `"#"`). Often also loosely called "routing key"   |
| **Virtual host (vhost)**     | A namespace: its own exchanges, queues, bindings, permissions. Like separate databases on one DB server    |
| **Default exchange**         | A nameless (`""`) direct exchange; every queue is automatically bound to it with its own name as key       |
| **Declare**                  | "Create if it doesn't exist; if it exists with the same settings, do nothing". Idempotent                  |
| **Durable**                  | Queue/exchange **definition** survives a broker restart                                                     |
| **Persistent message**       | Message is written to disk (`delivery_mode=2`) — survives restart **if the queue is durable**              |
| **Transient**                | Opposite of durable/persistent — lost on restart                                                            |
| **Exclusive queue**          | Used by only one connection; deleted when that connection closes                                            |
| **Auto-delete queue**        | Deleted when its last consumer unsubscribes                                                                 |
| **Ack**                      | Consumer → broker: "processed, you may delete it"                                                           |
| **Nack / Reject**            | Consumer → broker: "failed" — either requeue it or drop/dead-letter it                                      |
| **Auto-ack**                 | Broker considers a message acked the moment it's sent. Fast but messages can be lost                       |
| **Delivery tag**             | A number identifying a delivery on a channel; used when acking                                              |
| **Redelivered flag**         | `True` if this message was delivered before and not acked                                                   |
| **Prefetch (QoS)**           | Max number of unacked messages the broker will push to a consumer at once                                   |
| **Publisher confirm**        | Broker → producer: "message safely received" (ack) or "I couldn't" (nack)                                   |
| **Mandatory flag**           | Producer asks: "if no queue matches, return the message to me instead of dropping it"                      |
| **TTL**                      | Time-to-live: how long a message (or an unused queue) may live                                              |
| **DLX (Dead Letter Exchange)** | Where rejected / expired / overflowed messages are sent                                                   |
| **Plugin**                   | Optional feature you enable (management UI, shovel, MQTT…)                                                  |
| **Erlang**                   | The language/runtime RabbitMQ is written in. Must be installed (Docker image includes it)                  |
| **Erlang cookie**            | A shared secret file; nodes and CLI tools must have the same cookie to talk to each other                  |
| **Policy**                   | A server-side rule that applies settings (TTL, DLX, max-length…) to queues matching a name pattern         |

### 1.1 Message flow terms: ready vs unacked

In the UI and CLI you see two message counters per queue:

| Counter       | Meaning                                                                    |
|---------------|----------------------------------------------------------------------------|
| **Ready**     | Waiting in the queue, not yet delivered to any consumer                    |
| **Unacked**   | Delivered to a consumer, but the consumer has not acked yet                |
| **Total**     | Ready + Unacked                                                            |

If **Ready** keeps growing → consumers are too slow or missing. If **Unacked** is stuck → a consumer forgot to ack or is hung.

---

## Step 2: The Four Exchange Types

### 2.1 Direct — exact match

The message goes to every queue whose binding key **equals** the routing key.

```
                       binding key "error"     ┌──────────────┐
 routing key "error" ┌──────────────────────▶  │ error_logs   │
 ──────────────────▶ │ DIRECT                  └──────────────┘
                     │ "logs_direct"           ┌──────────────┐
                     └──────────────────────▶  │ all_logs     │  bound with "info", "warning" AND "error"
                       binding keys "info",    └──────────────┘
                       "warning", "error"
```

- Routing key `"error"` → both queues.
- Routing key `"info"` → only `all_logs`.
- Routing key `"debug"` → nobody → dropped.

Use for: task types, log severity, "send this job to the PDF queue".

### 2.2 Fanout — broadcast

Ignores the routing key. Copies the message to **every** bound queue.

```
                     ┌──▶ queue "email_service"
 any routing key ──▶ FANOUT ──▶ queue "stock_service"
                     └──▶ queue "analytics_service"
```

Use for: "an order was placed" → every interested service gets a copy; cache invalidation broadcasts; chat rooms.

### 2.3 Topic — pattern match

Routing keys are words separated by dots: `blog.post.created`, `shop.order.paid`.
Binding keys can use two wildcards:

| Wildcard | Matches                         | Example binding     | Matches                                    | Does not match           |
|----------|---------------------------------|---------------------|--------------------------------------------|--------------------------|
| `*`      | **exactly one** word            | `blog.*.created`    | `blog.post.created`, `blog.comment.created` | `blog.created`, `blog.post.x.created` |
| `#`      | **zero or more** words          | `blog.#`            | `blog`, `blog.post`, `blog.post.created`    | `shop.order.paid`        |
| (none)   | exact                           | `shop.order.paid`   | `shop.order.paid`                           | anything else            |

- Binding key `#` alone = receive everything (acts like fanout).
- Binding key with no wildcards = acts like direct.
- Max routing key length: 255 bytes.

Use for: event buses (`<app>.<entity>.<action>`), logs (`<service>.<severity>`).

### 2.4 Headers — match on message headers

Ignores the routing key. Bindings specify header values and `x-match`:

| `x-match`     | Rule                                                                  |
|---------------|-----------------------------------------------------------------------|
| `all` (default) | All listed headers must match                                       |
| `any`         | At least one header must match                                        |
| `all-with-x` / `any-with-x` | Same, but headers starting with `x-` are also compared  |

Use for: routing on several attributes (`format=pdf AND type=report`). Rare in practice; topic usually suffices.

### 2.5 Built-in exchanges

Every vhost already has these:

| Name                 | Type    | Purpose                                                    |
|----------------------|---------|------------------------------------------------------------|
| `""` (AMQP default)  | direct  | Routes to the queue whose **name** = routing key           |
| `amq.direct`         | direct  | Pre-declared, ready to use                                 |
| `amq.fanout`         | fanout  | Pre-declared                                               |
| `amq.topic`          | topic   | Pre-declared (MQTT/STOMP plugins use it)                   |
| `amq.headers`, `amq.match` | headers | Pre-declared                                         |
| `amq.rabbitmq.trace` | topic   | Used by the Firehose tracer                                |

Names starting with `amq.` are reserved — you cannot declare your own.

### 2.6 Exchange properties

| Property      | Meaning                                                                          |
|---------------|----------------------------------------------------------------------------------|
| `name`        | Unique within the vhost                                                          |
| `type`        | direct / fanout / topic / headers (or plugin types like `x-delayed-message`)     |
| `durable`     | Survives restart                                                                 |
| `auto_delete` | Deleted when the last queue is unbound from it                                   |
| `internal`    | Producers cannot publish to it directly; only other exchanges can (Step 22)      |
| `arguments`   | Extra options, e.g. `alternate-exchange`                                         |

---

# Part 2 — Install and Administer

## Step 3: Install RabbitMQ

Pick **one** option. Docker is easiest and is what the rest of the guide assumes.

### 3.1 Option A — Docker Desktop (recommended)

1. Install **Docker Desktop for Windows** from https://www.docker.com/products/docker-desktop/ and start it (it uses WSL 2).
2. Check it works:

   ```powershell
   docker --version
   ```

3. Run RabbitMQ **with the management UI**:

   ```powershell
   docker run -d `
     --name rabbitmq `
     --hostname my-rabbit `
     -p 5672:5672 `
     -p 15672:15672 `
     -e RABBITMQ_DEFAULT_USER=admin `
     -e RABBITMQ_DEFAULT_PASS=admin123 `
     -v rabbitmq_data:/var/lib/rabbitmq `
     --restart unless-stopped `
     rabbitmq:4-management
   ```

   Every flag explained:

   | Flag                                 | Why                                                                                         |
   |--------------------------------------|---------------------------------------------------------------------------------------------|
   | `-d`                                 | Run in the background (detached)                                                            |
   | `--name rabbitmq`                    | Container name, used in `docker exec rabbitmq ...`                                          |
   | `--hostname my-rabbit`               | **Important.** RabbitMQ stores data in a folder named after the node name `rabbit@<hostname>`. Without a fixed hostname, a new container gets a random hostname and "loses" its data |
   | `-p 5672:5672`                       | AMQP port — your apps connect here                                                          |
   | `-p 15672:15672`                     | Management UI / HTTP API                                                                    |
   | `-e RABBITMQ_DEFAULT_USER/PASS`      | Creates this admin user on **first** start (instead of `guest`). Ignored if data already exists |
   | `-v rabbitmq_data:/var/lib/rabbitmq` | Named volume so queues/messages/users survive container removal                             |
   | `--restart unless-stopped`           | Start automatically with Docker Desktop                                                     |
   | `rabbitmq:4-management`              | Official image, version 4.x, **with** the management plugin enabled. (`rabbitmq:4` has no UI) |

4. Check it's running:

   ```powershell
   docker ps
   docker logs rabbitmq --tail 30
   ```

   Look for a line like: `Server startup complete; 4 plugins started.`

5. Useful container commands:

   ```powershell
   docker stop rabbitmq          # stop
   docker start rabbitmq         # start again
   docker restart rabbitmq       # restart
   docker exec -it rabbitmq bash # shell inside the container
   docker rm -f rabbitmq         # delete container (data stays in the volume)
   docker volume rm rabbitmq_data  # delete data too (careful!)
   ```

6. Throughout this guide, a CLI command like `rabbitmqctl list_queues` is run inside the container like this:

   ```powershell
   docker exec rabbitmq rabbitmqctl list_queues
   ```

#### 3.1.1 docker-compose version

Create `docker-compose.yml` (useful when you also run Postgres/Redis/Django):

```yaml
services:
  rabbitmq:
    image: rabbitmq:4-management
    container_name: rabbitmq
    hostname: my-rabbit
    ports:
      - "5672:5672"     # AMQP
      - "15672:15672"   # Management UI
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin123
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
      # - ./rabbitmq/rabbitmq.conf:/etc/rabbitmq/conf.d/20-custom.conf:ro   # Step 31
      # - ./rabbitmq/enabled_plugins:/etc/rabbitmq/enabled_plugins:ro
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "-q", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

volumes:
  rabbitmq_data:
```

```powershell
docker compose up -d
docker compose logs -f rabbitmq
docker compose down          # stop and remove container (volume kept)
```

The `healthcheck` lets other services wait with `depends_on: rabbitmq: condition: service_healthy`.

### 3.2 Option B — Native Windows install

RabbitMQ needs **Erlang** first. Versions must be compatible — check the table at https://www.rabbitmq.com/docs/which-erlang (RabbitMQ 4.x needs Erlang 26 or 27).

1. **Install Erlang/OTP**: download the Windows 64-bit installer from https://www.erlang.org/downloads and run it **as Administrator**.
2. **Install RabbitMQ**: download `rabbitmq-server-4.x.x.exe` from https://www.rabbitmq.com/docs/install-windows and run it **as Administrator**. It installs a Windows **service** named `RabbitMQ` that starts automatically.
   - Alternative with Chocolatey (installs Erlang too): `choco install rabbitmq`
3. Open the Start menu → **RabbitMQ Command Prompt (sbin dir)**. The CLI tools (`rabbitmqctl.bat`, `rabbitmq-plugins.bat`…) live in
   `C:\Program Files\RabbitMQ Server\rabbitmq_server-4.x.x\sbin`. Add that folder to your `PATH` to use them from any terminal.
4. Enable the management UI:

   ```powershell
   rabbitmq-plugins enable rabbitmq_management
   ```

5. Manage the service:

   ```powershell
   Get-Service RabbitMQ
   Restart-Service RabbitMQ      # (admin PowerShell)
   # or from the sbin folder:
   rabbitmq-service.bat stop
   rabbitmq-service.bat start
   ```

6. Default user is `guest` / `guest`, which **only works from localhost**.
7. Data, logs and config live in `%APPDATA%\RabbitMQ\` (for the service: the account's AppData, often `C:\Windows\System32\config\systemprofile\AppData\Roaming\RabbitMQ`).

**Common Windows problem — Erlang cookie mismatch.** The service uses the cookie at `C:\Windows\System32\config\systemprofile\.erlang.cookie`, while CLI tools use `%USERPROFILE%\.erlang.cookie`. If `rabbitmqctl status` says *"authentication failed"* / *"could not connect"*, copy the service's cookie file over your user one, then retry.

### 3.3 Option C — WSL (Ubuntu)

```bash
sudo apt-get update
sudo apt-get install -y rabbitmq-server     # distro version (may be older)
sudo service rabbitmq-server start
sudo rabbitmq-plugins enable rabbitmq_management
```

For the latest version, use the official Cloudsmith apt repositories described at https://www.rabbitmq.com/docs/install-debian.

### 3.4 macOS / Linux

```bash
# macOS
brew install rabbitmq
brew services start rabbitmq          # CLI tools are in $(brew --prefix)/sbin

# Ubuntu / Debian (see official docs for the latest-version repo)
sudo apt-get install -y rabbitmq-server
sudo systemctl enable --now rabbitmq-server
sudo rabbitmq-plugins enable rabbitmq_management
```

### 3.5 Verify the install (any option)

```powershell
docker exec rabbitmq rabbitmq-diagnostics ping          # → "Ping succeeded"
docker exec rabbitmq rabbitmq-diagnostics status        # versions, memory, listeners, alarms
docker exec rabbitmq rabbitmq-diagnostics server_version
docker exec rabbitmq rabbitmq-diagnostics listeners     # which ports are open
```

Open http://localhost:15672 and log in with `admin` / `admin123` (Docker) or `guest` / `guest` (native).

---

## Step 4: The Management UI (Tour)

http://localhost:15672 — every tab:

| Tab             | What you see / do                                                                                         |
|-----------------|-----------------------------------------------------------------------------------------------------------|
| **Overview**    | Totals (queued messages, message rates), nodes, memory/disk usage, ports, **Export/Import definitions**   |
| **Connections** | Every open TCP connection: client IP, user, vhost, state (`running`, `blocked`, `flow`), channels count. You can **force close** one |
| **Channels**    | Every channel: prefetch, unacked count, publish/deliver rates, confirm mode                               |
| **Exchanges**   | List exchanges; **Add a new exchange**; click one → see bindings, **Publish message** manually            |
| **Queues and Streams** | List queues: type, state, Ready, Unacked, Total, rates. **Add a new queue**. Click one → consumers, bindings, **Get messages** (peek), **Purge**, **Delete**, **Move messages** (with shovel plugin) |
| **Admin**       | Users, Virtual Hosts, Feature Flags, Policies, Limits, Cluster name, (Shovel/Federation status if enabled)|

Try this now (no code):

1. **Queues and Streams → Add a new queue**: Name `ui_test`, Durability `Durable`, click **Add queue**.
2. **Exchanges → (AMQP default)** → *Publish message*: Routing key `ui_test`, Payload `hello from UI`, **Publish message**. You see "Message published."
3. **Queues → ui_test**: Ready = 1.
4. **Get messages** → Ack Mode `Nack message requeue true` (peek without removing) → **Get Message(s)**. You see the payload.
5. Change Ack Mode to `Automatic ack` and get again → message removed (Ready = 0).
6. **Delete** the queue at the bottom of the page.

> "Get messages" in the UI is for debugging only — it's a polling `basic.get`, not how apps should consume.

---

## Step 5: Users, Virtual Hosts and Permissions

Never let apps use the admin account. Create a **vhost** per application/environment and a **user** per app.

### 5.1 Create a vhost and a user (CLI)

```powershell
# 1. Virtual host for the blog project
docker exec rabbitmq rabbitmqctl add_vhost blog

# 2. Application user
docker exec rabbitmq rabbitmqctl add_user blog blogpass

# 3. Give it full rights on the "blog" vhost only
#    set_permissions -p <vhost> <user> <configure> <write> <read>
docker exec rabbitmq rabbitmqctl set_permissions -p blog blog ".*" ".*" ".*"

# 4. (Optional) let it log in to the management UI, read-only monitoring
docker exec rabbitmq rabbitmqctl set_user_tags blog monitoring

# Check
docker exec rabbitmq rabbitmqctl list_vhosts
docker exec rabbitmq rabbitmqctl list_users
docker exec rabbitmq rabbitmqctl list_permissions -p blog
docker exec rabbitmq rabbitmqctl list_user_permissions blog
```

### 5.2 What the three permission regexes mean

Each is a regular expression matched against **resource names** (exchange/queue names):

| Permission    | Allows                                                                       |
|---------------|------------------------------------------------------------------------------|
| **configure** | Declare / delete queues and exchanges whose names match                      |
| **write**     | Publish to matching exchanges; bind matching queues (write to queue side)    |
| **read**      | Consume from matching queues; get messages; bind (read from exchange side)   |

Examples:

| Regex                     | Meaning                                       |
|---------------------------|-----------------------------------------------|
| `".*"`                    | Everything                                    |
| `""` (empty)              | Nothing                                       |
| `"^blog\..*"`             | Only names starting with `blog.`              |
| `"^(emails|pdfs)$"`       | Only `emails` and `pdfs`                      |

A read-only consumer: `set_permissions -p blog reporter "" "" "^reports$"`.

### 5.3 User tags (management UI access level)

| Tag              | Can do in the UI / HTTP API                                                     |
|------------------|----------------------------------------------------------------------------------|
| (none)           | Cannot log in to the UI (can still use AMQP)                                     |
| `management`     | See/manage things in vhosts they have permissions for                           |
| `policymaker`    | `management` + manage policies in their vhosts                                   |
| `monitoring`     | `management` + see all connections/channels/node stats                          |
| `administrator`  | Everything: users, vhosts, permissions, all policies                            |

```powershell
docker exec rabbitmq rabbitmqctl set_user_tags admin administrator
docker exec rabbitmq rabbitmqctl set_user_tags blog            # remove all tags
```

### 5.4 Other user commands

```powershell
docker exec rabbitmq rabbitmqctl change_password blog newStrongPass
docker exec rabbitmq rabbitmqctl authenticate_user blog newStrongPass   # test a password
docker exec rabbitmq rabbitmqctl clear_permissions -p blog blog
docker exec rabbitmq rabbitmqctl delete_user blog
docker exec rabbitmq rabbitmqctl delete_vhost blog      # deletes ALL queues/messages in it!
```

### 5.5 The `guest` user

- Exists by default on native installs (password `guest`).
- By default it can only connect from **localhost** (`loopback_users.guest = true`). The Docker image relaxes this, but you replaced it anyway with `RABBITMQ_DEFAULT_USER`.
- **In production: delete it** — `rabbitmqctl delete_user guest`.

### 5.6 Doing it from the UI

**Admin → Virtual Hosts → Add a new virtual host** → name `blog`.
**Admin → Users → Add a user** → username, password, tags.
Click the user → **Set permission** → choose vhost, keep `.*` `.*` `.*`.

### 5.7 vhost limits (optional)

```powershell
# At most 100 connections and 500 queues in the blog vhost
docker exec rabbitmq rabbitmqctl set_vhost_limits -p blog '{\"max-connections\": 100, \"max-queues\": 500}'
docker exec rabbitmq rabbitmqctl clear_vhost_limits -p blog
```

> **PowerShell + JSON quoting:** Windows PowerShell 5.1 strips plain `"` when passing arguments to programs, so write `\"` inside single quotes as above. In bash/zsh write `'{"max-connections": 100}'`.

---

## Step 6: The CLI Tools

| Tool                     | Purpose                                                                     |
|--------------------------|-----------------------------------------------------------------------------|
| `rabbitmqctl`            | Main admin tool: users, vhosts, permissions, list objects, policies, cluster |
| `rabbitmq-diagnostics`   | Health checks, status, memory, logs, environment                            |
| `rabbitmq-plugins`       | Enable / disable / list plugins                                             |
| `rabbitmq-queues`        | Quorum queue and stream replica management                                  |
| `rabbitmq-upgrade`       | Helpers for rolling upgrades (`drain`, `revive`, `await_online_quorum_plus_one`) |
| `rabbitmqadmin`          | Separate tool that uses the **HTTP API** (declare/publish/get from a script) |

Every tool: `<tool> help` and `<tool> help <command>`.

### 6.1 `rabbitmqctl` — listing things

```powershell
docker exec rabbitmq rabbitmqctl list_queues -p blog name type durable messages_ready messages_unacknowledged consumers
docker exec rabbitmq rabbitmqctl list_exchanges -p blog name type durable auto_delete
docker exec rabbitmq rabbitmqctl list_bindings -p blog source_name destination_name routing_key
docker exec rabbitmq rabbitmqctl list_connections user peer_host peer_port state channels
docker exec rabbitmq rabbitmqctl list_channels connection number prefetch_count messages_unacknowledged
docker exec rabbitmq rabbitmqctl list_consumers -p blog
docker exec rabbitmq rabbitmqctl list_policies -p blog
```

Add `--formatter json` (or `pretty_table`) for different output: `rabbitmqctl list_queues --formatter json`.

### 6.2 `rabbitmqctl` — changing things

```powershell
docker exec rabbitmq rabbitmqctl purge_queue -p blog emails           # delete all READY messages
docker exec rabbitmq rabbitmqctl delete_queue -p blog emails          # delete the queue
docker exec rabbitmq rabbitmqctl close_connection "<connection name>" "reason text"
docker exec rabbitmq rabbitmqctl close_all_connections "maintenance"
docker exec rabbitmq rabbitmqctl stop_app      # stop RabbitMQ application (Erlang VM keeps running)
docker exec rabbitmq rabbitmqctl start_app
docker exec rabbitmq rabbitmqctl reset         # DANGER: wipe this node (needs stop_app first)
docker exec rabbitmq rabbitmqctl enable_feature_flag all   # after an upgrade
docker exec rabbitmq rabbitmqctl list_feature_flags
```

### 6.3 `rabbitmq-diagnostics`

```powershell
docker exec rabbitmq rabbitmq-diagnostics ping
docker exec rabbitmq rabbitmq-diagnostics status
docker exec rabbitmq rabbitmq-diagnostics check_running
docker exec rabbitmq rabbitmq-diagnostics check_local_alarms      # memory/disk alarms?
docker exec rabbitmq rabbitmq-diagnostics check_port_connectivity
docker exec rabbitmq rabbitmq-diagnostics check_virtual_hosts
docker exec rabbitmq rabbitmq-diagnostics memory_breakdown --unit mb
docker exec rabbitmq rabbitmq-diagnostics environment             # effective config
docker exec rabbitmq rabbitmq-diagnostics log_tail -N 50
docker exec rabbitmq rabbitmq-diagnostics log_tail_stream         # follow live
docker exec -it rabbitmq rabbitmq-diagnostics observer            # top-like console view
```

### 6.4 `rabbitmq-plugins`

```powershell
docker exec rabbitmq rabbitmq-plugins list                 # [E*] = explicitly enabled, [e*] = enabled as dependency
docker exec rabbitmq rabbitmq-plugins enable rabbitmq_shovel rabbitmq_shovel_management
docker exec rabbitmq rabbitmq-plugins disable rabbitmq_shovel_management
```

Useful plugins (all ship with RabbitMQ):

| Plugin                         | What                                         |
|--------------------------------|----------------------------------------------|
| `rabbitmq_management`          | Web UI + HTTP API                            |
| `rabbitmq_prometheus`          | `/metrics` endpoint on port 15692            |
| `rabbitmq_shovel` (+ `_management`) | Move messages between queues/brokers   |
| `rabbitmq_federation` (+ `_management`) | Link brokers across data centres     |
| `rabbitmq_stream` (+ `_management`) | Binary stream protocol (port 5552)      |
| `rabbitmq_mqtt`, `rabbitmq_web_mqtt` | MQTT for IoT / browsers                 |
| `rabbitmq_stomp`, `rabbitmq_web_stomp` | STOMP protocol                        |
| `rabbitmq_consistent_hash_exchange` | Exchange type that shards by hash      |
| `rabbitmq_auth_backend_ldap`, `rabbitmq_auth_backend_oauth2` | External auth |

Third-party (download `.ez` file): `rabbitmq_delayed_message_exchange` (Step 20).

### 6.5 `rabbitmqadmin` (HTTP API from the command line)

RabbitMQ 4.x uses **rabbitmqadmin v2**, a standalone binary: download it from https://github.com/rabbitmq/rabbitmqadmin-ng/releases. Examples:

```powershell
rabbitmqadmin --host localhost --username admin --password admin123 show overview
rabbitmqadmin --vhost blog --username admin --password admin123 list queues
rabbitmqadmin --vhost blog --username admin --password admin123 declare queue --name test --type quorum
rabbitmqadmin --vhost blog --username admin --password admin123 publish message --routing-key test --payload "hi"
```

(Run `rabbitmqadmin help` for the exact sub-commands of your version.) You can always use the HTTP API directly with `curl` — see Step 34.

---

# Part 3 — Coding With Python (pika)

## Step 7: Set Up the Practice Project

### 7.1 Folder and virtual environment

```powershell
cd "C:\Users\Hareram ray\web\backend-practical"
mkdir rabbitmq-practice
cd rabbitmq-practice
python -m venv venv
.\venv\Scripts\Activate.ps1          # macOS/Linux: source venv/bin/activate
pip install pika
pip freeze > requirements.txt
```

If PowerShell refuses to run `Activate.ps1`: `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`.

### 7.2 Which Python client?

| Library       | Style                         | Use when                                             |
|---------------|-------------------------------|------------------------------------------------------|
| **pika**      | Blocking (and callback-based) | Scripts, Django, simple workers. Official tutorials use it |
| **aio-pika**  | `asyncio`                     | FastAPI / async apps (Step 27)                        |
| **kombu**     | High-level, used by Celery    | You're already using Celery                          |
| **rstream**   | Stream protocol               | High-throughput RabbitMQ Streams                     |

### 7.3 A shared connection helper

Create `rabbitmq-practice/rmq.py` — every script imports from here:

```python
import os

import pika

# amqp://<user>:<password>@<host>:<port>/<vhost>
# The vhost "/" must be written as %2F, e.g. amqp://guest:guest@localhost:5672/%2F
RABBITMQ_URL = os.environ.get("RABBITMQ_URL", "amqp://blog:blogpass@localhost:5672/blog")


def get_connection():
    params = pika.URLParameters(RABBITMQ_URL)
    params.heartbeat = 60                    # seconds; detect dead connections (Step 25)
    params.blocked_connection_timeout = 300  # give up if broker blocks us for 5 min (Step 32)
    params.client_properties = {"connection_name": "rabbitmq-practice"}  # shows in the UI
    return pika.BlockingConnection(params)
```

### 7.4 The URL format

```
amqp://blog:blogpass@localhost:5672/blog?heartbeat=60&connection_attempts=3&retry_delay=5
│      │    │        │         │    │    └ query options (pika supports: heartbeat, connection_attempts,
│      │    │        │         │    │      retry_delay, socket_timeout, blocked_connection_timeout, ...)
│      │    │        │         │    └ virtual host ("/" → %2F)
│      │    │        │         └ port (5672 plain, 5671 TLS)
│      │    │        └ host
│      │    └ password (URL-encode special chars: @ → %40, / → %2F, : → %3A)
│      └ username
└ scheme: amqp (plain) or amqps (TLS)
```

The same with explicit objects instead of a URL:

```python
credentials = pika.PlainCredentials("blog", "blogpass")
params = pika.ConnectionParameters(
    host="localhost",
    port=5672,
    virtual_host="blog",
    credentials=credentials,
    heartbeat=60,
    blocked_connection_timeout=300,
    connection_attempts=3,
    retry_delay=5,
)
connection = pika.BlockingConnection(params)
```

---

## Step 8: Hello World (Default Exchange)

The simplest possible case: one producer, one queue, one consumer.

```
 send.py ──▶ [ default exchange "" ] ──routing key "hello"──▶ [ queue "hello" ] ──▶ receive.py
```

### 8.1 Producer — `send.py`

```python
from rmq import get_connection

connection = get_connection()
channel = connection.channel()

# Make sure the queue exists. Declaring is idempotent: safe to run every time.
channel.queue_declare(queue="hello")

# exchange="" is the DEFAULT exchange: it routes to the queue named by routing_key.
channel.basic_publish(exchange="", routing_key="hello", body="Hello World!")
print(" [x] Sent 'Hello World!'")

# Closing flushes network buffers — without it the message may never leave.
connection.close()
```

### 8.2 Consumer — `receive.py`

```python
import sys

from rmq import get_connection


def main():
    connection = get_connection()
    channel = connection.channel()

    # Declare again: we don't know whether send.py or receive.py starts first.
    channel.queue_declare(queue="hello")

    def callback(ch, method, properties, body):
        # ch         – the channel
        # method     – delivery info: delivery_tag, redelivered, exchange, routing_key, consumer_tag
        # properties – pika.BasicProperties: content_type, headers, message_id, ...
        # body       – bytes
        print(f" [x] Received {body.decode()}")

    # auto_ack=True: the broker forgets the message as soon as it's sent (OK for this demo only).
    channel.basic_consume(queue="hello", on_message_callback=callback, auto_ack=True)

    print(" [*] Waiting for messages. To exit press CTRL+C")
    channel.start_consuming()   # blocks forever, calling callback for each message


if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        print("Interrupted")
        sys.exit(0)
```

### 8.3 Run it

```powershell
# Terminal 1
python receive.py

# Terminal 2 (run it several times)
python send.py
```

Terminal 1 prints `[x] Received Hello World!` every time.

Experiments:

1. Stop `receive.py`, run `send.py` 3 times, then check: `docker exec rabbitmq rabbitmqctl list_queues -p blog` → `hello 3`. Start `receive.py` → it gets all 3. **Messages wait for consumers.**
2. Look at **Connections**, **Channels** and **Queues → hello → Consumers** in the UI while `receive.py` runs.

### 8.4 What `queue_declare` returns

```python
result = channel.queue_declare(queue="hello")
print(result.method.queue)           # 'hello'
print(result.method.message_count)   # ready messages right now
print(result.method.consumer_count)  # consumers right now

# passive=True: only CHECK the queue exists — raises ChannelClosedByBroker (404 NOT_FOUND) if not.
channel.queue_declare(queue="hello", passive=True)
```

### 8.5 Pulling one message instead of subscribing (`basic_get`)

```python
method, properties, body = channel.basic_get(queue="hello", auto_ack=False)
if method is None:
    print("queue empty")
else:
    print(body)
    channel.basic_ack(method.delivery_tag)
```

`basic_get` polls — one network round-trip per message. Use `basic_consume` for real consumers; `basic_get` is fine for scripts and tests.

---

## Step 9: Work Queues (Acks, Prefetch, Durability)

Goal: spread slow tasks across several workers.

```
                                       ┌──▶ worker.py #1
 new_task.py ──▶ [ queue "task_queue" ]┤
                                       └──▶ worker.py #2
```

### 9.1 Producer — `new_task.py`

```python
import sys

import pika

from rmq import get_connection

connection = get_connection()
channel = connection.channel()

# durable=True: the queue definition survives a broker restart.
channel.queue_declare(queue="task_queue", durable=True)

message = " ".join(sys.argv[1:]) or "Hello World!"
channel.basic_publish(
    exchange="",
    routing_key="task_queue",
    body=message,
    properties=pika.BasicProperties(
        delivery_mode=pika.DeliveryMode.Persistent,   # = 2: write message to disk
    ),
)
print(f" [x] Sent {message}")
connection.close()
```

### 9.2 Worker — `worker.py`

Each `.` in the message = 1 second of fake work.

```python
import time

from rmq import get_connection

connection = get_connection()
channel = connection.channel()
channel.queue_declare(queue="task_queue", durable=True)
print(" [*] Waiting for messages. To exit press CTRL+C")


def callback(ch, method, properties, body):
    text = body.decode()
    print(f" [x] Received {text}")
    time.sleep(text.count("."))
    print(" [x] Done")
    # Manual ack: tell RabbitMQ the message can be deleted. MUST be on the same channel.
    ch.basic_ack(delivery_tag=method.delivery_tag)


# Fair dispatch: don't give this worker a new message until it has acked the previous one.
channel.basic_qos(prefetch_count=1)

# auto_ack defaults to False → manual acknowledgements.
channel.basic_consume(queue="task_queue", on_message_callback=callback)
channel.start_consuming()
```

### 9.3 Run it

```powershell
# Terminal 1 and Terminal 2
python worker.py

# Terminal 3
python new_task.py First message.
python new_task.py Second message..
python new_task.py Third message...
python new_task.py Fourth message....
python new_task.py Fifth message.....
```

The workers share the messages.

### 9.4 Concept 1 — Round-robin dispatch

By default RabbitMQ sends message 1 to worker A, message 2 to worker B, 3 to A, … regardless of how busy each is. Want more speed? **Start more workers.** That's horizontal scaling.

### 9.5 Concept 2 — Message acknowledgement

- With `auto_ack=True`, a message is deleted as soon as it's sent. If the worker crashes mid-task, **the message is lost**.
- With manual ack (`auto_ack=False` + `basic_ack`), if the worker's channel/connection closes before acking, RabbitMQ **re-queues** the message and delivers it to another worker (with `redelivered=True`).

Test it: start two workers, send `python new_task.py long..........`, kill the worker that got it (Ctrl+C) before "Done". The other worker receives it.

**Forgotten ack = memory leak.** If you never ack, messages stay *Unacked* forever (until the consumer disconnects), and RabbitMQ keeps them in memory. Check with:

```powershell
docker exec rabbitmq rabbitmqctl list_queues -p blog name messages_ready messages_unacknowledged
```

**Consumer timeout.** RabbitMQ closes a channel whose consumer hasn't acked a delivery within `consumer_timeout` (default **30 minutes**) with `PRECONDITION_FAILED - delivery acknowledgement timed out`. For tasks longer than that, raise the timeout (Step 31) or redesign the work.

### 9.6 Concept 3 — Durability (survive a restart)

Two things are needed, **both**:

| Setting                                          | What it protects                     |
|--------------------------------------------------|--------------------------------------|
| `queue_declare(..., durable=True)`               | The queue itself                     |
| `BasicProperties(delivery_mode=Persistent)`      | Each message                         |

Test: send 3 messages with no workers running, then `docker restart rabbitmq`, then list queues — the 3 messages are still there.

Notes:

- You **cannot change** the settings of an existing queue. Declaring `hello` with `durable=True` after it was created non-durable fails with `PRECONDITION_FAILED - inequivalent arg 'durable'`. Delete the queue first or use a new name.
- Persistence is not an absolute guarantee: there's a short window between "received" and "written to disk". For a real guarantee, use **publisher confirms** (Step 17) + **quorum queues** (Step 23).

### 9.7 Concept 4 — Prefetch (`basic_qos`)

Without prefetch, RabbitMQ pushes **all** ready messages to consumers immediately — one worker can get 100 slow tasks while another sits idle.

`basic_qos(prefetch_count=N)` = "never have more than N unacked messages on this consumer".

| prefetch_count | Behaviour                                                            |
|----------------|----------------------------------------------------------------------|
| 0              | Unlimited (default) — dangerous for slow consumers                   |
| 1              | Fairest; slowest throughput (one round-trip per message)             |
| 10–50          | Good default for fast tasks                                          |
| 100–300        | High throughput, fast consumers                                      |

Rule of thumb: slow/uneven tasks → small prefetch; fast/uniform tasks → larger prefetch.

`basic_qos(prefetch_count=10, global_qos=True)` applies the limit to the whole channel instead of each consumer (not supported by quorum queues — avoid it).

---

## Step 10: Publish/Subscribe (Fanout Exchange)

Goal: deliver **every** message to **every** consumer (a log broadcaster).

```
                                  ┌──▶ [ amq.gen-JzTY... (exclusive) ] ──▶ receive_logs.py #1
 emit_log.py ──▶ [ "logs" FANOUT ]┤
                                  └──▶ [ amq.gen-Qi4L... (exclusive) ] ──▶ receive_logs.py #2
```

### 10.1 Producer — `emit_log.py`

```python
import sys

from rmq import get_connection

connection = get_connection()
channel = connection.channel()

channel.exchange_declare(exchange="logs", exchange_type="fanout")

message = " ".join(sys.argv[1:]) or "info: Hello World!"
# routing_key is ignored by fanout exchanges, but must be passed — use "".
channel.basic_publish(exchange="logs", routing_key="", body=message)
print(f" [x] Sent {message}")
connection.close()
```

### 10.2 Consumer — `receive_logs.py`

```python
from rmq import get_connection

connection = get_connection()
channel = connection.channel()

channel.exchange_declare(exchange="logs", exchange_type="fanout")

# queue="" → server generates a unique name (amq.gen-...).
# exclusive=True → only this connection can use it, deleted when we disconnect.
result = channel.queue_declare(queue="", exclusive=True)
queue_name = result.method.queue

# Binding: "send messages from exchange 'logs' to my queue"
channel.queue_bind(exchange="logs", queue=queue_name)

print(f" [*] Waiting for logs on {queue_name}. To exit press CTRL+C")


def callback(ch, method, properties, body):
    print(f" [x] {body.decode()}")


channel.basic_consume(queue=queue_name, on_message_callback=callback, auto_ack=True)
channel.start_consuming()
```

### 10.3 Run it

```powershell
python receive_logs.py            # terminal 1
python receive_logs.py            # terminal 2
python emit_log.py "something happened"   # terminal 3
```

Both receivers print the message.

Important points:

- If you publish when **no** queue is bound, the message is **dropped** — fanout only copies to existing queues.
- **Temporary (exclusive) queue** = "I only care about messages while I'm running". For a service that must not miss events while restarting, use a **named, durable** queue instead, e.g. `email_service.order_events`, and bind it once.
- Check bindings: `rabbitmqctl list_bindings -p blog`.

---

## Step 11: Routing (Direct Exchange)

Goal: subscribe to **some** messages — e.g. one consumer saves only `error` logs to disk, another prints everything.

```
                                              "error"                  ┌──────────┐
                                         ┌──────────────────────────▶  │  Q1      │──▶ C1 (errors to disk)
 emit_log_direct.py ─▶ [ "direct_logs" ] ┤                             └──────────┘
   routing key = severity                │ "info","warning","error"   ┌──────────┐
                                         └──────────────────────────▶  │  Q2      │──▶ C2 (print all)
                                                                       └──────────┘
```

### 11.1 Producer — `emit_log_direct.py`

```python
import sys

from rmq import get_connection

connection = get_connection()
channel = connection.channel()
channel.exchange_declare(exchange="direct_logs", exchange_type="direct")

severity = sys.argv[1] if len(sys.argv) > 1 else "info"
message = " ".join(sys.argv[2:]) or "Hello World!"
channel.basic_publish(exchange="direct_logs", routing_key=severity, body=message)
print(f" [x] Sent {severity}:{message}")
connection.close()
```

### 11.2 Consumer — `receive_logs_direct.py`

```python
import sys

from rmq import get_connection

connection = get_connection()
channel = connection.channel()
channel.exchange_declare(exchange="direct_logs", exchange_type="direct")

result = channel.queue_declare(queue="", exclusive=True)
queue_name = result.method.queue

severities = sys.argv[1:]
if not severities:
    sys.stderr.write(f"Usage: {sys.argv[0]} [info] [warning] [error]\n")
    sys.exit(1)

# One binding per severity — a queue may have many bindings.
for severity in severities:
    channel.queue_bind(exchange="direct_logs", queue=queue_name, routing_key=severity)

print(" [*] Waiting for logs. To exit press CTRL+C")


def callback(ch, method, properties, body):
    print(f" [x] {method.routing_key}:{body.decode()}")


channel.basic_consume(queue=queue_name, on_message_callback=callback, auto_ack=True)
channel.start_consuming()
```

### 11.3 Run it

```powershell
python receive_logs_direct.py error                    # terminal 1: only errors
python receive_logs_direct.py info warning error       # terminal 2: everything
python emit_log_direct.py error "Run. Run. Or it will explode."
python emit_log_direct.py info "Just FYI"
```

Notes:

- **Multiple bindings with the same key** on different queues → each queue gets a copy (direct behaves like fanout for that key).
- To remove a binding: `channel.queue_unbind(queue=queue_name, exchange="direct_logs", routing_key="info")`.

---

## Step 12: Topics (Topic Exchange)

Goal: route on several criteria. Routing key format here: `<source>.<severity>`, e.g. `kern.critical`, `blog.info`.

### 12.1 Producer — `emit_log_topic.py`

```python
import sys

from rmq import get_connection

connection = get_connection()
channel = connection.channel()
channel.exchange_declare(exchange="topic_logs", exchange_type="topic")

routing_key = sys.argv[1] if len(sys.argv) > 2 else "anonymous.info"
message = " ".join(sys.argv[2:]) or "Hello World!"
channel.basic_publish(exchange="topic_logs", routing_key=routing_key, body=message)
print(f" [x] Sent {routing_key}:{message}")
connection.close()
```

### 12.2 Consumer — `receive_logs_topic.py`

```python
import sys

from rmq import get_connection

connection = get_connection()
channel = connection.channel()
channel.exchange_declare(exchange="topic_logs", exchange_type="topic")

result = channel.queue_declare(queue="", exclusive=True)
queue_name = result.method.queue

binding_keys = sys.argv[1:]
if not binding_keys:
    sys.stderr.write(f"Usage: {sys.argv[0]} [binding_key]...\n")
    sys.exit(1)

for binding_key in binding_keys:
    channel.queue_bind(exchange="topic_logs", queue=queue_name, routing_key=binding_key)

print(" [*] Waiting for logs. To exit press CTRL+C")


def callback(ch, method, properties, body):
    print(f" [x] {method.routing_key}:{body.decode()}")


channel.basic_consume(queue=queue_name, on_message_callback=callback, auto_ack=True)
channel.start_consuming()
```

### 12.3 Run it

```powershell
python receive_logs_topic.py "#"                       # everything
python receive_logs_topic.py "kern.*"                  # all kernel logs
python receive_logs_topic.py "*.critical"              # all critical logs
python receive_logs_topic.py "kern.*" "*.critical"     # either

python emit_log_topic.py "kern.critical" "A critical kernel error"
python emit_log_topic.py "blog.info" "Post published"
```

Quote `"#"` in PowerShell (`#` starts a comment).

Quiz — binding `*.orange.*` and `*.*.rabbit` and `lazy.#`:

| Routing key              | Goes to                         |
|--------------------------|---------------------------------|
| `quick.orange.rabbit`    | both first and second           |
| `lazy.orange.elephant`   | first and third                 |
| `quick.orange.fox`       | first                           |
| `lazy.brown.fox`         | third                           |
| `lazy.pink.rabbit`       | second and third (**once** — a queue gets one copy even if several bindings match) |
| `quick.brown.fox`        | nobody → dropped                |
| `orange`                 | nobody                          |
| `lazy.orange.new.rabbit` | third                           |

---

## Step 13: Headers Exchange

### 13.1 Consumer — `receive_headers.py`

```python
from rmq import get_connection

connection = get_connection()
channel = connection.channel()
channel.exchange_declare(exchange="reports", exchange_type="headers")

result = channel.queue_declare(queue="", exclusive=True)
queue_name = result.method.queue

# Receive only messages whose headers have format=pdf AND type=report.
channel.queue_bind(
    exchange="reports",
    queue=queue_name,
    routing_key="",          # ignored
    arguments={"x-match": "all", "format": "pdf", "type": "report"},
)


def callback(ch, method, properties, body):
    print(f" [x] headers={properties.headers} body={body.decode()}")


channel.basic_consume(queue=queue_name, on_message_callback=callback, auto_ack=True)
print(" [*] Waiting for pdf reports")
channel.start_consuming()
```

### 13.2 Producer — `emit_headers.py`

```python
import pika

from rmq import get_connection

connection = get_connection()
channel = connection.channel()
channel.exchange_declare(exchange="reports", exchange_type="headers")

for headers in (
    {"format": "pdf", "type": "report"},    # matches
    {"format": "zip", "type": "report"},    # doesn't
    {"format": "pdf", "type": "invoice"},   # doesn't
):
    channel.basic_publish(
        exchange="reports",
        routing_key="",
        body=f"file with {headers}",
        properties=pika.BasicProperties(headers=headers),
    )
    print(f" [x] Sent {headers}")
connection.close()
```

Change `"x-match": "all"` to `"any"` and all three are delivered.

---

## Step 14: RPC (Request/Reply)

Goal: call a function on a remote worker and **wait for the result**.

```
            request (reply_to=amq.gen-X, correlation_id=abc)
 client ──▶ [ queue "rpc_queue" ] ──▶ server
   ▲                                    │
   └──── [ queue amq.gen-X ] ◀──────────┘  response (correlation_id=abc)
```

Two message properties make it work:

| Property         | Purpose                                                                    |
|------------------|----------------------------------------------------------------------------|
| `reply_to`       | Name of the queue the server should send the answer to                     |
| `correlation_id` | Unique id per request, so the client can match answers to requests          |

### 14.1 Server — `rpc_server.py`

```python
import pika

from rmq import get_connection

connection = get_connection()
channel = connection.channel()
channel.queue_declare(queue="rpc_queue")


def fib(n):
    if n < 2:
        return n
    return fib(n - 1) + fib(n - 2)


def on_request(ch, method, props, body):
    n = int(body)
    print(f" [.] fib({n})")
    response = fib(n)

    ch.basic_publish(
        exchange="",
        routing_key=props.reply_to,                                  # answer to the caller's queue
        properties=pika.BasicProperties(correlation_id=props.correlation_id),
        body=str(response),
    )
    ch.basic_ack(delivery_tag=method.delivery_tag)


channel.basic_qos(prefetch_count=1)    # spread load across several servers
channel.basic_consume(queue="rpc_queue", on_message_callback=on_request)
print(" [x] Awaiting RPC requests")
channel.start_consuming()
```

### 14.2 Client — `rpc_client.py`

```python
import uuid

import pika

from rmq import get_connection


class FibonacciRpcClient:
    def __init__(self):
        self.connection = get_connection()
        self.channel = self.connection.channel()

        # One private reply queue per client.
        result = self.channel.queue_declare(queue="", exclusive=True)
        self.callback_queue = result.method.queue

        self.channel.basic_consume(
            queue=self.callback_queue,
            on_message_callback=self.on_response,
            auto_ack=True,
        )
        self.response = None
        self.corr_id = None

    def on_response(self, ch, method, props, body):
        if self.corr_id == props.correlation_id:    # ignore late answers to old requests
            self.response = body

    def call(self, n, timeout=10):
        self.response = None
        self.corr_id = str(uuid.uuid4())
        self.channel.basic_publish(
            exchange="",
            routing_key="rpc_queue",
            properties=pika.BasicProperties(
                reply_to=self.callback_queue,
                correlation_id=self.corr_id,
                expiration=str(timeout * 1000),     # drop the request if nobody picks it up in time
            ),
            body=str(n),
        )
        # Pump network events until our answer arrives (or time runs out).
        self.connection.process_data_events(time_limit=timeout)
        if self.response is None:
            raise TimeoutError("RPC call timed out")
        return int(self.response)


client = FibonacciRpcClient()
print(" [x] Requesting fib(30)")
print(f" [.] Got {client.call(30)}")
```

Run `python rpc_server.py` then `python rpc_client.py`.

### 14.3 Direct reply-to (no reply queue needed)

RabbitMQ has a pseudo-queue `amq.rabbitmq.reply-to`. Consume from it with `auto_ack=True` and set `reply_to="amq.rabbitmq.reply-to"`; the broker routes the response straight back to your channel. Faster — no queue is created per client:

```python
self.channel.basic_consume(queue="amq.rabbitmq.reply-to", on_message_callback=self.on_response, auto_ack=True)
# ... publish with reply_to="amq.rabbitmq.reply-to" on the SAME channel
```

### 14.4 When (not) to use RPC

- RPC over a queue hides network calls behind function calls; it adds latency and you must handle timeouts and a server that is down.
- For "do it later" work, **don't wait** — fire-and-forget (Step 9). For a synchronous answer between services, plain HTTP/gRPC is often simpler. RPC over RabbitMQ shines when you want load-balancing across many workers without a load balancer.

---

## Step 15: Message Properties

Every message can carry these (`pika.BasicProperties(...)`):

| Property           | Type     | Meaning / typical use                                                    |
|--------------------|----------|--------------------------------------------------------------------------|
| `content_type`     | str      | `"application/json"`, `"text/plain"` — tells the consumer how to decode  |
| `content_encoding` | str      | `"utf-8"`, `"gzip"`                                                      |
| `headers`          | dict     | Any custom key/values (`{"x-retry-count": 2, "tenant": "acme"}`)         |
| `delivery_mode`    | int      | `1` transient, `2` persistent (`pika.DeliveryMode.Persistent`)           |
| `priority`         | int      | 0–255, only used by priority queues (Step 21)                            |
| `correlation_id`   | str      | Match a reply to a request (RPC) or link related messages               |
| `reply_to`         | str      | Queue name for the reply (RPC)                                           |
| `expiration`       | str      | Per-message TTL in **milliseconds, as a string**: `"60000"`              |
| `message_id`       | str      | Unique id — use it for **de-duplication / idempotency**                  |
| `timestamp`        | int      | Unix time (seconds) when created                                         |
| `type`             | str      | Message type name: `"order.created"`                                     |
| `user_id`          | str      | If set, broker checks it equals the connection's user (spoof protection) |
| `app_id`           | str      | Name of the producing app                                                |

### 15.1 Sending JSON the right way — `publish_json.py`

```python
import json
import time
import uuid

import pika

from rmq import get_connection

connection = get_connection()
channel = connection.channel()
channel.queue_declare(queue="orders", durable=True)

order = {"order_id": 1001, "user_id": 7, "total": "49.90", "items": [{"sku": "BOOK-1", "qty": 2}]}

channel.basic_publish(
    exchange="",
    routing_key="orders",
    body=json.dumps(order).encode("utf-8"),
    properties=pika.BasicProperties(
        content_type="application/json",
        content_encoding="utf-8",
        delivery_mode=pika.DeliveryMode.Persistent,
        message_id=str(uuid.uuid4()),
        timestamp=int(time.time()),
        type="order.created",
        app_id="shop-web",
        headers={"schema_version": 1},
    ),
)
connection.close()
```

### 15.2 Reading it — `consume_json.py`

```python
import json

from rmq import get_connection

connection = get_connection()
channel = connection.channel()
channel.queue_declare(queue="orders", durable=True)


def callback(ch, method, properties, body):
    if properties.content_type != "application/json":
        print("unexpected content type, rejecting")
        ch.basic_reject(method.delivery_tag, requeue=False)
        return
    order = json.loads(body)
    print(f"id={properties.message_id} type={properties.type} headers={properties.headers}")
    print(f"order: {order}")
    ch.basic_ack(method.delivery_tag)


channel.basic_qos(prefetch_count=10)
channel.basic_consume(queue="orders", on_message_callback=callback)
channel.start_consuming()
```

Message design tips:

- Keep messages **small** (KBs, not MBs). The default max message size is 16 MiB in RabbitMQ 4.x; large files belong in storage (S3, disk) — send the **path/URL** instead.
- Send **ids + minimal data**, and include a `schema_version` so you can change the format later.
- Don't send Python `pickle` — insecure and Python-only. Use JSON.

---

# Part 4 — Reliability and Advanced Features

## Step 16: Consumer Acknowledgements in Depth

### 16.1 The three answers a consumer can give

| Method                                               | Meaning                                       |
|------------------------------------------------------|-----------------------------------------------|
| `ch.basic_ack(delivery_tag)`                         | Success — delete the message                  |
| `ch.basic_nack(delivery_tag, requeue=True)`          | Failed — put it back in the queue             |
| `ch.basic_nack(delivery_tag, requeue=False)`         | Failed — drop it (or dead-letter it, Step 19) |
| `ch.basic_reject(delivery_tag, requeue=...)`         | Same as nack but for **one** message only (older AMQP method) |
| `basic_ack/basic_nack(..., multiple=True)`           | Applies to this tag **and all earlier unacked tags** on the channel (batch ack) |

### 16.2 Rules

1. **Ack on the same channel** that received the message. Delivery tags are per channel. Acking on another channel → `PRECONDITION_FAILED - unknown delivery tag`, and the channel is closed.
2. **Ack exactly once.** Double ack → same error.
3. **Ack after the work is done** (and after DB commit), not before.
4. If the channel closes, all its unacked messages are re-queued automatically.

### 16.3 The infinite requeue loop (poison message)

```python
def callback(ch, method, properties, body):
    try:
        process(body)
        ch.basic_ack(method.delivery_tag)
    except Exception:
        ch.basic_nack(method.delivery_tag, requeue=True)   # ❌ a broken message loops forever, 100% CPU
```

A message that **always** fails (bad JSON, missing record) will be redelivered immediately, forever. Fix:

```python
def callback(ch, method, properties, body):
    try:
        process(body)
    except TemporaryError:                         # e.g. DB/network down — worth retrying
        if method.redelivered:
            ch.basic_nack(method.delivery_tag, requeue=False)   # second failure → dead-letter
        else:
            ch.basic_nack(method.delivery_tag, requeue=True)    # first failure → try once more
        return
    except Exception:                              # permanent error — never retry
        ch.basic_nack(method.delivery_tag, requeue=False)
        return
    ch.basic_ack(method.delivery_tag)
```

Better: a proper retry with delay and a counter (Step 19.4), or **quorum queues**' built-in `delivery-limit` (Step 23).

### 16.4 Delivery guarantees

| Mode                                  | Guarantee          | Risk                                               |
|---------------------------------------|--------------------|----------------------------------------------------|
| `auto_ack=True`                       | **At most once**   | Messages lost if the consumer crashes              |
| Manual ack after processing           | **At least once**  | A message can be processed **twice** (crash after work, before ack) |
| "Exactly once"                        | Not provided by any broker alone | Achieve with at-least-once + **idempotent** consumers |

### 16.5 Idempotent consumers

Make processing the same message twice harmless:

```python
# blog example: a table with a unique message_id
from django.db import IntegrityError, transaction

def handle(properties, payload):
    try:
        with transaction.atomic():
            ProcessedMessage.objects.create(message_id=properties.message_id)  # unique=True
            do_the_work(payload)
    except IntegrityError:
        pass   # already processed → just ack again
```

Other techniques: `update_or_create`, "set status = paid" instead of "add 1", checking current state before acting.

---

## Step 17: Publisher Confirms and Unroutable Messages

Consumer acks protect the **broker → consumer** side. Publisher confirms protect the **producer → broker** side.

Without confirms, `basic_publish` returns as soon as bytes are written to the socket. If the broker crashes a millisecond later, or the exchange doesn't exist, you never find out.

### 17.1 Turning on confirms — `confirm_publish.py`

```python
import pika
from pika.exceptions import NackError, UnroutableError

from rmq import get_connection

connection = get_connection()
channel = connection.channel()
channel.exchange_declare(exchange="orders_ex", exchange_type="direct", durable=True)
channel.queue_declare(queue="orders_paid", durable=True)
channel.queue_bind(exchange="orders_ex", queue="orders_paid", routing_key="order.paid")

# Put the channel into confirm mode. From now on every publish waits for the broker's answer.
channel.confirm_delivery()

for routing_key in ("order.paid", "order.unknown"):
    try:
        channel.basic_publish(
            exchange="orders_ex",
            routing_key=routing_key,
            body=b'{"order_id": 1}',
            properties=pika.BasicProperties(delivery_mode=pika.DeliveryMode.Persistent),
            mandatory=True,    # return the message if no queue matches
        )
        print(f"{routing_key}: confirmed by broker")
    except UnroutableError:
        print(f"{routing_key}: no queue bound — message returned")
    except NackError:
        print(f"{routing_key}: broker could not store it (nack)")

connection.close()
```

Output:

```
order.paid: confirmed by broker
order.unknown: no queue bound — message returned
```

### 17.2 What each outcome means

| Outcome                     | Cause                                                                     | What to do                      |
|-----------------------------|---------------------------------------------------------------------------|---------------------------------|
| Returns normally (ack)      | Message is stored (for persistent + durable: written to disk / replicated) | Done                            |
| `UnroutableError`           | `mandatory=True` and no queue matched                                     | Log, fix bindings, alert        |
| `NackError`                 | Broker internal error (rare), queue overflow with `reject-publish`        | Retry later / store locally     |
| `ChannelClosedByBroker` 404 | Exchange does not exist                                                   | Declare it; fix the name        |
| Connection error            | Broker down                                                               | Reconnect and **re-publish**    |

### 17.3 Performance

With `BlockingChannel`, each confirmed publish waits for one round-trip (~ms) — fine for web requests (hundreds/second). For high throughput (thousands/second), publish asynchronously and handle confirms in batches — use `aio-pika` (Step 27), pika's `SelectConnection`, or Celery's `confirm_publish`.

### 17.4 The "outbox" problem

Even with confirms, this can happen in a Django view:

```
1. save order in DB       ✓
2. publish "order.created" ✗ (broker down)  → DB has the order, nobody is notified
```

Solutions:

- Publish in `transaction.on_commit()` and retry on failure (simple; Step 29).
- **Transactional outbox** (robust): in the same DB transaction, insert a row into an `outbox` table. A separate process reads unsent rows, publishes them with confirms, marks them sent.

---

## Step 18: Queue Arguments (TTL, Length Limits, Expiry)

Queue arguments are passed at declare time with `arguments={...}`. (Prefer **policies** in production — Step 24 — because arguments cannot be changed later.)

| Argument                      | Value                    | Effect                                                                      |
|-------------------------------|--------------------------|-----------------------------------------------------------------------------|
| `x-message-ttl`               | int ms                   | Messages older than this are removed (or dead-lettered)                     |
| `x-expires`                   | int ms                   | Delete the **queue** after it's unused (no consumers, no gets) for this long |
| `x-max-length`                | int                      | Max number of ready messages                                                |
| `x-max-length-bytes`          | int                      | Max total bytes of ready message bodies                                     |
| `x-overflow`                  | `drop-head` (default), `reject-publish`, `reject-publish-dlx` | What to do when full                     |
| `x-dead-letter-exchange`      | str                      | Where dead messages go (Step 19)                                            |
| `x-dead-letter-routing-key`   | str                      | Routing key to use when dead-lettering (default: original key)             |
| `x-max-priority`              | int 1–255                | Make it a priority queue (classic)                                          |
| `x-single-active-consumer`    | bool                     | Only one consumer receives at a time (Step 21)                              |
| `x-queue-type`                | `classic`, `quorum`, `stream` | Queue type (Step 23)                                                   |
| `x-delivery-limit`            | int                      | Quorum queues: dead-letter/drop after this many redeliveries               |
| `x-queue-leader-locator`      | `client-local`, `balanced` | Where the queue leader is placed in a cluster                            |

### 18.1 Message TTL

```python
# Every message in this queue expires after 60 seconds
channel.queue_declare(queue="otp_codes", durable=True, arguments={"x-message-ttl": 60_000})

# Per-message TTL (the lower of queue TTL and message TTL wins)
channel.basic_publish(
    exchange="", routing_key="otp_codes", body=b"123456",
    properties=pika.BasicProperties(expiration="30000"),   # string, ms
)
```

Notes:

- Classic queues remove expired messages only when they reach the **head** of the queue. A message with a short per-message TTL stuck behind one with a long TTL won't expire until it's at the head. Queue-level TTL doesn't have this problem.
- `x-message-ttl: 0` = deliver only if a consumer can take it right now, else drop/dead-letter.

### 18.2 Length limit

```python
# Keep only the latest 1000 notifications; oldest are dropped
channel.queue_declare(queue="notifications", durable=True,
                      arguments={"x-max-length": 1000, "x-overflow": "drop-head"})

# Refuse new messages when full (with confirms the publisher gets a nack)
channel.queue_declare(queue="uploads", durable=True,
                      arguments={"x-max-length": 10_000, "x-overflow": "reject-publish"})
```

| `x-overflow`          | When full…                                                                |
|-----------------------|---------------------------------------------------------------------------|
| `drop-head`           | Oldest message is removed (dead-lettered if a DLX is set)                 |
| `reject-publish`      | New publishes are rejected (publisher gets nack if confirms are on)       |
| `reject-publish-dlx`  | Like reject-publish, and the rejected message is dead-lettered (classic only) |

### 18.3 Queue expiry

```python
# Per-user temporary result queues — delete after 30 min of no use
channel.queue_declare(queue="results.user42", arguments={"x-expires": 1_800_000})
```

### 18.4 "inequivalent arg" errors

If a queue already exists with different arguments:

```
pika.exceptions.ChannelClosedByBroker: (406, "PRECONDITION_FAILED - inequivalent arg 'x-message-ttl' for queue 'otp_codes' in vhost 'blog': received '30000' but current is '60000'")
```

The channel is closed. Fix: delete the queue (`rabbitmqctl delete_queue -p blog otp_codes`) and redeclare, or use a policy (policies can be changed any time).

---

## Step 19: Dead Letter Exchanges and Retries

A message becomes **dead** when:

| Reason (in `x-death`) | Cause                                                       |
|-----------------------|-------------------------------------------------------------|
| `rejected`            | Consumer did `basic_nack`/`basic_reject` with `requeue=False` |
| `expired`             | TTL ran out                                                  |
| `maxlen`              | Queue length limit exceeded (drop-head / reject-publish-dlx) |
| `delivery_limit`      | Quorum queue redelivery limit exceeded                       |

If the queue has a **dead letter exchange (DLX)**, dead messages are re-published there instead of being deleted. RabbitMQ adds an `x-death` header (list) describing what happened: `queue`, `reason`, `count`, `exchange`, `routing-keys`, `time`. Also `x-first-death-queue`, `x-first-death-reason`, `x-first-death-exchange`.

### 19.1 Basic DLX setup — `dlx_setup.py`

```
                                  nack(requeue=False)
 producer ─▶ [ "tasks" direct ] ─▶ [ queue "tasks.main" ] ──────────────▶ [ "tasks.dlx" fanout ] ─▶ [ queue "tasks.dead" ]
                                    x-dead-letter-exchange=tasks.dlx                                  (inspect / alert / replay)
```

```python
from rmq import get_connection

connection = get_connection()
channel = connection.channel()

# 1. The dead-letter exchange and queue
channel.exchange_declare(exchange="tasks.dlx", exchange_type="fanout", durable=True)
channel.queue_declare(queue="tasks.dead", durable=True)
channel.queue_bind(exchange="tasks.dlx", queue="tasks.dead")

# 2. The main exchange and queue, pointing at the DLX
channel.exchange_declare(exchange="tasks", exchange_type="direct", durable=True)
channel.queue_declare(
    queue="tasks.main",
    durable=True,
    arguments={"x-dead-letter-exchange": "tasks.dlx"},
)
channel.queue_bind(exchange="tasks", queue="tasks.main", routing_key="task")

connection.close()
print("DLX topology declared")
```

### 19.2 Consumer that rejects bad messages — `dlx_consumer.py`

```python
import json

from rmq import get_connection

connection = get_connection()
channel = connection.channel()


def callback(ch, method, properties, body):
    try:
        data = json.loads(body)
        print("processing", data)
        ch.basic_ack(method.delivery_tag)
    except json.JSONDecodeError:
        print("bad JSON → dead-letter")
        ch.basic_nack(method.delivery_tag, requeue=False)   # goes to tasks.dlx → tasks.dead


channel.basic_qos(prefetch_count=10)
channel.basic_consume(queue="tasks.main", on_message_callback=callback)
channel.start_consuming()
```

Publish a good and a bad message:

```python
channel.basic_publish(exchange="tasks", routing_key="task", body=b'{"ok": true}')
channel.basic_publish(exchange="tasks", routing_key="task", body=b"not json")
```

Then look at `tasks.dead` in the UI → **Get messages**: you see `x-death` with `reason: rejected`.

### 19.3 Reading `x-death` in code

```python
def death_count(properties):
    deaths = (properties.headers or {}).get("x-death", [])
    return sum(d.get("count", 0) for d in deaths)
```

### 19.4 Retry with a delay (TTL + DLX "wait queue")

Pattern: failed messages go to a **wait queue** with a TTL and no consumers. When the TTL expires, the wait queue dead-letters them **back** to the main exchange. A header counts attempts; after N attempts the message goes to a final "parking lot" queue.

```
            ┌────────────────────────────── retry after 10 s (TTL expires → DLX back to "work") ───────┐
            ▼                                                                                          │
 [ "work" ] ─▶ [ work.main ] ─▶ consumer ── fails, attempt < 3 ──▶ publish to [ work.retry (TTL 10s) ] ─┘
                                   │
                                   └─ fails, attempt = 3 ──▶ publish to [ work.parking_lot ]  (humans look here)
```

`retry_setup.py`:

```python
from rmq import get_connection

connection = get_connection()
ch = connection.channel()

ch.exchange_declare(exchange="work", exchange_type="direct", durable=True)

# Main queue
ch.queue_declare(queue="work.main", durable=True)
ch.queue_bind(exchange="work", queue="work.main", routing_key="work.main")

# Wait queue: no consumers; after 10 s messages are dead-lettered back to work.main
ch.queue_declare(
    queue="work.retry",
    durable=True,
    arguments={
        "x-message-ttl": 10_000,
        "x-dead-letter-exchange": "work",
        "x-dead-letter-routing-key": "work.main",
    },
)

# Final resting place
ch.queue_declare(queue="work.parking_lot", durable=True)

connection.close()
```

`retry_consumer.py`:

```python
import random

import pika

from rmq import get_connection

MAX_ATTEMPTS = 3
connection = get_connection()
ch = connection.channel()


def process(body):
    if random.random() < 0.7:
        raise RuntimeError("flaky external API")
    print("  ✓ processed", body)


def callback(ch, method, properties, body):
    headers = properties.headers or {}
    attempt = headers.get("x-attempt", 1)
    try:
        process(body)
    except Exception as exc:
        print(f"  ✗ attempt {attempt} failed: {exc}")
        target = "work.retry" if attempt < MAX_ATTEMPTS else "work.parking_lot"
        # Re-publish a copy with an incremented counter (default exchange → queue by name)…
        ch.basic_publish(
            exchange="",
            routing_key=target,
            body=body,
            properties=pika.BasicProperties(
                delivery_mode=pika.DeliveryMode.Persistent,
                content_type=properties.content_type,
                message_id=properties.message_id,
                headers={**headers, "x-attempt": attempt + 1, "x-last-error": str(exc)},
            ),
        )
    # …then ack the original (success OR handed to retry/parking lot).
    ch.basic_ack(method.delivery_tag)


ch.basic_qos(prefetch_count=5)
ch.basic_consume(queue="work.main", on_message_callback=callback)
print(" [*] retry consumer running")
ch.start_consuming()
```

Publish a few messages to exchange `work`, routing key `work.main`, and watch them bounce through `work.retry` in the UI.

> For **production-grade safety**, enable confirms on the channel before re-publishing, so the original is only acked after the copy is stored. Multiple delay levels (10 s, 1 min, 10 min) = multiple wait queues.

### 19.5 Replaying dead letters

After fixing the bug, move messages from `tasks.dead` back to the main exchange:

- UI: with the shovel plugin enabled (`rabbitmq_shovel`, `rabbitmq_shovel_management`), open the queue → **Move messages** → destination queue.
- Or a small script: `basic_get` from the dead queue, `basic_publish` to the main exchange, `basic_ack`.

---

## Step 20: Delayed Messages

"Send this email in 15 minutes." Two options:

### 20.1 Option A — TTL + DLX (built in)

Exactly the wait-queue trick from 19.4: one queue per delay length (`delay.15m` with `x-message-ttl: 900000` → DLX to the real exchange). Works well for a **few fixed delays**. Does **not** work for arbitrary per-message delays in one queue (head-of-line blocking: a 1-hour message blocks a 1-minute message behind it).

### 20.2 Option B — Delayed message exchange plugin

A community plugin adding exchange type `x-delayed-message`.

1. Download the `.ez` file matching your RabbitMQ version from https://github.com/rabbitmq/rabbitmq-delayed-message-exchange/releases
2. Copy it into the plugins folder and enable it:

   ```powershell
   docker cp rabbitmq_delayed_message_exchange-4.x.x.ez rabbitmq:/opt/rabbitmq/plugins/
   docker exec rabbitmq rabbitmq-plugins enable rabbitmq_delayed_message_exchange
   ```

3. Use it:

   ```python
   channel.exchange_declare(
       exchange="delayed",
       exchange_type="x-delayed-message",
       durable=True,
       arguments={"x-delayed-type": "direct"},   # how to route once the delay is over
   )
   channel.queue_declare(queue="reminders", durable=True)
   channel.queue_bind(exchange="delayed", queue="reminders", routing_key="reminder")

   channel.basic_publish(
       exchange="delayed",
       routing_key="reminder",
       body=b'{"user_id": 7}',
       properties=pika.BasicProperties(headers={"x-delay": 15 * 60 * 1000}),   # ms
   )
   ```

Limitations (read the plugin README): delayed messages are stored on **one node** (not replicated), not meant for millions of delayed messages or very long delays, and `mandatory` doesn't work. For heavy scheduling use Celery `eta`/`countdown`, Celery Beat, or a database-backed scheduler.

---

## Step 21: Priority Queues and Single Active Consumer

### 21.1 Priority queues

```python
# Classic queue with 10 priority levels (0 = lowest, 10 = highest)
channel.queue_declare(queue="emails", durable=True, arguments={"x-max-priority": 10})

channel.basic_publish(exchange="", routing_key="emails", body=b"password reset",
                      properties=pika.BasicProperties(priority=9))
channel.basic_publish(exchange="", routing_key="emails", body=b"newsletter",
                      properties=pika.BasicProperties(priority=1))
```

- Higher priority messages are delivered first **among messages waiting in the queue**. If consumers are idle and messages are consumed instantly, priority has no effect. Use a low prefetch so the queue (not the consumer's buffer) holds the backlog.
- Each priority level costs memory/CPU — use small values (≤ 10; 2–5 is typical).
- Messages without a priority are treated as 0.
- **Quorum queues** (4.0+) support priorities without any argument: two levels — priority 0–4 = normal, 5+ = high.
- Alternative many teams prefer: **separate queues** (`emails.high`, `emails.low`) with dedicated workers.

### 21.2 Single active consumer (SAC)

Many consumers subscribe, but only **one** receives messages at a time; the others are hot standbys. If the active one disconnects, the next takes over. Keeps **strict ordering** with failover.

```python
channel.queue_declare(queue="ledger_updates", durable=True,
                      arguments={"x-single-active-consumer": True})
```

Start two consumers → UI shows one as *active*, the other *waiting*. Kill the active one → the other starts receiving.

### 21.3 Ordering guarantees in general

- A queue is FIFO. With **one consumer** and no requeues, order is preserved.
- With **many consumers**, messages are processed in parallel → order across consumers is not guaranteed.
- Requeued messages go back to (roughly) their original position, but may be delivered after newer ones.
- Need order **per entity** (per user/order) plus parallelism? Shard: route `user.<id % N>` to N queues each with one consumer (the `rabbitmq_consistent_hash_exchange` plugin does this), or use streams with SAC.

---

## Step 22: Alternate Exchanges and Exchange-to-Exchange Bindings

### 22.1 Alternate exchange (catch unroutable messages)

Instead of dropping messages no queue matches, send them to another exchange:

```python
channel.exchange_declare(exchange="unrouted", exchange_type="fanout", durable=True)
channel.queue_declare(queue="unrouted_messages", durable=True)
channel.queue_bind(exchange="unrouted", queue="unrouted_messages")

channel.exchange_declare(
    exchange="events",
    exchange_type="topic",
    durable=True,
    arguments={"alternate-exchange": "unrouted"},
)

channel.basic_publish(exchange="events", routing_key="nobody.listens.to.this", body=b"lost?")
# → lands in "unrouted_messages" instead of vanishing
```

### 22.2 Exchange-to-exchange bindings

An exchange can be bound to another exchange — build routing trees:

```python
channel.exchange_declare(exchange="all_events", exchange_type="topic", durable=True)
channel.exchange_declare(exchange="blog_events", exchange_type="topic", durable=True)

# Everything starting with "blog." that reaches all_events is forwarded to blog_events
channel.exchange_bind(destination="blog_events", source="all_events", routing_key="blog.#")
```

Use an `internal=True` exchange when only other exchanges should publish into it:

```python
channel.exchange_declare(exchange="audit_internal", exchange_type="fanout", durable=True, internal=True)
```

### 22.3 Deleting things from code

```python
channel.queue_delete(queue="tmp", if_unused=True, if_empty=True)   # only if no consumers and empty
channel.queue_purge(queue="tmp")                                    # remove ready messages
channel.exchange_delete(exchange="old_ex", if_unused=True)          # only if no bindings
channel.queue_unbind(queue="q", exchange="ex", routing_key="k")
channel.exchange_unbind(destination="d", source="s", routing_key="k")
```

---

## Step 23: Queue Types — Classic, Quorum, Stream

| Feature                          | **Classic**                      | **Quorum**                                      | **Stream**                                   |
|----------------------------------|----------------------------------|-------------------------------------------------|----------------------------------------------|
| Replicated across nodes          | No (single node)                 | **Yes** (Raft consensus)                        | Yes                                           |
| Survives node loss               | No                               | Yes, if a majority of replicas is up            | Yes                                           |
| Message removed after ack        | Yes                              | Yes                                             | **No** — append-only log, kept by size/age    |
| Replay old messages              | No                               | No                                              | **Yes** (consume from offset/timestamp)       |
| Multiple consumers               | Compete for messages             | Compete                                         | Each reads **all** messages independently     |
| Non-durable / exclusive          | Yes                              | No (always durable)                             | No                                            |
| Priorities                       | `x-max-priority`                 | 2 levels built in (4.0+)                        | No                                            |
| Poison message handling          | Manual                           | **`x-delivery-limit`** (default 20 in 4.0)      | n/a                                           |
| Best for                         | Temporary/exclusive queues, single-node dev, RPC reply queues | **Default for important data** in production | Event logs, fan-out to many readers, replay, huge volume |

> RabbitMQ 4.0 **removed** the old "classic mirrored queues" (HA policies `ha-mode`). Use quorum queues for replication.

### 23.1 Quorum queue

```python
channel.queue_declare(
    queue="payments",
    durable=True,                         # required
    arguments={
        "x-queue-type": "quorum",
        "x-delivery-limit": 5,            # after 5 failed deliveries → dead-letter (or drop)
        "x-dead-letter-exchange": "payments.dlx",
    },
)
```

- Works on a single node too (no replication, but same API — good habit).
- In a 3-node cluster, the queue has 3 replicas; tolerates 1 node failure. 5 nodes → tolerates 2.
- Quorum queues track redeliveries; a message requeued too many times is dead-lettered with reason `delivery_limit` — no more infinite poison loops.
- Safer DLX: `"x-dead-letter-strategy": "at-least-once"` (requires `"x-overflow": "reject-publish"`) guarantees dead-lettered messages aren't lost in transit.
- Don't use `global_qos=True` with quorum queues.

### 23.2 Stream

```python
# Declare a stream
channel.queue_declare(
    queue="page_views",
    durable=True,
    arguments={
        "x-queue-type": "stream",
        "x-max-length-bytes": 5_000_000_000,           # keep at most ~5 GB
        "x-max-age": "7D",                             # …and at most 7 days (Y, M, D, h, m, s)
        "x-stream-max-segment-size-bytes": 100_000_000,
    },
)

# Publish exactly like a queue
channel.basic_publish(exchange="", routing_key="page_views", body=b'{"path": "/posts/1"}')
```

Consume with pika (AMQP) — prefetch and manual ack are **required**, and you choose where to start:

```python
def on_view(ch, method, properties, body):
    print(body)
    ch.basic_ack(method.delivery_tag)   # acks only control flow; messages are NOT deleted

channel.basic_qos(prefetch_count=100)
channel.basic_consume(
    queue="page_views",
    on_message_callback=on_view,
    arguments={"x-stream-offset": "first"},   # "first", "last", "next", an int offset, or a datetime
)
channel.start_consuming()
```

Run the consumer twice — **both** get every message (unlike a queue). For high throughput, enable the `rabbitmq_stream` plugin and use the native stream protocol (port 5552) via the `rstream` Python library.

### 23.3 Which one should I pick?

- Important business messages (orders, payments, emails) → **quorum**.
- Temporary per-connection queues (RPC replies, exclusive pub/sub queues) → **classic**.
- Event history, many independent readers, replay → **stream**.

> **Lazy queues** (`x-queue-mode: lazy`) are obsolete: since 3.12, classic queues already keep messages on disk and load them on demand. The setting is ignored.

---

## Step 24: Policies

Hard-coded queue `arguments` can't be changed without deleting the queue. **Policies** apply the same settings from the server side, by **name pattern**, and can be changed any time.

### 24.1 Create a policy

```powershell
# All queues in vhost "blog" whose names start with "tasks." get a DLX and a max length.
docker exec rabbitmq rabbitmqctl set_policy -p blog TasksPolicy "^tasks\." '{\"dead-letter-exchange\":\"tasks.dlx\",\"max-length\":100000}' --apply-to queues --priority 1
```

bash/zsh:

```bash
rabbitmqctl set_policy -p blog TasksPolicy '^tasks\.' '{"dead-letter-exchange":"tasks.dlx","max-length":100000}' --apply-to queues --priority 1
```

| Part             | Meaning                                                                       |
|------------------|-------------------------------------------------------------------------------|
| `TasksPolicy`    | Policy name                                                                   |
| `"^tasks\."`     | Regex on queue/exchange names                                                 |
| JSON             | Definition (keys **without** `x-` prefix)                                    |
| `--apply-to`     | `queues`, `classic_queues`, `quorum_queues`, `streams`, `exchanges`, or `all` |
| `--priority`     | If several policies match, the **highest** priority wins (only one policy applies; they don't merge) |

Policy keys:

| Policy key               | Equivalent argument          |
|--------------------------|------------------------------|
| `message-ttl`            | `x-message-ttl`              |
| `expires`                | `x-expires`                  |
| `max-length`             | `x-max-length`               |
| `max-length-bytes`       | `x-max-length-bytes`         |
| `overflow`               | `x-overflow`                 |
| `dead-letter-exchange`   | `x-dead-letter-exchange`     |
| `dead-letter-routing-key`| `x-dead-letter-routing-key`  |
| `delivery-limit`         | `x-delivery-limit`           |
| `max-age`                | `x-max-age` (streams)        |
| `alternate-exchange`     | `alternate-exchange` (exchanges) |
| `queue-leader-locator`   | `x-queue-leader-locator`     |

If both a policy and an argument set the same thing, the **argument** usually wins (for numeric limits like TTL and max-length, the lower value is used).

### 24.2 List / clear

```powershell
docker exec rabbitmq rabbitmqctl list_policies -p blog
docker exec rabbitmq rabbitmqctl clear_policy -p blog TasksPolicy
```

UI: **Admin → Policies → Add / update a policy**. A queue's page shows which policy is applied ("Effective policy definition").

### 24.3 Operator policies

Admins can set **operator policies** that cap what apps can do (e.g. max-length ≤ 1 000 000) — `rabbitmqctl set_operator_policy`. They merge with normal policies, using the stricter value.

---

## Step 25: Connections, Channels, Heartbeats and Reconnecting

### 25.1 Rules for connections and channels

| Do                                                                    | Don't                                                          |
|-----------------------------------------------------------------------|----------------------------------------------------------------|
| Open **one connection per process** and keep it open                 | Open a connection per message (very slow; can exhaust the broker) |
| Use **one channel per thread**                                       | Share a channel between threads                                |
| Use **separate connections** for publishing and consuming in busy apps | Publish on the same connection you consume from under heavy load (broker flow control on the publisher slows consumer acks) |
| Close connections cleanly on shutdown                                  | Leave thousands of idle connections                            |
| Set `connection_name` so you can identify it in the UI                |                                                                 |

Channel-level errors (404, 406…) **close the channel**, not the connection. After `ChannelClosedByBroker`, open a new channel.

### 25.2 Heartbeats

A heartbeat is a tiny frame sent both ways so each side knows the other is alive (and firewalls/NATs don't drop the idle TCP connection).

- Negotiated at connect time; RabbitMQ default **60 seconds**.
- If no traffic is seen for ~2 intervals, the connection is considered dead.
- With `pika.BlockingConnection`, heartbeats are only processed **while pika is running** (inside `start_consuming`, `process_data_events`, or a call). If your callback blocks for 5 minutes, heartbeats are missed and the broker closes the connection → you see `StreamLostError` / `ConnectionResetError` on the next ack. Fix: Step 26.
- Never set `heartbeat=0` (disabled) in production.

### 25.3 A consumer that reconnects automatically — `robust_consumer.py`

```python
import time

import pika
from pika.exceptions import AMQPChannelError, AMQPConnectionError, ConnectionClosedByBroker

from rmq import get_connection


def callback(ch, method, properties, body):
    print("got", body)
    ch.basic_ack(method.delivery_tag)


def run():
    while True:
        try:
            connection = get_connection()
            channel = connection.channel()
            channel.queue_declare(queue="task_queue", durable=True)
            channel.basic_qos(prefetch_count=10)
            channel.basic_consume(queue="task_queue", on_message_callback=callback)
            print(" [*] connected, consuming")
            try:
                channel.start_consuming()
            except KeyboardInterrupt:
                channel.stop_consuming()
                connection.close()
                break
        except ConnectionClosedByBroker as e:
            # Broker closed us (restart, admin closed connection) — retry
            print(f"closed by broker: {e}; retrying in 5 s")
            time.sleep(5)
        except AMQPChannelError as e:
            # Channel error = bug in our code (wrong queue args, bad ack) — don't loop forever
            print(f"channel error: {e}; stopping")
            break
        except AMQPConnectionError as e:
            # Network problem / broker down — retry
            print(f"connection error: {e}; retrying in 5 s")
            time.sleep(5)


if __name__ == "__main__":
    run()
```

Test: run it, then `docker restart rabbitmq`. It reconnects when the broker is back.

### 25.4 Re-declare topology after reconnecting

On reconnect, **declare your exchanges/queues/bindings again** (as above). Exclusive and auto-delete queues are gone after a disconnect.

---

## Step 26: Threads and Long-Running Work With pika

`pika` objects are **not thread-safe**. Don't call `basic_ack` (or anything) on a channel from another thread.

For long tasks inside a `BlockingConnection` consumer, run the work in a thread and hand the ack back to the connection's thread with `add_callback_threadsafe`:

`threaded_consumer.py`:

```python
import functools
import threading
import time

from rmq import get_connection


def ack_message(ch, delivery_tag):
    """Runs in the connection's thread."""
    if ch.is_open:
        ch.basic_ack(delivery_tag)
    # else: channel died; the message was already re-queued by the broker


def do_work(ch, delivery_tag, body):
    """Runs in a worker thread — may take minutes; heartbeats keep flowing meanwhile."""
    print(f"working on {body!r} in {threading.current_thread().name}")
    time.sleep(120)
    cb = functools.partial(ack_message, ch, delivery_tag)
    ch.connection.add_callback_threadsafe(cb)       # the ONLY thread-safe pika method


def on_message(ch, method, properties, body, threads):
    t = threading.Thread(target=do_work, args=(ch, method.delivery_tag, body))
    t.start()
    threads.append(t)


connection = get_connection()
channel = connection.channel()
channel.queue_declare(queue="long_tasks", durable=True)
channel.basic_qos(prefetch_count=1)     # = max parallel threads

threads = []
channel.basic_consume(queue="long_tasks",
                      on_message_callback=functools.partial(on_message, threads=threads))
try:
    channel.start_consuming()
except KeyboardInterrupt:
    channel.stop_consuming()

for t in threads:
    t.join()
connection.close()
```

Alternatives: use a separate **process** per consumer (simplest — just run several `worker.py`), or use Celery, which handles all of this.

For **web apps** (Django with threads): don't share one pika connection between request threads. Use one connection per thread (`threading.local()`), a connection pool, or open a short-lived connection per publish when traffic is low (Step 29).

---

## Step 27: Async Python With aio-pika

```powershell
pip install aio-pika
```

`async_producer.py`:

```python
import asyncio
import json

import aio_pika

URL = "amqp://blog:blogpass@localhost:5672/blog"


async def main():
    connection = await aio_pika.connect_robust(URL)        # robust = auto-reconnect + re-declare
    async with connection:
        channel = await connection.channel(publisher_confirms=True)   # confirms on by default
        exchange = await channel.declare_exchange("events", aio_pika.ExchangeType.TOPIC, durable=True)

        for i in range(10):
            message = aio_pika.Message(
                body=json.dumps({"n": i}).encode(),
                content_type="application/json",
                delivery_mode=aio_pika.DeliveryMode.PERSISTENT,
            )
            await exchange.publish(message, routing_key="blog.post.created")   # awaits the confirm
        print("sent 10")


asyncio.run(main())
```

`async_consumer.py`:

```python
import asyncio
import json

import aio_pika

URL = "amqp://blog:blogpass@localhost:5672/blog"


async def main():
    connection = await aio_pika.connect_robust(URL)
    async with connection:
        channel = await connection.channel()
        await channel.set_qos(prefetch_count=10)

        exchange = await channel.declare_exchange("events", aio_pika.ExchangeType.TOPIC, durable=True)
        queue = await channel.declare_queue("search_indexer", durable=True,
                                            arguments={"x-queue-type": "quorum"})
        await queue.bind(exchange, routing_key="blog.post.*")

        async with queue.iterator() as messages:
            async for message in messages:
                # process(): acks on success; on exception it rejects (requeue=False by default)
                async with message.process():
                    data = json.loads(message.body)
                    print(message.routing_key, data)
                    await asyncio.sleep(0.1)       # real async work (HTTP call, DB…)


asyncio.run(main())
```

Notes:

- `message.process(requeue=True)` requeues on exception; `ignore_processed=True` lets you ack/nack manually inside.
- Many messages in parallel: `prefetch_count` > 1 + create tasks (`asyncio.create_task`) per message.
- FastAPI: create the connection in the app's `lifespan` startup and close it on shutdown.

---

# Part 5 — Django Integration

Two different ways to use RabbitMQ from the `blog` project:

| Approach                         | Use when                                                                           |
|----------------------------------|------------------------------------------------------------------------------------|
| **Celery with RabbitMQ broker** (Step 28) | You want background **tasks** in Python (send email, resize image). Celery hides RabbitMQ details |
| **Raw pika events** (Steps 29–30) | You want to publish **events** other services (maybe not Python, maybe not Celery) consume |

## Step 28: Use RabbitMQ as the Celery Broker

This continues [django-celery-guide.md](django-celery-guide.md), which used Redis. Switching the broker is a settings change.

### 28.1 Packages

```powershell
cd "C:\Users\Hareram ray\web\backend-practical\blog"     # your Django project folder
.\venv\Scripts\Activate.ps1
pip install "celery>=5.4" django-celery-results
```

Celery talks to RabbitMQ through **kombu** + **py-amqp**, installed automatically with Celery. Nothing else needed.

### 28.2 Settings — `config/settings.py`

```python
import os

# --- Celery with RabbitMQ ---
CELERY_BROKER_URL = os.environ.get(
    "CELERY_BROKER_URL",
    "amqp://blog:blogpass@localhost:5672/blog",
)

# RabbitMQ is a BROKER, not a result store. Keep results in the DB (or Redis).
CELERY_RESULT_BACKEND = "django-db"

CELERY_ACCEPT_CONTENT = ["json"]
CELERY_TASK_SERIALIZER = "json"
CELERY_RESULT_SERIALIZER = "json"
CELERY_TIMEZONE = TIME_ZONE
CELERY_TASK_TRACK_STARTED = True
CELERY_BROKER_CONNECTION_RETRY_ON_STARTUP = True

# Reliability
CELERY_TASK_ACKS_LATE = True             # ack AFTER the task finishes (redeliver if worker dies)
CELERY_TASK_REJECT_ON_WORKER_LOST = True # requeue if the worker process is killed mid-task
CELERY_WORKER_PREFETCH_MULTIPLIER = 1    # prefetch = concurrency × 1 → fair for slow tasks

# Publisher confirms: Django gets an error if RabbitMQ didn't store the task
CELERY_BROKER_TRANSPORT_OPTIONS = {"confirm_publish": True}

# Use quorum queues for Celery's queues (Celery 5.5+)
CELERY_TASK_DEFAULT_QUEUE_TYPE = "quorum"
```

Make sure `"django_celery_results"` is in `INSTALLED_APPS` and you ran `python manage.py migrate`.

### 28.3 What Celery creates in RabbitMQ

Start a worker:

```powershell
celery -A config worker -l info --pool=solo      # Windows needs --pool=solo (or threads)
```

The banner shows `transport: amqp://blog:**@localhost:5672/blog`. In the UI (vhost `blog`):

| Object                                 | What it is                                                                 |
|----------------------------------------|----------------------------------------------------------------------------|
| Exchange `celery` (direct)             | Default exchange for tasks                                                 |
| Queue `celery`                         | Default task queue (routing key `celery`)                                  |
| Exchanges `celery.pidbox`, `celeryev`  | Remote control (`celery inspect/control`) and events (Flower monitoring)   |
| Queues `<worker>.celery.pidbox`        | One per worker for remote-control commands                                  |

Call a task (`send_welcome_email.delay(user.id)`) and watch the message go through queue `celery`.

### 28.4 Routing tasks to separate queues

```python
CELERY_TASK_ROUTES = {
    "blog.tasks.send_welcome_email": {"queue": "emails"},
    "blog.tasks.generate_pdf": {"queue": "pdfs"},
}
```

```powershell
celery -A config worker -l info --pool=solo -Q emails        # worker only for emails
celery -A config worker -l info --pool=solo -Q pdfs,celery   # another for pdfs + default
```

Celery declares the `emails` and `pdfs` queues for you.

### 28.5 Celery + RabbitMQ gotchas

| Problem                                                    | Explanation / fix                                                                         |
|------------------------------------------------------------|-------------------------------------------------------------------------------------------|
| Tasks with `eta`/`countdown` far in the future get redelivered or the channel closes | ETA tasks are held **unacked** by the worker; after `consumer_timeout` (30 min default) RabbitMQ closes the channel. Use short delays only, raise `consumer_timeout` (Step 31), or use Celery 5.5's native delayed delivery for quorum queues |
| `PRECONDITION_FAILED inequivalent arg 'x-queue-type'`      | You switched to quorum but the `celery` queue already exists as classic. Delete the queue (when empty) and restart the worker |
| `global QoS` errors with quorum queues                     | Celery 5.5+ detects quorum queues and disables global QoS (`worker_detect_quorum_queues`, default on). Upgrade Celery |
| Result backend `rpc://`                                    | Sends results back as messages — results can be read **once** by the caller. Prefer `django-db` or Redis |
| Workers can't connect from another container               | Use the service name as host (`amqp://blog:blogpass@rabbitmq:5672/blog`), not `localhost` |

---

## Step 29: Publish Domain Events From Django (pika)

Goal: when a `Post` is published in the `blog` app, publish a `blog.post.published` event to a topic exchange. Any service (search indexer, notifications, analytics — any language) can subscribe.

### 29.1 Install pika in the Django project

```powershell
pip install pika
```

### 29.2 Settings

```python
# config/settings.py
RABBITMQ_URL = os.environ.get("RABBITMQ_URL", "amqp://blog:blogpass@localhost:5672/blog")
RABBITMQ_EVENTS_EXCHANGE = "blog.events"
```

### 29.3 Messaging module — `blog/messaging.py`

```python
import json
import logging
import threading
import time
import uuid

import pika
from django.conf import settings
from django.core.serializers.json import DjangoJSONEncoder

logger = logging.getLogger(__name__)

# One connection+channel per thread: pika objects must not be shared between threads.
_local = threading.local()


def _connect():
    params = pika.URLParameters(settings.RABBITMQ_URL)
    params.heartbeat = 60
    params.blocked_connection_timeout = 30
    params.client_properties = {"connection_name": "blog-web"}
    connection = pika.BlockingConnection(params)
    channel = connection.channel()
    channel.exchange_declare(exchange=settings.RABBITMQ_EVENTS_EXCHANGE, exchange_type="topic", durable=True)
    channel.confirm_delivery()
    return connection, channel


def _get_channel():
    channel = getattr(_local, "channel", None)
    if channel is None or channel.is_closed or channel.connection.is_closed:
        _local.connection, _local.channel = _connect()
    return _local.channel


def publish_event(routing_key, payload, retries=3):
    """Publish a persistent JSON event with publisher confirms. Raises after `retries` failures."""
    body = json.dumps(payload, cls=DjangoJSONEncoder).encode("utf-8")
    properties = pika.BasicProperties(
        content_type="application/json",
        delivery_mode=pika.DeliveryMode.Persistent,
        message_id=str(uuid.uuid4()),
        timestamp=int(time.time()),
        type=routing_key,
        app_id="blog",
    )
    for attempt in range(1, retries + 1):
        try:
            _get_channel().basic_publish(
                exchange=settings.RABBITMQ_EVENTS_EXCHANGE,
                routing_key=routing_key,
                body=body,
                properties=properties,
            )
            return properties.message_id
        except (pika.exceptions.AMQPError, OSError) as exc:
            logger.warning("publish %s failed (attempt %s/%s): %s", routing_key, attempt, retries, exc)
            _local.channel = None          # force reconnect on the next attempt
            time.sleep(0.5 * attempt)
    raise RuntimeError(f"Could not publish {routing_key} after {retries} attempts")
```

Why `mandatory` is **not** set: for an event bus, "nobody is subscribed yet" is normal. (Use `mandatory=True` + an alternate exchange if you need to know.)

### 29.4 Publish after the DB commit — `blog/signals.py`

```python
from functools import partial

from django.db import transaction
from django.db.models.signals import post_save
from django.dispatch import receiver

from .messaging import publish_event
from .models import Post


@receiver(post_save, sender=Post)
def post_saved(sender, instance, created, **kwargs):
    if instance.status != "published":
        return
    payload = {
        "post_id": instance.pk,
        "title": instance.title,
        "slug": instance.slug,
        "author_id": instance.author_id,
        "published_at": instance.publish,
        "schema_version": 1,
    }
    # Only publish if the transaction commits — never announce data that was rolled back.
    transaction.on_commit(partial(publish_event, "blog.post.published", payload))
```

(Adjust field names to your `Post` model.) Register the signals in `blog/apps.py`:

```python
from django.apps import AppConfig


class BlogConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "blog"

    def ready(self):
        from . import signals  # noqa: F401
```

### 29.5 Try it

1. Publish a post in the admin.
2. In the UI → **Exchanges → blog.events** you see a message rate — but with no queue bound, the event is dropped. Next step adds a consumer with a durable queue.

---

## Step 30: A Consumer as a Django Management Command

A long-running process that consumes `blog.post.*` events and can use the Django ORM.

### 30.1 Folders

```
blog/
  management/
    __init__.py
    commands/
      __init__.py
      consume_post_events.py
```

### 30.2 `blog/management/commands/consume_post_events.py`

```python
import json
import logging
import time

import pika
from django.conf import settings
from django.core.management.base import BaseCommand
from django.db import close_old_connections

logger = logging.getLogger(__name__)

QUEUE = "blog.post_events.notifier"
DLX = "blog.events.dlx"
DEAD_QUEUE = "blog.post_events.notifier.dead"


class Command(BaseCommand):
    help = "Consume blog.post.* events from RabbitMQ"

    def add_arguments(self, parser):
        parser.add_argument("--prefetch", type=int, default=10)

    def handle(self, *args, **options):
        while True:
            try:
                self.consume(options["prefetch"])
            except KeyboardInterrupt:
                self.stdout.write("Stopping.")
                return
            except pika.exceptions.AMQPChannelError as exc:
                logger.error("Channel error, stopping: %s", exc)
                raise
            except (pika.exceptions.AMQPConnectionError, OSError) as exc:
                logger.warning("Connection lost (%s); reconnecting in 5 s", exc)
                time.sleep(5)

    def consume(self, prefetch):
        params = pika.URLParameters(settings.RABBITMQ_URL)
        params.heartbeat = 60
        params.client_properties = {"connection_name": "blog-notifier"}
        connection = pika.BlockingConnection(params)
        channel = connection.channel()

        # Topology (idempotent)
        channel.exchange_declare(exchange=settings.RABBITMQ_EVENTS_EXCHANGE, exchange_type="topic", durable=True)
        channel.exchange_declare(exchange=DLX, exchange_type="fanout", durable=True)
        channel.queue_declare(queue=DEAD_QUEUE, durable=True)
        channel.queue_bind(exchange=DLX, queue=DEAD_QUEUE)
        channel.queue_declare(
            queue=QUEUE,
            durable=True,
            arguments={
                "x-queue-type": "quorum",
                "x-delivery-limit": 5,
                "x-dead-letter-exchange": DLX,
            },
        )
        channel.queue_bind(exchange=settings.RABBITMQ_EVENTS_EXCHANGE, queue=QUEUE, routing_key="blog.post.*")

        channel.basic_qos(prefetch_count=prefetch)
        channel.basic_consume(queue=QUEUE, on_message_callback=self.on_message)
        self.stdout.write(self.style.SUCCESS(f"Consuming from {QUEUE}. Ctrl+C to stop."))
        try:
            channel.start_consuming()
        finally:
            if connection.is_open:
                connection.close()

    def on_message(self, ch, method, properties, body):
        # Long-running processes must drop stale DB connections (like Django does per request).
        close_old_connections()
        try:
            event = json.loads(body)
            self.handle_event(method.routing_key, event)
        except (json.JSONDecodeError, KeyError) as exc:
            logger.error("Bad message %s: %s", properties.message_id, exc)
            ch.basic_nack(method.delivery_tag, requeue=False)      # → dead-letter queue
            return
        except Exception:
            logger.exception("Temporary failure for %s, requeueing", properties.message_id)
            ch.basic_nack(method.delivery_tag, requeue=True)       # quorum delivery-limit stops loops
            return
        ch.basic_ack(method.delivery_tag)

    def handle_event(self, routing_key, event):
        from django.contrib.auth import get_user_model

        if routing_key == "blog.post.published":
            User = get_user_model()
            author = User.objects.get(pk=event["author_id"])
            # Here you would send notifications, update a search index, etc.
            self.stdout.write(f"New post '{event['title']}' by {author.username} → notify followers")
```

### 30.3 Run it

```powershell
python manage.py consume_post_events
```

Publish a post in the admin → the consumer prints the notification line. Stop the consumer, publish two posts, start it again → it catches up (durable quorum queue).

### 30.4 Run it in production

Run it as a separate long-running process, like a Celery worker: a systemd service, a Docker container (`command: python manage.py consume_post_events`), Supervisor, or a Kubernetes Deployment. Scale by running more copies — they share the queue.

---

# Part 6 — Operations

## Step 31: Configuration Files and Ports

### 31.1 Files

| File                | Format          | Purpose                                                    |
|---------------------|-----------------|------------------------------------------------------------|
| `rabbitmq.conf`     | `key = value`   | Main configuration                                         |
| `advanced.config`   | Erlang terms    | Rare settings not expressible in `rabbitmq.conf`           |
| `rabbitmq-env.conf` (Linux) / `rabbitmq-env-conf.bat` (Windows) | env vars | Node name, paths, ports      |
| `enabled_plugins`   | Erlang list     | `[rabbitmq_management,rabbitmq_prometheus].`               |

Locations:

| Install       | Config folder                                                  |
|---------------|----------------------------------------------------------------|
| Docker        | `/etc/rabbitmq/` (plus `/etc/rabbitmq/conf.d/*.conf`)          |
| Linux package | `/etc/rabbitmq/`                                               |
| Windows       | `%APPDATA%\RabbitMQ\` (create `rabbitmq.conf` there; re-install the service after changing env vars) |
| Homebrew      | `$(brew --prefix)/etc/rabbitmq/`                               |

Check which file is loaded: `rabbitmq-diagnostics status` (section "Config files") and the effective values with `rabbitmq-diagnostics environment`.

### 31.2 A commented `rabbitmq.conf`

```ini
## ---- Networking ----
listeners.tcp.default = 5672
management.tcp.port = 15672

## ---- Default user (only used when the database is empty) ----
default_user = admin
default_pass = change-me
default_vhost = /
loopback_users.guest = true          # guest only from localhost

## ---- Resource limits ----
vm_memory_high_watermark.relative = 0.6    # block publishers above 60 % of RAM
disk_free_limit.absolute = 2GB             # block publishers when free disk < 2 GB (default 50MB is too low)

## ---- Connections ----
heartbeat = 60
channel_max = 128                     # max channels per connection
# connection_max = 10000

## ---- Consumers ----
consumer_timeout = 3600000            # ms; 1 hour to ack a delivery (default 30 min)

## ---- Messages ----
max_message_size = 16777216           # bytes (16 MiB)

## ---- Logging ----
log.console = true
log.console.level = info
log.file.level = info

## ---- Load definitions on boot (Step 33) ----
# definitions.import_backend = local_filesystem
# definitions.local.path = /etc/rabbitmq/definitions.json
```

Docker: mount it as `./rabbitmq/rabbitmq.conf:/etc/rabbitmq/conf.d/20-custom.conf:ro` (keeps the image's defaults file) and restart.

### 31.3 Ports

| Port         | Used by                                               |
|--------------|-------------------------------------------------------|
| **5672**     | AMQP 0-9-1 and AMQP 1.0 clients                       |
| **5671**     | AMQP over TLS                                         |
| **15672**    | Management UI + HTTP API                              |
| 15671        | Management over HTTPS (if configured)                 |
| 15692        | Prometheus metrics                                    |
| 5552 / 5551  | Stream protocol / over TLS                            |
| 4369         | epmd (Erlang port mapper) — clustering & CLI          |
| 25672        | Inter-node and CLI communication                      |
| 35672–35682  | CLI tools                                             |
| 1883 / 8883  | MQTT / MQTT over TLS                                  |
| 61613 / 61614 | STOMP / STOMP over TLS                               |
| 15674 / 15675 | Web-STOMP / Web-MQTT (WebSockets)                    |

Only expose **5672/5671** (apps) and **15672** (admins, ideally behind a VPN) to the outside. Never expose 4369/25672 publicly.

### 31.4 Environment variables (common)

| Variable                        | Meaning                                           |
|---------------------------------|---------------------------------------------------|
| `RABBITMQ_NODENAME`             | Node name, default `rabbit@<hostname>`            |
| `RABBITMQ_CONFIG_FILE`          | Path to `rabbitmq.conf`                           |
| `RABBITMQ_MNESIA_BASE` / `RABBITMQ_DATA_DIR` | Data directory                      |
| `RABBITMQ_LOG_BASE`             | Log directory                                     |
| `RABBITMQ_DEFAULT_USER/PASS/VHOST` | First-boot user (Docker image)                 |

---

## Step 32: Memory, Disk Alarms and Flow Control

RabbitMQ protects itself when resources run low.

### 32.1 Memory alarm

When memory use passes `vm_memory_high_watermark` (default 60 % of RAM), RabbitMQ raises a **memory alarm**:

- **All publishing connections are blocked** (state `blocked` in the UI). `basic_publish` hangs.
- Consumers keep running, draining queues; when memory drops, publishing resumes.

### 32.2 Disk alarm

When free disk space falls below `disk_free_limit` (default 50 MB — **raise it**, e.g. 2 GB or `disk_free_limit.relative = 1.5` × RAM), publishers are blocked the same way.

### 32.3 Check alarms

```powershell
docker exec rabbitmq rabbitmq-diagnostics check_local_alarms
docker exec rabbitmq rabbitmq-diagnostics alarms
docker exec rabbitmq rabbitmq-diagnostics memory_breakdown --unit mb
```

UI → **Overview → Nodes**: memory and disk bars turn red.

### 32.4 What clients should do

- Set `blocked_connection_timeout` in pika (Step 7): if blocked longer than that, pika raises `ConnectionBlockedTimeout` instead of hanging forever.
- Watch queue lengths — a queue growing without bound is the usual cause.

### 32.5 Flow control (credit flow)

Even without alarms, if a queue can't keep up, RabbitMQ slows fast publishers down (connection state `flow` in the UI). It's normal under load; constant `flow` means consumers or disk are the bottleneck.

---

## Step 33: Definitions — Export and Import Your Setup

**Definitions** = users, vhosts, permissions, exchanges, queues, bindings, policies (not messages). Store them in Git so a fresh broker is set up automatically.

### 33.1 Export

```powershell
docker exec rabbitmq rabbitmqctl export_definitions /tmp/definitions.json
docker cp rabbitmq:/tmp/definitions.json .\rabbitmq\definitions.json
```

Or UI → **Overview → Export definitions**. Or HTTP: `curl -u admin:admin123 http://localhost:15672/api/definitions -o definitions.json`.

### 33.2 Example `definitions.json`

```json
{
  "users": [
    {"name": "blog", "password": "blogpass", "tags": ["monitoring"]}
  ],
  "vhosts": [{"name": "blog"}],
  "permissions": [
    {"user": "blog", "vhost": "blog", "configure": ".*", "write": ".*", "read": ".*"}
  ],
  "exchanges": [
    {"name": "blog.events", "vhost": "blog", "type": "topic", "durable": true, "auto_delete": false, "internal": false, "arguments": {}}
  ],
  "queues": [
    {"name": "blog.post_events.notifier", "vhost": "blog", "durable": true, "auto_delete": false,
     "arguments": {"x-queue-type": "quorum"}}
  ],
  "bindings": [
    {"source": "blog.events", "vhost": "blog", "destination": "blog.post_events.notifier",
     "destination_type": "queue", "routing_key": "blog.post.*", "arguments": {}}
  ],
  "policies": []
}
```

(Exported files contain `password_hash` instead of `password` — both forms are accepted on import. Don't commit real passwords; generate the file from secrets in CI.)

### 33.3 Import

```powershell
docker cp .\rabbitmq\definitions.json rabbitmq:/tmp/definitions.json
docker exec rabbitmq rabbitmqctl import_definitions /tmp/definitions.json
```

Or load automatically on boot (`rabbitmq.conf`):

```ini
definitions.import_backend = local_filesystem
definitions.local.path = /etc/rabbitmq/definitions.json
```

---

## Step 34: Monitoring (UI, HTTP API, Prometheus)

### 34.1 What to watch

| Metric                           | Why                                                          | Alert when                        |
|----------------------------------|--------------------------------------------------------------|-----------------------------------|
| Messages **ready** per queue     | Backlog                                                      | Keeps growing / over threshold    |
| Messages **unacked**             | Stuck consumers                                              | High and not moving               |
| **Consumers** per queue          | Workers alive?                                               | 0 on an important queue           |
| Publish / deliver / ack rates    | Throughput                                                   | Deliver rate < publish rate for long |
| Memory & disk, **alarms**        | Publishers get blocked                                       | Any alarm                         |
| Connections / channels count     | Leaks (opening a connection per message)                     | Constantly increasing             |
| File descriptors                 | Each connection uses one                                     | > 80 %                            |
| Dead-letter queue length         | Failed messages                                              | > 0                               |
| Node up / cluster partitions     | Availability                                                 | Node down                         |

### 34.2 HTTP API (management plugin)

All UI data is available as JSON. vhost `/` is written `%2F`.

```powershell
# PowerShell has curl.exe built in (use curl.exe, not the curl alias for Invoke-WebRequest)
curl.exe -u admin:admin123 http://localhost:15672/api/overview
curl.exe -u admin:admin123 http://localhost:15672/api/queues/blog
curl.exe -u admin:admin123 http://localhost:15672/api/queues/blog/blog.post_events.notifier
curl.exe -u admin:admin123 http://localhost:15672/api/connections
curl.exe -u admin:admin123 http://localhost:15672/api/health/checks/alarms
curl.exe -u admin:admin123 http://localhost:15672/api/aliveness-test/blog   # (deprecated in newer versions; prefer health checks)
```

Publish through HTTP (testing only — slow):

```powershell
curl.exe -u admin:admin123 -H "content-type: application/json" `
  -X POST http://localhost:15672/api/exchanges/blog/amq.default/publish `
  -d '{\"properties\":{},\"routing_key\":\"hello\",\"payload\":\"hi from http\",\"payload_encoding\":\"string\"}'
```

Python check (e.g. a health endpoint or cron alert):

```python
import requests

r = requests.get("http://localhost:15672/api/queues/blog", auth=("admin", "admin123"), timeout=5)
for q in r.json():
    print(f"{q['name']:40} ready={q.get('messages_ready', 0):6} unacked={q.get('messages_unacknowledged', 0):6} consumers={q.get('consumers', 0)}")
```

### 34.3 Prometheus + Grafana

```powershell
docker exec rabbitmq rabbitmq-plugins enable rabbitmq_prometheus
```

Expose port `15692` (`-p 15692:15692`) and open http://localhost:15692/metrics. Point Prometheus at it and import the official RabbitMQ Grafana dashboards ("RabbitMQ-Overview", ID 10991). Per-object metrics: `/metrics/per-object` or `/metrics/detailed`.

### 34.4 Logs

```powershell
docker logs -f rabbitmq
docker exec rabbitmq rabbitmq-diagnostics log_tail -N 100
```

Look for: `accepting AMQP connection`, `closing AMQP connection`, `missed heartbeats`, `memory resource limit alarm set`, `client unexpectedly closed TCP connection`.

### 34.5 Tracing messages (debugging)

- **Firehose**: `rabbitmqctl trace_on -p blog` copies every message to `amq.rabbitmq.trace`. Turn off with `trace_off`. Heavy — dev only.
- `rabbitmq_tracing` plugin: UI tab **Admin → Tracing** to write traces to files.

---

## Step 35: Security (Users, TLS)

### 35.1 Checklist

1. Delete `guest`: `rabbitmqctl delete_user guest`.
2. One user per application, permissions limited to its vhost and, ideally, its own queue/exchange names.
3. Strong passwords from environment variables / a secrets manager — never in Git.
4. Don't expose the management UI to the internet; restrict by firewall/VPN.
5. Use **TLS** (`amqps://`) whenever traffic leaves a trusted network.
6. Keep RabbitMQ and Erlang updated.
7. Set the Erlang cookie to a long random secret (cluster/CLI authentication) and keep port 25672/4369 private.
8. Use `user_id` property if consumers must trust who published.

### 35.2 Enable TLS on the broker (`rabbitmq.conf`)

```ini
listeners.ssl.default = 5671

ssl_options.cacertfile = /etc/rabbitmq/certs/ca_certificate.pem
ssl_options.certfile   = /etc/rabbitmq/certs/server_certificate.pem
ssl_options.keyfile    = /etc/rabbitmq/certs/server_key.pem
ssl_options.verify     = verify_peer
ssl_options.fail_if_no_peer_cert = false    # true = require client certificates (mTLS)

# Optionally disable plain AMQP entirely:
# listeners.tcp = none

# HTTPS for the management UI
# management.ssl.port       = 15671
# management.ssl.cacertfile = /etc/rabbitmq/certs/ca_certificate.pem
# management.ssl.certfile   = /etc/rabbitmq/certs/server_certificate.pem
# management.ssl.keyfile    = /etc/rabbitmq/certs/server_key.pem
```

For local testing, generate certificates with https://github.com/rabbitmq/tls-gen (`make` → `result/` folder).

### 35.3 Connect with TLS from Python

```python
import ssl

import pika

context = ssl.create_default_context(cafile="ca_certificate.pem")
# For mTLS: context.load_cert_chain("client_certificate.pem", "client_key.pem")

params = pika.ConnectionParameters(
    host="rabbit.example.com",
    port=5671,
    virtual_host="blog",
    credentials=pika.PlainCredentials("blog", "blogpass"),
    ssl_options=pika.SSLOptions(context, server_hostname="rabbit.example.com"),
)
connection = pika.BlockingConnection(params)
```

With a URL: `amqps://blog:blogpass@rabbit.example.com:5671/blog` (uses the system CA store). Celery: `CELERY_BROKER_URL = "amqps://..."` and `CELERY_BROKER_USE_SSL = {"ca_certs": "...", "cert_reqs": ssl.CERT_REQUIRED}`.

---

## Step 36: Clustering

A **cluster** is several nodes that share metadata (users, vhosts, exchanges, bindings, queue definitions). Messages live in queues; with **quorum queues/streams** they're replicated to several nodes.

### 36.1 Facts to know

- Use an **odd** number of nodes: **3** (tolerates 1 failure) or **5** (tolerates 2).
- All nodes must share the same **Erlang cookie** and run compatible versions.
- Nodes find each other by **hostname** — names must resolve.
- Clients can connect to **any** node; put a load balancer (HAProxy, cloud LB) or a list of hosts in front.
- Clustering is for a **LAN** (same data centre). Across regions use Federation/Shovel (Step 37).
- RabbitMQ 4.x stores metadata in **Khepri** (Raft-based), replacing Mnesia — more predictable during network partitions.

### 36.2 A 3-node cluster with docker-compose (for learning)

`cluster/docker-compose.yml`:

```yaml
x-rabbit: &rabbit
  image: rabbitmq:4-management
  environment:
    RABBITMQ_ERLANG_COOKIE: "a-long-random-secret-cookie-value"   # same on all nodes
    RABBITMQ_DEFAULT_USER: admin
    RABBITMQ_DEFAULT_PASS: admin123

services:
  rabbit1:
    <<: *rabbit
    hostname: rabbit1
    ports: ["5672:5672", "15672:15672"]
  rabbit2:
    <<: *rabbit
    hostname: rabbit2
    ports: ["5673:5672", "15673:15672"]
  rabbit3:
    <<: *rabbit
    hostname: rabbit3
    ports: ["5674:5672", "15674:15672"]
```

> `RABBITMQ_ERLANG_COOKIE` is deprecated (it still works in current images and logs a warning). In production, mount a `.erlang.cookie` file (permissions `600`) at `/var/lib/rabbitmq/.erlang.cookie` instead.

Start and join nodes 2 and 3 to node 1:

```powershell
cd cluster
docker compose up -d

docker compose exec rabbit2 rabbitmqctl stop_app
docker compose exec rabbit2 rabbitmqctl reset
docker compose exec rabbit2 rabbitmqctl join_cluster rabbit@rabbit1
docker compose exec rabbit2 rabbitmqctl start_app

docker compose exec rabbit3 rabbitmqctl stop_app
docker compose exec rabbit3 rabbitmqctl reset
docker compose exec rabbit3 rabbitmqctl join_cluster rabbit@rabbit1
docker compose exec rabbit3 rabbitmqctl start_app

docker compose exec rabbit1 rabbitmqctl cluster_status
```

`cluster_status` lists 3 disk nodes and 3 running nodes. The UI (any node) shows all three under **Overview → Nodes**.

Automatic alternative: **peer discovery** in `rabbitmq.conf`:

```ini
cluster_formation.peer_discovery_backend = classic_config
cluster_formation.classic_config.nodes.1 = rabbit@rabbit1
cluster_formation.classic_config.nodes.2 = rabbit@rabbit2
cluster_formation.classic_config.nodes.3 = rabbit@rabbit3
```

(Other backends: DNS, AWS, Kubernetes, Consul, etcd. On Kubernetes, use the **RabbitMQ Cluster Operator**.)

### 36.3 See replication in action

1. Declare a quorum queue `ha_test` on rabbit1 (UI → Add queue → Type: Quorum). Its page shows **Leader** and **Members** on all 3 nodes.
2. Publish 10 persistent messages.
3. `docker compose stop rabbit1` → open http://localhost:15673 (rabbit2). The queue elected a new leader and still has 10 messages.
4. `docker compose start rabbit1` — it rejoins and catches up.

Quorum queue maintenance:

```powershell
docker compose exec rabbit1 rabbitmq-queues quorum_status ha_test -p /
docker compose exec rabbit1 rabbitmq-queues add_member ha_test rabbit@rabbit3 -p /
docker compose exec rabbit1 rabbitmq-queues grow rabbit@rabbit3 all       # add replicas on a new node
docker compose exec rabbit1 rabbitmq-queues rebalance quorum              # spread leaders evenly
docker compose exec rabbit1 rabbitmq-queues check_if_node_is_quorum_critical
```

### 36.4 Connecting clients to a cluster

pika accepts a list of parameters and tries them in order:

```python
nodes = [
    pika.URLParameters("amqp://blog:blogpass@localhost:5672/blog"),
    pika.URLParameters("amqp://blog:blogpass@localhost:5673/blog"),
    pika.URLParameters("amqp://blog:blogpass@localhost:5674/blog"),
]
connection = pika.BlockingConnection(nodes)
```

Celery: `CELERY_BROKER_URL = "amqp://u:p@host1//;amqp://u:p@host2//;amqp://u:p@host3//"` (semicolon-separated failover list).

### 36.5 Removing a node / maintenance

```powershell
# Gracefully move leaders and clients away before maintenance
rabbitmq-upgrade drain
# ...do the work, then
rabbitmq-upgrade revive

# Remove a dead node permanently (run on a remaining node)
rabbitmqctl forget_cluster_node rabbit@rabbit3
```

Rolling upgrade: upgrade one node at a time, wait until `rabbitmq-upgrade await_online_quorum_plus_one` passes, continue; finally `rabbitmqctl enable_feature_flag all`.

---

## Step 37: Shovel and Federation

Both connect **separate** brokers (different data centres, or migrating from an old broker).

| Feature      | **Shovel**                                                     | **Federation**                                                     |
|--------------|----------------------------------------------------------------|--------------------------------------------------------------------|
| What         | Consumes from a source queue and republishes to a destination | Links exchanges/queues so messages flow from "upstream" brokers     |
| Direction    | Point-to-point, explicit                                       | Declarative; exchange federation replicates matching messages       |
| Typical use  | Move messages between brokers/vhosts, migration, replay DLQs   | Multi-region event distribution, loose coupling of brokers          |

### 37.1 Dynamic shovel

```powershell
docker exec rabbitmq rabbitmq-plugins enable rabbitmq_shovel rabbitmq_shovel_management

docker exec rabbitmq rabbitmqctl set_parameter -p blog shovel move-dead-letters '{\"src-protocol\":\"amqp091\",\"src-uri\":\"amqp:///blog\",\"src-queue\":\"tasks.dead\",\"dest-protocol\":\"amqp091\",\"dest-uri\":\"amqp:///blog\",\"dest-exchange\":\"tasks\",\"dest-exchange-key\":\"task\",\"src-delete-after\":\"queue-length\"}'
```

This moves everything currently in `tasks.dead` back to exchange `tasks` with key `task`, then deletes itself (`src-delete-after: queue-length`). `amqp:///blog` means "this broker, vhost blog". Status: `rabbitmqctl shovel_status` or **Admin → Shovel Status**.

Delete: `rabbitmqctl clear_parameter -p blog shovel move-dead-letters`.

### 37.2 Federation (overview)

```powershell
rabbitmq-plugins enable rabbitmq_federation rabbitmq_federation_management
# On the DOWNSTREAM broker: define where to pull from
rabbitmqctl set_parameter federation-upstream eu-broker '{\"uri\":\"amqp://user:pass@eu-host:5672\"}'
# Federate exchanges whose names start with "events."
rabbitmqctl set_policy --apply-to exchanges federate-events "^events\." '{\"federation-upstream-set\":\"all\"}'
```

Messages published to `events.*` on the EU broker are now also routed on the downstream broker.

---

## Step 38: Best Practices Checklist

**Design**

- [ ] Producers publish to **exchanges** (named, durable), never rely on queues existing.
- [ ] Use a **topic exchange** for events with `<app>.<entity>.<action>` routing keys.
- [ ] Each consuming service owns its **own named durable queue** (`<service>.<purpose>`).
- [ ] Messages are small JSON with `message_id`, `type`, `schema_version`; big data goes in storage.
- [ ] Consumers are **idempotent** (at-least-once delivery).

**Reliability**

- [ ] Durable exchanges/queues + persistent messages.
- [ ] **Quorum queues** for important data (3-node cluster in production).
- [ ] **Publisher confirms** on; handle `UnroutableError`/`NackError`.
- [ ] Manual acks after work; never `auto_ack` for important work.
- [ ] Every important queue has a **DLX** and a dead-letter queue that is monitored.
- [ ] Poison messages limited by `delivery-limit` or a retry counter.
- [ ] Publish from Django inside `transaction.on_commit` (or an outbox).

**Performance**

- [ ] Long-lived connections; one channel per thread; don't open a connection per message.
- [ ] Set **prefetch** (never unlimited).
- [ ] Keep queues **short** — a queue with millions of messages is slow and uses resources. Scale consumers instead.
- [ ] Separate connections for publishing and consuming in busy services.
- [ ] Avoid huge numbers of queues/bindings created dynamically without expiry (`x-expires`).

**Operations**

- [ ] Configuration via **policies** and **definitions** in Git, not hard-coded arguments.
- [ ] `disk_free_limit` raised; memory watermark reviewed.
- [ ] Monitoring: queue depth, consumers, unacked, alarms, connections (Prometheus).
- [ ] `guest` deleted, per-app users, TLS, UI not public.
- [ ] Heartbeats on; clients reconnect automatically and re-declare topology.

---

## Step 39: Troubleshooting

| Symptom / error                                                                 | Cause                                                              | Fix                                                                                  |
|---------------------------------------------------------------------------------|--------------------------------------------------------------------|--------------------------------------------------------------------------------------|
| `AMQPConnectionError` / `ConnectionRefusedError` (10061)                        | Broker not running / wrong host or port                            | `docker ps`, `rabbitmq-diagnostics ping`, check `-p 5672:5672`                       |
| `ProbableAuthenticationError` / `ACCESS_REFUSED - Login was refused`            | Wrong user/password, or `guest` from non-localhost                 | Check credentials; create a proper user (Step 5)                                     |
| `ProbableAccessDeniedError` / `NOT_ALLOWED - vhost blog not found` / `access to vhost refused` | vhost missing or user has no permission on it        | `add_vhost`, `set_permissions -p blog ...`                                           |
| `ChannelClosedByBroker: (404, "NOT_FOUND - no exchange 'x'")`                   | Publishing to an exchange that doesn't exist                       | Declare it first                                                                     |
| `(404, "NOT_FOUND - no queue 'x'")`                                             | Consuming/passive-declaring a missing queue                        | Declare it                                                                           |
| `(406, "PRECONDITION_FAILED - inequivalent arg ...")`                           | Re-declaring a queue/exchange with different settings              | Use the same settings, delete and recreate, or use policies                          |
| `(406, "PRECONDITION_FAILED - unknown delivery tag")`                           | Ack on wrong channel, double ack, or ack after `auto_ack=True`     | Ack once, on the receiving channel                                                    |
| `(406, "PRECONDITION_FAILED - delivery acknowledgement timed out")`             | Consumer held a message longer than `consumer_timeout`             | Ack faster, raise `consumer_timeout`, avoid long Celery ETAs                         |
| `(403, "ACCESS_REFUSED - ... resource locked")`                                 | Exclusive queue used from another connection                       | Use a non-exclusive queue                                                            |
| `StreamLostError` / `ConnectionResetError` / log "missed heartbeats from client" | Callback blocked heartbeats; network dropped idle connection       | Threads + `add_callback_threadsafe` (Step 26); keep heartbeats on; reconnect          |
| Publisher hangs                                                                 | Memory/disk alarm — connection `blocked`                           | `check_local_alarms`; drain queues, add disk/RAM; set `blocked_connection_timeout`    |
| Messages disappear after restart                                                | Queue not durable or messages not persistent                       | `durable=True` + `delivery_mode=2`                                                    |
| Messages vanish on publish                                                      | No queue bound for that routing key                                | Check bindings; use `mandatory=True` or an alternate exchange                         |
| Message processed twice                                                         | Consumer crashed/reconnected after work but before ack             | Make consumers idempotent                                                            |
| Unacked count stuck high                                                        | Consumer forgot to ack or is hung                                  | Fix code; restart consumer (messages are requeued)                                   |
| One worker busy, others idle                                                    | No prefetch → all messages pushed to the first consumer            | `basic_qos(prefetch_count=1..N)`                                                     |
| Same message redelivered forever, high CPU                                      | `nack(requeue=True)` on a message that always fails                | Dead-letter it; retry counter; quorum `delivery-limit`                               |
| Docker: users/queues gone after recreating the container                        | No volume, or hostname changed (data dir is per node name)         | Fixed `--hostname` + volume                                                          |
| Windows: `rabbitmqctl` "authentication failed" / "unable to connect to node"    | Erlang cookie mismatch between service and your user               | Copy `C:\Windows\System32\config\systemprofile\.erlang.cookie` to `%USERPROFILE%`    |
| `set_policy` / JSON args fail in PowerShell                                      | PowerShell strips double quotes                                    | Use `'{\"key\":\"value\"}'`                                                          |
| `pika` errors when used from several threads                                    | pika isn't thread-safe                                             | One connection per thread, or `add_callback_threadsafe`                              |

---

## Step 40: RabbitMQ vs Redis vs Kafka

| Aspect                     | **RabbitMQ**                                        | **Redis** (lists/streams as broker)          | **Kafka**                                     |
|----------------------------|-----------------------------------------------------|----------------------------------------------|-----------------------------------------------|
| Model                      | Smart broker: exchanges route to queues; push       | In-memory data store used as a queue         | Distributed append-only log; consumers pull   |
| Routing                    | Rich (direct, topic, headers, fanout)               | Basic                                        | By topic/partition                            |
| Message after consume      | Deleted (queues) / kept (streams)                   | Deleted (lists) / kept (streams)             | Kept for retention period; replay any time    |
| Durability                 | Disk + replication (quorum)                         | Optional persistence (AOF/RDB), less strict  | Disk + replication, very strong               |
| Throughput                 | Tens of thousands msg/s per node (more with streams)| Very high, small messages                    | Millions msg/s (cluster)                      |
| Per-message ack, retry, DLX, TTL, priority | **Yes, built in**                   | Mostly DIY                                   | DIY (offsets)                                 |
| Ordering                   | Per queue                                           | Per list/stream                              | Per partition                                 |
| Ops complexity             | Medium                                              | Low                                          | High                                          |
| Best for                   | Task queues, microservice messaging, complex routing, reliable delivery | Simple Celery setups, caching + light queuing | Event streaming, analytics pipelines, event sourcing, huge volume |

Celery works with both RabbitMQ (most feature-complete broker for Celery) and Redis.

---

## Step 41: Cheat Sheet

### Docker

```powershell
docker run -d --name rabbitmq --hostname my-rabbit -p 5672:5672 -p 15672:15672 -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=admin123 -v rabbitmq_data:/var/lib/rabbitmq rabbitmq:4-management
docker exec rabbitmq rabbitmqctl <command>
docker logs -f rabbitmq
```

UI: http://localhost:15672 — AMQP: `amqp://user:pass@localhost:5672/vhost`

### Admin

```powershell
rabbitmqctl add_vhost blog
rabbitmqctl add_user blog blogpass
rabbitmqctl set_permissions -p blog blog ".*" ".*" ".*"
rabbitmqctl set_user_tags admin administrator
rabbitmqctl list_queues -p blog name messages_ready messages_unacknowledged consumers
rabbitmqctl list_exchanges -p blog
rabbitmqctl list_bindings -p blog
rabbitmqctl list_connections
rabbitmqctl purge_queue -p blog <queue>
rabbitmqctl delete_queue -p blog <queue>
rabbitmqctl set_policy -p blog <name> "<regex>" '<json>' --apply-to queues
rabbitmqctl export_definitions /tmp/defs.json
rabbitmq-diagnostics ping | status | check_local_alarms | log_tail -N 50
rabbitmq-plugins enable rabbitmq_management rabbitmq_prometheus
```

### pika

```python
import pika

conn = pika.BlockingConnection(pika.URLParameters("amqp://blog:blogpass@localhost:5672/blog"))
ch = conn.channel()

ch.exchange_declare(exchange="ex", exchange_type="topic", durable=True)       # direct|fanout|topic|headers
ch.queue_declare(queue="q", durable=True, arguments={"x-queue-type": "quorum"})
ch.queue_bind(queue="q", exchange="ex", routing_key="blog.#")

ch.confirm_delivery()
ch.basic_publish(exchange="ex", routing_key="blog.post.created", body=b"{}",
                 properties=pika.BasicProperties(content_type="application/json",
                                                 delivery_mode=pika.DeliveryMode.Persistent),
                 mandatory=True)

def cb(ch, method, props, body):
    ch.basic_ack(method.delivery_tag)            # or basic_nack(tag, requeue=False)

ch.basic_qos(prefetch_count=10)
ch.basic_consume(queue="q", on_message_callback=cb)
ch.start_consuming()
```

### Patterns at a glance

| Need                                  | Use                                                     |
|---------------------------------------|---------------------------------------------------------|
| Background jobs, many workers         | One durable queue + manual ack + prefetch (Step 9)      |
| Broadcast to all                      | Fanout exchange (Step 10)                               |
| Route by type                         | Direct exchange (Step 11)                               |
| Route by pattern                      | Topic exchange (Step 12)                                |
| Request/response                      | RPC with `reply_to` + `correlation_id` (Step 14)        |
| Don't lose messages                   | Durable + persistent + confirms + quorum + manual ack   |
| Handle failures                       | DLX + retry queue with TTL + parking lot (Step 19)      |
| Delay                                 | TTL + DLX, or delayed-message plugin (Step 20)          |
| Urgent first                          | Priority queue or separate queues (Step 21)             |
| Strict order with failover            | Single active consumer (Step 21)                        |
| Replay history / many readers         | Stream (Step 23)                                        |
| Python background tasks in Django     | Celery with RabbitMQ broker (Step 28)                   |
| Events to other services from Django  | pika publisher + `on_commit` + consumer command (29–30) |

---

**Next steps:** read the official tutorials (https://www.rabbitmq.com/tutorials) and docs (https://www.rabbitmq.com/docs) — especially *Reliability Guide*, *Production Checklist*, *Quorum Queues* and *Consumer Acknowledgements and Publisher Confirms*.
