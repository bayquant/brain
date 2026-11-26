---
tags: [redis, databases, caching, macos, cli]
---
# Redis

Redis is an in-memory key-value store used as a cache, message broker, and lightweight database. Data lives in RAM by default, with optional persistence to disk. See also [[Caching]] for where Redis fits as a distributed cache layer.

---

## INSTALL ON MAC

```bash
# Install via Homebrew
brew install redis

# Start as a background service (survives reboots, auto-restarts)
brew services start redis

# Stop the service
brew services stop redis

# Restart after a config change
brew services restart redis

# Check service status
brew services info redis
```

Config file installed at `/opt/homebrew/etc/redis.conf` (Apple Silicon) or `/usr/local/etc/redis.conf` (Intel).

---

## RUN WITHOUT A SERVICE

```bash
# Run in the foreground (Ctrl+C to stop)
redis-server

# Run with a specific config file
redis-server /opt/homebrew/etc/redis.conf

# Run on a different port
redis-server --port 6380
```

---

## DOCKER

For a Redis defined in a Compose file, such as `xpectral-trading/redis/compose.yaml`. Start Docker Desktop first, then run from the repo root. Don't also run a Homebrew Redis: both use port 6379.

```bash
# Start and wait until healthy
docker compose -f redis/compose.yaml up -d --wait

# Status, logs, restart, stop
docker compose -f redis/compose.yaml ps
docker compose -f redis/compose.yaml logs -f
docker compose -f redis/compose.yaml restart
docker compose -f redis/compose.yaml stop

# Remove the container (data is kept; adding -v deletes it)
docker compose -f redis/compose.yaml down

# Open the redis-cli prompt
docker compose -f redis/compose.yaml exec redis redis-cli

# Run a single command
docker compose -f redis/compose.yaml exec redis redis-cli PING
```

### Alias

An alias typed at the prompt lasts only until that terminal session closes. To keep it, add this line to `~/.zshrc` (use the full path to the Compose file so it works from any folder):

```bash
alias rcli='docker compose -f path/to/compose.yaml exec redis redis-cli'
```

Then reload with `source ~/.zshrc`.

```bash
rcli            # open the prompt
rcli PING       # single command
unalias rcli    # remove it from the current session
```

---

## CLI

```bash
# Connect to local instance
redis-cli

# Connect to a specific host/port
redis-cli -h <host> -p <port>

# Connect with auth
redis-cli -a <password>

# Run a single command without an interactive session
redis-cli PING
redis-cli SET foo bar

# Check the server is alive
redis-cli PING          # → PONG
```

---

## CORE COMMANDS

```
# Strings
SET key value
GET key
DEL key
EXISTS key
EXPIRE key seconds
TTL key

# Numbers
INCR key
DECR key
INCRBY key 5

# Hashes
HSET user:1 name "Alice" age 30
HGET user:1 name
HGETALL user:1

# Lists
LPUSH queue task1
RPUSH queue task2
LRANGE queue 0 -1

# Sets
SADD tags redis cache
SMEMBERS tags

# Sorted sets
ZADD leaderboard 100 alice
ZRANGE leaderboard 0 -1 WITHSCORES

# Keys
KEYS *              # avoid in production — O(N), blocks the server
SCAN 0              # non-blocking cursor-based alternative to KEYS
```

---

## PERSISTENCE

```
RDB (snapshotting) — point-in-time dump to disk on interval or SAVE command
AOF (append-only file) — logs every write; replayed on restart for durability
```

```bash
# Trigger a manual snapshot
redis-cli SAVE

# Trigger a background snapshot (non-blocking)
redis-cli BGSAVE
```

Both can be enabled together in `redis.conf` for RDB's fast restarts plus AOF's durability.

---

## INSPECT AND DEBUG

```bash
# Server info (memory, clients, stats)
redis-cli INFO

# Monitor commands in real time
redis-cli MONITOR

# Show memory usage of a key
redis-cli MEMORY USAGE <key>

# Number of keys in the current db
redis-cli DBSIZE

# Clear the current database
redis-cli FLUSHDB

# Clear all databases
redis-cli FLUSHALL
```

---

## LOGS AND CONFIG (HOMEBREW PATHS)

```bash
# Log file
tail -f /opt/homebrew/var/log/redis.log

# Data directory
/opt/homebrew/var/db/redis/
```
