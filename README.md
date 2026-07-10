# EconomyBridge

An Endstone (Python) plugin that provides **two-way synchronization**:

```
JWEconomy (Coin)  <--->  uMoney (Money)
```

Whichever side changed since the last confirmed sync gets pushed to the
other side — including decreases (e.g. shop purchases). If **both** sides
change independently within the same cycle (a genuine conflict), their
deltas are merged additively rather than picking one arbitrary winner. See
"Conflict Resolution Rule" below for details.

## Why Python, not Java?

Endstone **does not support Java plugins** — only Python and C++. This project
is built as an official Endstone Python plugin, while still following the
SOLID architecture (provider abstraction, single-responsibility classes)
requested in the original spec.

## Project Structure

```
economy-bridge/
├── pyproject.toml
├── README.md
└── src/endstone_economy_bridge/
    ├── __init__.py
    ├── plugin.py                     # EconomyBridgePlugin (main class)
    ├── config_manager.py             # ConfigManager
    ├── bridge_manager.py             # BridgeManager (sync logic)
    ├── sync_task.py                  # SyncTask (periodic scheduler)
    ├── sync_state_store.py           # SyncStateStore (last-synced baseline per player)
    ├── economy_provider.py           # Provider abstraction
    ├── resources/config.yml          # Bundled default config
    └── providers/
        ├── jweconomy_provider.py     # JWEconomy adapter (async)
        └── umoney_provider.py        # uMoney adapter (sync)
```

## Class Responsibilities

| Class | Responsibility |
|---|---|
| `EconomyBridgePlugin` | Plugin entry point: wires up components, handles lifecycle (`on_load`/`on_enable`/`on_disable`), and the `/ecobridge` command. |
| `ConfigManager` | Reads/writes `config.yml` (instead of Endstone's built-in `config.toml`), with safe defaults. |
| `EconomyProvider` (`AsyncEconomyProvider` / `SyncEconomyProvider`) | Abstract contracts so new economy providers can be added without touching `BridgeManager`. Split by calling convention (async vs sync), not by direction — both sides now read AND write. |
| `JWEconomyProvider` | Adapter for JWEconomy. Its API is **async**, called through JWEconomy's own `run_async()`. Read & write. |
| `UMoneyProvider` | Adapter for uMoney. Its API is plain sync, read & write (`api_get_player_money` / `api_reset_player_money`). |
| `BridgeManager` | Core logic: listens to `PlayerJoinEvent`, compares balances against the last-synced baseline, pushes whichever side legitimately changed, and additively merges genuine simultaneous conflicts (see below). |
| `SyncStateStore` | Persists the last confirmed-synced balance per player (`sync_state.json` in the data folder), so a real change (including a decrease, e.g. a shop purchase) can be told apart from a genuine conflict. |
| `SyncTask` | Schedules `BridgeManager.sync_all_online()` periodically via `server.scheduler`, interval read from `config.yml`. |

## Conflict Resolution Rule (Two-Way Sync)

Since neither JWEconomy nor uMoney expose a balance-change event, EconomyBridge
can't directly observe "who changed first". Instead, it keeps a small local
baseline (`SyncStateStore`) of the last balance both sides were confirmed to
agree on, and compares against it every cycle:

| Situation | Meaning | Action |
|---|---|---|
| Coin differs from baseline, Money doesn't | Coin legitimately changed (e.g. a command, shop purchase, or reward — up **or down**) | Push Coin → uMoney |
| Money differs from baseline, Coin doesn't | Money legitimately changed | Push Money → JWEconomy |
| **Both** differ from baseline | Genuine conflict — both sides changed independently within the same cycle | **Merge additively**: `final = baseline + Δcoin + Δmoney`, floored at 0 |
| Neither differs | Already in sync | No write |

> ⚠️ **Why not just "highest balance wins" for conflicts?**
> A naive tie-break that just picks the higher value discards whichever
> change happened on the *other* side. E.g. baseline 1000, a shop
> purchase cuts Coin to 800 while a quest reward bumps Money to 1200 in
> the same cycle — "highest wins" would settle on 1200 and silently erase
> the purchase. Treating both changes as independent, legitimate
> transactions and summing their deltas onto the baseline instead
> (`1000 + (-200) + (+200) = 1000`) keeps the effect of **both**
> transactions — the same idea as merging two concurrent edits to a
> shared counter from a common ancestor. The merged result is floored at
> 0 so two large simultaneous deductions can never push a balance
> negative.

The very first time a player is reconciled (no baseline recorded yet, e.g.
right after installing the plugin), there's no baseline to compute a delta
against, so that one time only falls back to "highest balance wins" purely
to establish the initial baseline.

## Why Isn't There a Real-Time "Coin Changed" Event?

Based on the current source code of JWEconomy (`junggamyeon/JWEconomy`), the
plugin **does not expose an event** when a balance changes (whether via
commands, shop purchases, quest rewards, or battle rewards). Because of that,
the realistic and robust mechanism is:

1. **Sync on join** (`sync-on-join: true`) — closes the gap when a player
   first joins.
2. **Periodic scheduler** (`sync-interval`, in seconds) — covers every other
   case (add/remove coin commands, shop, rewards, etc.) within a maximum
   delay equal to the interval.

If JWEconomy adds a balance-change event in a future release, you can simply
add a new `@event_handler` in `BridgeManager` that calls `sync_player()` —
no other component needs to change.

## Choosing an Optimal `sync-interval`

There's no single number that fits every server — it's a trade-off between
load on JWEconomy's database and how quickly balance changes show up in
uMoney.

| Server size (concurrent players) | Suggested `sync-interval` |
|---|---|
| Small (< 20 players) | 10–15 seconds |
| Medium (20–100 players) | 20–30 seconds |
| Large (100+ players) | 45–60 seconds (default) |

- Too small → the scheduler calls `fetch_balance()` for **every** online
  player on each cycle, putting load on JWEconomy's database even when most
  balances haven't changed (the code already skips the *write* when equal,
  but the *read* still happens).
