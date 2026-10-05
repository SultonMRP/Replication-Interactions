# Replication-Interactions

Public notes for Minecraft addon authors working with **Buuz135 Replication** (NeoForge) next to AE2, Refined Storage, Cooking for Blockheads, Tom's Simple Storage, and the official matter bridges.

These are **collision and behavior notes**, not source from any closed addon. Use them so you do not repeat the same double-craft / ghost-stock / void-tank mistakes.

Learned against: NeoForge **1.21.1**, Replication **1.21.1-1.2.7** + Titanium, AE2, Refined Storage 2, Cooking for Blockheads, Farmers Delight, Mystical Agriculture, Applied Replicatics (`apprep`), `rep_ae2_bridge` / `repae2bridge`, `replication_rs2_bridge`, Tom's Simple Storage (`toms_storage` 1.21-2.4.2), Applied Cooking / Applied Delight / Refined Delight, `replication_matter_overflow`.

**Not this ecosystem:** GTCEu **Material Replication** and AE2 **Matter Condenser Tiers**. Different “matter” systems. Do not treat them as Replication tanks, chips, or bridges.

---

## Replication (base)

- Matter networks are **per dimension**. Elements talk through Titanium / `MatterNetwork`.
- Patterns live on **Chip Storage / `IMatterPatternHolder`**. Terminals and replicators consume **published** matter (`cachedSyncTag` / live matter compound). Do not invent leftovers by recalculating a compound the client never got.
- Identification Chamber writes a default instance at identification progress, then consumes the item when pattern completion reaches 1.0. Energy is taken **once per finished cycle**, not every tick.
- Stock chamber refuses `#replication:cant_be_scanned`. Any addon that fills a library, creative chip, or “max database” must use the **same live tag**. Honor pack KubeJS / datapack members. **Do not hardcode pack item ids.**
- `config/replication/replication.toml` → `IGNORED_RECIPE_MODS` skips **recipe id namespaces** during matter calc. If you also skip items, match **item registry namespace** against that live list. Do not bake pack mod ids into your jar.
- A **creative memory chip** must not add items that have **no published matter**. Same for any bulk-fill command. “Has a leftover compound if you recompute” is not enough. Terminal / JEI / tooltip use the published tag.
- Replicator tasks carry a **source `BlockPos`**. Delivery often goes **UP** into an inventory on top of the source machine. If you add a remote output, world **UP** should accept inserts so stock `getSource()` landing still works.
- Disintegrator has **6 internal tanks**. If fill hits a tank that looks non-empty (`isEmpty()` false) but then accepts **0**, leftover types sit in queue and progress stalls. Fake ghost amounts (for example `1e-9`) cause this. Use real tank contents. Locked typed tanks need a fill path that a stale lock predicate cannot block.
- `replication_matter_overflow` void tanks are **dual supplier + consumer**. If you rank holders, put **void / overflow last** or they eat incoming matter before real tanks. Heuristic: class or name `MatterVoid` / `VoidTank` / `VoidMatter`, or namespace/path contains overflow or void.
- Sensible holder fill order when several exist on one network: **virtual / AppRep cells first → high-cap typed tanks → stock tanks → void last**. Connector `getPriority()` still wins among virtual holders. Two tanks of the same class keep their own tank priority. Stock extract among stock tanks is still low-first.
- AE2 / RS bridges that only **fake matter as items** are not Replication holders. Their AE2 / RS priority must not reorder real matter tanks.
- AppRep cells are **not** the same cell type as other matter-cell mods. AE2 world-tank index is typically `instanceof MatterTank`. A typed / locked tank should extend that if you want it indexed.

---

## Official AE2 / RS bridges vs a custom fallback

Known official / companion mods:

- **Applied Replicatics** (`apprep`) — connector / `ReplicationConnectorLogicHost`. `pushPattern` posts a Replication task. Cells are holder tanks.
- **AE2 official bridge** — `repae2bridge`, `rep_ae2_bridge`, class names like `RepAE2Bridge` / `ReplicationConnector`.
- **RS official bridge** — `replication_rs2_bridge`, class names like `RepRS2Bridge`.

