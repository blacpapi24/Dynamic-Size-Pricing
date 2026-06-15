[README.md](https://github.com/user-attachments/files/28971777/README.md)
# Dynamic Size Pricing

BepInEx 5 / Harmony mod for **Paralives** Build/Buy mode.

When a player scales or resizes a **catalog object**, the mod adjusts household funds **only when they confirm the new size** — not while dragging, and never by overwriting catalog prices or the floating `-P` cost label.

**Scope: objects only** (`ItemObjectRoot`). Walls, floors, platforms, and terrain are vanilla-priced. No exclusion lists — anything with the scale widget or resize handles qualifies.

## How money moves

```
sizeMultiplier = currentSize / baseline        // linear, NOT volumetric
scaledPrice    = max(1, round(basePrice × sizeMultiplier))
sizePremium    = scaledPrice − basePrice       // charged on size confirm
```

Example with a §800 sofa scaled to 150%:

| Action | Wallet change | Source |
|---|---|---|
| Confirm size (release blue arrow) | −§400 (premium) | this mod, `MoneyManager.ChangeMoney` |
| Place item (green check) | −§800 (base) | vanilla `EndMoveItem.TryPlaceItem` |
| **Total** | **−§1,200** | |

Verified in the game's IL: vanilla placement charges `-GetPrice()`, which never reads scale, so the two charges don't overlap. Repeated confirms compose multiplicatively — scaling back down and confirming refunds the difference. Discarding a new item before placement (Esc, right-click, undo, UI cancel, swatch click…) refunds the full premium.

## Install

1. Build (requires .NET SDK; game path is set via `GameDir` in `DynamicSizePricing.csproj`):

   ```powershell
   dotnet build -c Release
   ```

2. Copy `bin\Release\net472\DynamicSizePricing.dll` to:

   ```
   C:\Program Files (x86)\Steam\steamapps\common\Paralives\BepInEx\plugins\DynamicSizePricing\
   ```

3. Launch Paralives. Check `BepInEx\LogOutput.log` for `[DynamicSizePricing]` lines.

## Architecture

| File | Role |
|------|------|
| `Plugin.cs` | BepInEx bootstrap, `Harmony.PatchAll` |
| `Core/ScalePriceCalculator.cs` | Linear multiplier math, current scale/size readers |
| `Core/CataloguePriceResolver.cs` | Base price lookup (handles prefab-GUID vs setting-GUID mismatch) |
| `Core/PricingSessionTracker.cs` | Per-item session, cumulative multiplier, wallet delta on confirm, refund rules |
| `Patches/ScaleAndResizePatches.cs` | `StartScale/EndScale/StartResize/EndResize/CancelResizeOrScale` hooks |
| `Patches/PurchaseAndPlacementPatches.cs` | `TryPlaceItem` finalization + `CancelMoveItem` refund safety net |
| `Patches/PlacementPatches.cs` | Seeds base price when an item spawns from the catalog |

## Harmony patches (verified against Paralives.dll IL)

| Target | Purpose |
|---|---|
| `StartScaleItem.UpdateForPlayer` / `StartResizeItem.UpdateForPlayer` | Begin/refresh pricing session |
| `EndScaleItem.UpdateForPlayer` | Prefix captures item + `OriginalItemScale` + `transform.localScale`; postfix charges only when vanilla cleared `ItemInScale` (a real confirm) |
| `EndResizeItem.UpdateForPlayer` | Same pattern with `OriginalItemSize` + `CubeTransform.Size` |
| `CancelResizeOrScaleItem.UpdateForPlayer` | Prune uncharged sessions (geometry reverts to op baseline, charges already match) |
| `EndMoveItem.TryPlaceItem` (private, string-named) | Bank premium into placed-item ledger, close session |
| `CancelMoveItem.UpdateForPlayer` | Single funnel for ALL cancel paths; refunds premium when a new item is discarded |
| `DeleteItemEvent.DeleteItem` (private, string-named) | Sell/bulldoze/delete-with-refund: returns the size premium on top of the vanilla price refund |
| `CreateItemForPlacementEvent.UpdateMessage` | Warm catalogue cache + seed session base price |

## Design constraints (do not regress)

1. Never patch `GetPrice` globally — breaks catalog display.
2. Never write `ItemObjectRoot._price` — overwrites vanilla.
3. Never update the floating price label live.
4. Objects only; linear pricing; no exclusion lists.
5. All wallet movement via `MoneyManager.ChangeMoney` on size confirm / refund.
6. Use Harmony string names for non-public methods.

## Known limitations

- The placed-premium ledger lives in memory only: premiums charged before saving and reloading the game cannot be refunded on a later sell (the item sells for vanilla price only after a reload).
- Scaling an already-placed item starts a fresh session at its current size; shrinking below that size refunds linearly against the catalogue base.