- Too large → balance changes (shop, rewards) can take up to that many
  seconds to show up in uMoney.
- If your own shop/quest/battle plugin exposes an event (e.g.
  `ShopPurchaseEvent`), it's better to add a dedicated listener in
  `BridgeManager` that calls `sync_player()` when that event fires — so a
  larger `sync-interval` stays safe to use purely as a backup, while big
  transactions still feel instant.

## Commands

- `/ecobridge sync` — force-sync all online players right now.
- `/ecobridge reload` — reload `config.yml` and restart the scheduler.
- `/ecobridge status` — show the current configuration status.

Registered usage is `/ecobridge (sync|reload|status)<action: EcoBridgeAction>`,
following Endstone's **user-defined enum type** syntax: the list of choices
goes inside `(...)`, followed by the parameter name and type inside `<...>`.

Permission: `economy_bridge.command.ecobridge` (default: **op**).

## Installation

```bash
pip install build
cd economy-bridge
python -m build --wheel
```

Copy the resulting `.whl` file (in the `dist/` folder) into your Endstone
server's `plugins/` folder, then start the server. `config.yml` will be
created automatically at `plugins/economy_bridge/config.yml` the first time
the plugin is enabled, alongside a `sync_state.json` that stores each
player's last-synced baseline (see "Conflict Resolution Rule" above). You
generally shouldn't need to touch `sync_state.json` by hand; deleting it
just resets every player's baseline, causing the next reconciliation for
each of them to fall back to "highest balance wins" once.

## Fix History

The following issues previously caused the plugin to fail loading, and have
already been fixed in the current code — noted here for reference in case
you reuse this pattern in another plugin:

1. **`permissions` must be a `dict`, not a `list`** — `plugin_loader.py`
   calls `.items()` on this attribute, just like `commands`.
2. **Don't use `from __future__ import annotations` in any file containing
   `@event_handler`** — it turns all type hints (including the event type)
   into plain strings at runtime, causing Endstone to fail to recognize
   `PlayerJoinEvent` via signature introspection.
3. **Enum syntax in command `usages`**: `(value1|value2)<name: TypeName>`
   — the list of choices comes **first** inside `()`, followed by the
   parameter name and type inside `<...>` or `[...]` (optional).
4. **Naive "highest balance always wins" breaks spending** — an earlier
   version of the two-way sync compared Coin vs Money directly with no
   memory of past state, which silently reverted any purchase/deduction
   on either side within one sync cycle. Fixed by introducing
   `SyncStateStore` (a persisted last-synced baseline per player) so real
   changes — including decreases — are told apart from genuine conflicts.
5. **"Highest wins" as the conflict tie-break also discards one side's
   change** — even with a baseline in place, picking the higher balance
   when both sides genuinely changed still erases whichever change was
   smaller (e.g. a purchase gets wiped out by a simultaneous reward).
   Replaced with an additive merge of both deltas
   (`baseline + Δcoin + Δmoney`, floored at 0), which preserves the
   effect of both transactions instead of discarding one.

## Parts You MUST Re-Verify on Your Own Server

Since third-party plugin APIs can change between versions, double-check the
following (marked with `# TODO` in the code):

1. `JWEconomyProvider.PLUGIN_NAME` — JWEconomy's registered name in
   `plugin_manager`.
2. `JWEconomyProvider`: how the API is obtained (`get_api()`) and the
   `get_balance(uuid, currency)` signature.
3. `UMoneyProvider.PLUGIN_NAME` — uMoney's registered name in
   `plugin_manager`.
4. `UMoneyProvider`: the `api_get_player_money` / `api_reset_player_money`
   method names.