### Collision: double-published patterns

If a custom block **also** publishes Chip Storage patterns into AE2 / RS **and** an official bridge is on the **same `MatterNetwork`**, AE2 / RS will see the same outputs twice. Crafts double, or two machines take the same job.

**“Is the jar loaded?” is the wrong skip.** Official jars on disk (or in the instance, unused) must not disable a fallback on a network that has **no** official machine. Skip **per network**: walk that pattern holder’s `MatterNetwork` public lists, plus Collection / Map fields on the network / NetworkManager (pipes, Titanium graph), plus about **2-hop** `NetworkManager.getElement` neighbors.

Why the extra walk: official AE2 / RS block entities often **extend `ReplicationMachine`** but implement **none** of the holder / chip interfaces, so they **vanish from public holder lists**. You will miss them unless you look at pipes / graph / neighbors.

Detect by block namespace (`apprep`, `repae2bridge`, `rep_ae2_bridge`, `replication_rs2_bridge`), class name (`RepAE2Bridge`, `ReplicationConnector`, `RepRS2Bridge`), plus path heuristics. Cache briefly (about 20 ticks).

### Fallback pattern shape

When a fallback publishes Chip Storage into AE2 / RS so terminals can request Replication items:

- AE2 fallback patterns are often **zero-input** `IPatternDetails`. `pushPattern` should post a `ReplicationTask` sourced at the bound machine.
- RS fallback patterns often use a **bookkeeping token** (`minecraft:structure_void`) plus EXTERNAL parent storage.
- Official matter-input bridges look like “real” patterns but their inputs are **matter**, not kitchen items.

AE2 `isCraftable` / craftable lists **include** those fallback faces. If a kitchen or pot uses `isCraftable` to pick an ingredient, **filter**: a **real pattern** has at least one **normal item** input. Empty inputs, structure-void bookkeeping, or official matter-input = Replication face, not an ME / RS craft.

### Queue lock (double craft from two faces)

If AE2 `pushPattern`, RS `accept`, **and** a kitchen / Q path can all post a `ReplicationTask` for the same pattern holder, they will collide.

Hold a **per-holder lock** (AE2 | RS | holder) with a short grace (about 200 ticks). If one face holds the lock, the others wait. After a holder-side Q posts, take the holder face so a later `pushPattern` cannot double-post.

`isBusy` on an AE2 / RS provider should be true if a pending Replication task’s `task.getSource()` is this machine **or** the holder queue lock is held.

### AppRep + a faster replicator

AppRep `pushPattern` defaults to **SINGLE × 1**. If another machine on the network can craft batches, hold accepts for a few ticks (or until 64) and post **one SINGLE of that size**. Otherwise you get 64 one-item tasks.

AE2 `tickCraftingLogic` / co-processors can look like they pushed 64 while a batch job runs. Only spoof that for AppRep / official-bridge providers, not every AE2 crafter.

AppRep `insertReplicatorResult` uses the **stack count**. After a **partial** ME insert, do not re-insert the full original stack.

### Applied Delight ME pot

The ME Cooking Pot should **queue AE2 jobs only**. Connector / official AE2 bridge `pushPattern` does the Replication work. Do not fake Chip Storage as ME storage and do not add Replication tasks from the pot. Request an AE2-craftable **ingredient** (after filtering fallback faces), `submitJob` as the clicking player + wireless access point. Pot fills when items exist in ME.

Some AE2 bridges **accumulate** (about 100 ticks). If the pot is waiting, flush on the next tick. AppRep is already instant.

---

## Cooking for Blockheads (CFB)

- Kitchen item providers are `#cookingforblockheads:kitchen_item_providers`. A provider that is extract-only on the kitchen side is fine. Do not let **C** dump crafted food back into a remote output or an ME Kitchen Station (re-entry / ping loops).
- CFB **Q** (queue missing) and **C** (craft / consume) are different:
  - **C / consume** must extract **real stock only**. Ghost tokens = free food. Never mark a network craft as “in kitchen” unless you actually pulled the item.
  - **Q** may ask a bound Replication library, submit a **real** AE2 `submitJob` / RS `ensureTask`, or post a Replication task. **One path per missing ingredient.**
