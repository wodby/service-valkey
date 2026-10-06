# Valkey on Wodby

What Wodby sets up for this Valkey service. Check it before adding connection settings or a Valkey configuration file to an application.

## How applications reach it

- Host: the name of this app service inside the environment. Port: `6379`. The protocol is the Redis protocol; Redis clients work unchanged.
- A password is always required. Wodby generates it (token `password`) and the server starts with `requirepass`.
- A service linked to this one receives the host, port and password as environment variables. The names are chosen by the linking service and usually keep the Redis prefix, for example `REDIS_HOST`, `REDIS_PORT` and `REDIS_PASSWORD`, or one `REDIS_URL`. Read those variables in code; do not hardcode the host or copy the password into the repository.
- Inside this container the password is in `VALKEY_PASSWORD`.

## Generated configuration

On every start the container renders `/etc/valkey.conf` from its environment variables and starts `valkey-server` with it. The file is rewritten on each start: never edit it. Configuration is changed with environment variables on this service, and applies with the next deployment. The variables of this service start with `VALKEY_`, not `REDIS_`.

| Variable | Effect | Image default |
| --- | --- | --- |
| `VALKEY_MAXMEMORY` | `maxmemory` | `128m` |
| `VALKEY_MAXMEMORY_POLICY` | `maxmemory-policy` | `allkeys-lru` |
| `VALKEY_DATABASES` | number of databases | `16` |
| `VALKEY_TIMEOUT` | idle client timeout, seconds | `300` |
| `VALKEY_SAVE_TO_DISK` | any value turns persistence on | set by Wodby when the data volume exists |
| `VALKEY_APPENDONLY`, `VALKEY_APPENDFSYNC`, `VALKEY_SAVES` | AOF and RDB settings, used only with persistence on | `yes`, `everysec`, `900:1/300:10/60:10000` |
| `VALKEY_NOTIFY_KEYSPACE_EVENTS` | keyspace notifications | empty |

## Eviction and persistence

- With the defaults this is a cache: once `VALKEY_MAXMEMORY` is reached, any key can be evicted (`allkeys-lru`), including keys without a TTL.
- A job queue, a session store or any data that must not disappear needs `VALKEY_MAXMEMORY_POLICY` set to `noeviction` and enough `VALKEY_MAXMEMORY`. Using the default cache settings as a queue store is the usual mistake: jobs are dropped silently under memory pressure.
- The `data` volume is optional. Without it nothing is written to disk (`save ""`, no AOF) and every restart or deployment starts empty. With it, Wodby turns persistence on and data is kept in `/data` (RDB snapshots plus AOF).
- The manifest declares no backups, imports or scheduled jobs for this service.

## Check the result

From this service's container:

- `valkey-cli -a "$VALKEY_PASSWORD" ping` answers `PONG`.
- `valkey-cli -a "$VALKEY_PASSWORD" config get maxmemory-policy` and `config get appendonly` show the eviction policy and whether persistence is on.
- `valkey-cli -a "$VALKEY_PASSWORD" info keyspace` shows which databases hold keys; `info stats` has `evicted_keys`.
