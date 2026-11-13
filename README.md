# KandyyAuth

Secure authentication for offline-mode ("cracked") Minecraft servers, with real Mojang premium auto-login.

**Created and developed by Tyxen — Satyam “Senzor”.**

KandyyAuth gives cracked players Argon2id-protected password accounts while genuine premium players are verified against Mojang during the login phase and never type a password. Everything runs on your server: no MySQL, no Redis, no external API keys.

---

## Highlights

- **Real premium auto-login** — ProtocolLib-driven RSA/AES login handshake validated by `sessionserver.mojang.com`.
- **Argon2id password hashing** — 19 MiB / 2 iterations / 1 lane by default, random 16-byte salt per password, constant-time comparison.
- **Local SQLite storage** — one file at `plugins/KandyyAuth/auth.db`, WAL mode, prepared statements only.
- **Brute-force protection** — per-connection limits plus a per-IP moving-window throttle with temporary locks and a registration flood limit.
- **Lobby lockdown** — unauthenticated players are frozen, blinded, and blocked from chat, commands, interactions, inventories, item pickup/drop, damage and portals.
- **Graceful fallback** — if ProtocolLib is missing or a server build changes the login packet layout, premium verification disables itself and everyone uses `/login` or `/register`.
- **Admin tooling** — `/auth reload | unregister | forcelogin | info`.
- **Developer API** — `PlayerAuthenticatedEvent` for other plugins.

---

## Supported versions

| Requirement | Details |
| --- | --- |
| Server | Paper 1.21.x through 26.x+ (Paper forks such as Purpur should work) |
| Java | Java 21+ — 26.x servers require Java 25 |
| Mode | `online-mode=false` in `server.properties` |
| ProtocolLib | Optional; required only for premium auto-login |
| Conflicts | Do not run AuthMe, FastLogin or nLogin at the same time |

The jar is compiled against Paper `1.21.11` on Java 21 so the same file loads on both the 1.21.x line (Java 21) and the newer 26.x line (Java 25).

---

## Download

Grab the latest build from **[Releases](https://github.com/Kandyy-Auth/Kandyy/releases/latest)** — or use `KandyyAuth-1.0.0.jar` from the root of this repository.

---

## Installation

1. **Set offline mode** in `server.properties`:
   ```properties
   online-mode=false
   ```
2. **Install ProtocolLib** (recommended) into `plugins/` for premium auto-login. Without it, KandyyAuth still loads and uses passwords.
3. **Remove conflicting auth plugins** — AuthMe, FastLogin, nLogin, and similar.
4. **Drop the jar** into `plugins/`:
   ```
   plugins/ProtocolLib.jar
   plugins/KandyyAuth-1.0.0.jar
   ```
5. **Restart the server** (do not use `/reload`).
6. On first start the plugin creates:
   ```
   plugins/KandyyAuth/
     config.yml
     messages.yml
     auth.db
   ```
7. Join with a cracked client — you should be frozen and prompted to `/register`. Join with a premium account — you should be logged in automatically.

---

## Commands

| Command | Description | Permission |
| --- | --- | --- |
| `/login <password>` | Log in with your password. Aliases: `/l`, `/lg` | none |
| `/register <password> <password>` | Create a password account. Alias: `/reg` | none |
| `/changepassword <old> <new>` | Change your password. Aliases: `/changepass`, `/cp` | none |
| `/auth reload` | Reload `config.yml` and `messages.yml` | `kandyy.auth.admin` |
| `/auth unregister <player>` | Delete a stored account | `kandyy.auth.admin` |
| `/auth forcelogin <player>` | Force-login an online player | `kandyy.auth.admin` |
| `/auth info <player>` | Show stored account metadata | `kandyy.auth.admin` |

Aliases for `/auth`: `/kandyy`, `/authadmin`.

## Permissions

| Node | Description | Default |
| --- | --- | --- |
| `kandyy.auth.admin` | Access to `/auth` administration commands | `op` |

---

## Configuration

Everything lives in `plugins/KandyyAuth/config.yml`. The generated file is fully commented; the important groups are:

```yaml
settings:
  auth-enabled: true
  auth-timeout-seconds: 60
  min-password-length: 8
  max-password-length: 64
  max-login-attempts: 5
  freeze-movement: true
  blind-unauthorized: true
  teleport-to-spawn: false

security:
  ip-max-login-attempts: 10
  ip-login-window-seconds: 60
  ip-lock-seconds: 300
  ip-max-registrations: 5
  log-auth-actions: true

premium:
  enabled: true
  fail-closed: true
  verify-ip: false
  cache-minutes: 360

hashing:
  memory-kib: 19456
  iterations: 2
  parallelism: 1
  hash-length: 32
  salt-length: 16
  threads: 2
  max-concurrent: 4

storage:
  file: auth.db
```

Notes:

- Keep `premium.fail-closed: true`. It stops an impostor from registering a premium name while Mojang is unreachable.
- Leave `premium.verify-ip: false` behind proxies unless IP forwarding is configured correctly.
- `/auth reload` applies messages and most security settings; hashing parameters and the database file need a restart.
- Messages in `messages.yml` support legacy `&` colour codes and `{placeholders}`.

---

## Premium verification

1. The plugin looks up the connecting username with Mojang.
2. If the name is premium, the login is restarted with a standard Minecraft encryption request.
3. The client encrypts a shared secret with the server's RSA key.
4. The plugin validates the server hash with `sessionserver.mojang.com/session/minecraft/hasJoined`.
5. Only a genuine Mojang session can pass, so cracked clients cannot impersonate premium names.

If the server cannot reach Mojang, the login is refused while `fail-closed` is enabled. If ProtocolLib is absent or the packet layout is unsupported, premium verification disables itself and password login remains available.

Required outbound access: `api.mojang.com` and `sessionserver.mojang.com` over HTTPS (443).

---

## Security notes

- Passwords are never logged.
- All SQL uses prepared statements.
- Account names are canonicalized to lower case so case variants cannot duplicate an account.
- The hashing pool is bounded so a login flood cannot exhaust server memory.
- Unauthenticated players cannot chat, run commands, move, interact, open inventories, pick up items, take damage or use portals.
- The SQLite database contains every account — back up `plugins/KandyyAuth/auth.db` regularly.

---

## Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| Players are never prompted | Confirm `online-mode=false` and `settings.auth-enabled: true`. |
| Premium players are asked for a password | ProtocolLib missing or unsupported build; check the startup log. Password login is the intended fallback. |
| “Could not verify this account with Mojang” | Server cannot reach Mojang. Allow outbound HTTPS or set `premium.enabled: false`. |
| “Unsupported API version” | Server is older than 1.21; KandyyAuth targets Paper 1.21.x+. |

---

## License

Proprietary. All rights reserved by Tyxen. Redistribution or modification requires permission from the author.

---

**Author:** Tyxen — Satyam “Senzor”  
**Plugin:** KandyyAuth v1.0.0