- Priority if the kitchen is missing an item: kitchen stock (skip) → **native** AE2 / RS pattern (`submitJob` / `ensureTask`) → Replication task. Native = at least one normal-item input. Chip-storage fallbacks stay on the Replication path.
- AE2 submit should be sequential calculate / submit until the job is **stored**, not fire-and-forget spam.
- While the cooking table is open, poll stock signatures (about 1s) so the overlay refreshes when AE2 / RS / Tom's stock lands.
- Dual AE2 + RS: a Combined / stacked provider that returns on first AE2 hit will **skip RS**. Flatten / fall through so both stock taps are visible. If both have a native pattern, **AE2 wins** (one path).
- **Kitchen Station** (`appliedcooking` and similar) already taps the same ME grid. If your block also exposes AE2 `getAvailableStacks` into the same kitchen, you **double-count and double-extract**. Walk CFB neighbors (including two blocks UP; Station continues through `appliedcooking`) and skip your AE2 tap when it is the **same `IGrid`**.
- Farmers Delight pot: container slot **7**, output slot **8**. Q can queue the vessel (`getOutputContainer()`); C pulls that vessel from the kitchen walk into slot 7. Do not invent the bowl.

Search / extract order that avoided “wrong inventory first”:

1. Real linked inventories (for example Tom's) — real get / pull / push.
2. The machine’s own item slots.
3. Linked AE2 (if not the same grid as a Kitchen Station).
4. Linked RS.

First match wins. C still never ghost-consumes.

---

## Tom's Simple Storage

- An **Inventory Connector** is **one shared `IItemHandler`**. AE2 storage bus + RS external storage on the same connector both see the same items. Native AE2 / RS crafts from that stock behave. Do not also ghost those stacks in a second provider.
- Cheap wireless terminal is **range-raycast only**. The linkable item is **Adv Wireless Terminal** (`toms_storage:bound_pos` / `WorldPos`).
- `#toms_storage:remote_activate` = storage_terminal + crafting_terminal.
- Menu stay-open is `StorageTerminalBlockEntity.canInteractWith` (beacon / range). A custom infinite-range remote must skip that check **only** when the player is actually bound to **that** terminal.
- Storage terminal = inventory. Crafting terminal = vanilla 2×2 / 3×3. One bind; rebind replaces.
- Tom's stock is a **real** inventory. If several taps exist, extract Tom's **first**.
- If `toms_storage` is not loaded, do not show Tom's tooltips. Same rule for AE2 / RS link tooltips when those mods are absent.

---

## Linking / remotes (behavior)

- Prefer **one bind per link type**. Rebind replaces. Store a `GlobalPos` (cross-dimension).
- Ignore third-party beacon / dim range if you already have infinite-range remotes.
- A remote output that is **not** a cable element should not `addElement` / join the matter cable graph. Borrow the bound pattern holder’s attached network for tasks / tanks.
- Do not show a “link this AE2 / RS / Tom's thing” tooltip when that mod is not installed.

---

## Practical rules for a new addon

1. Honor live `#replication:cant_be_scanned` and live `IGNORED_RECIPE_MODS`. No hardcoded pack ids.
2. Do not add library / chip rows without **published** matter.
3. If official AE2 / RS / AppRep exists **on this network**, do not also publish the same Chip Storage patterns into AE2 / RS.
4. Detect official machines via graph / pipes / neighbors, not “is the jar loaded.”
5. One craft path. Lock the pattern holder while AE2, RS, or a kitchen Q owns the job.
6. CFB C = real extract. Filter AE2 / RS `isCraftable` so fallback faces are not treated as ME recipes.
7. Same ME grid as a Kitchen Station → do not add a second AE2 kitchen tap.
8. Void / overflow tanks fill last. No ghost tank amounts.
9. Tom's connector is one inventory. Extract Tom's before virtual taps.
