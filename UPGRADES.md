# Fork upgrades over upstream

This fork (GPL-3.0, same as upstream) changes one thing in code, ships healthier defaults, and documents the field testing behind both. The mod itself — every vehicle, every spawn point, the whole concept — is [sruckstar's work](https://github.com/sruckstar/gta5-all-mp-vehicles-in-sp). Nothing here changes what the mod does; it changes how reliably it does it.

## The code fix: emptied classes fall back instead of going dead

**Upstream behavior:** each parking lot is assigned a vehicle class. `mp_blacklist.txt` removes models from those class lists — and when *every* model in a lot's class is blacklisted, the spawner returns nothing, so the lot stays permanently empty.

Real case: blacklisting all 15 police-liveried models (see below) killed every police-assigned lot on the map, including the two cop cars that had permanently staked out the lot by Lester's place.

**Fork behavior:** an emptied class falls back to a mixed pool of ordinary street cars — sedans, coupes, compacts, muscle, sports classics, supers, tuners, SUVs. The blacklist is still respected inside the fallback pool. Dead lots become normal lots.

Code: `ParkedMP.MixedFallbackPick()`, called from `GenerateVehicleModelName()` when a class list is empty or a single-model class returns `"Blocked"`.

## The second code fix: traffic spawns never stack on survivors of the area clear

**Upstream behavior:** with `ClearSpawnArea = 1`, the spawner clears a 6 m bubble at the chosen lane position and then *skips its own occupancy check* — on the assumption the bubble is empty. But `CLEAR_AREA_OF_VEHICLES` does not remove persistent script vehicles, and the mod's own ghost traffic is persistent. A stationary player makes the road-node search deterministic (same node returned every attempt), so consecutive spawns land on the same spot and stack. Observed live: three cars materializing on one node after a script reload.

**Fork behavior:** a vehicle still standing within 5 m of the spawn position fails the attempt outright, regardless of the clear flag. Ambient traffic is still cleared exactly as before; only the stack-on-survivor case is refused. Code: the occupancy guard in `TrafficMP.AttemptSpawnOptimized`.

## Default config: Healthy Mixed Mode

Ships in [`config/`](config/) — copy both files into your `scripts\` folder next to the DLL.

```ini
[MAIN]
parking_lots_spawn = 1        ; online cars in parking lots
spawn_traffic = 1             ; online cars join road traffic
tuning = 1                    ; spawned cars arrive tuned
tuning_hsw = 1                ; ...including Hao's Special Works
doors = 0                     ; locked - steal them like a criminal
blips = 0                     ; no map blips, ever
traffic_cars_blips = 0
new_license_plates = 1
time_traffic_gen = 3000       ; mod minimum - densest allowed
max_traffic_vehicles = 6      ; lively, not Online-chaotic (range 1-10)

[ADVANCED]
SpawnDistance = 300.0         ; spawns settle off-screen before you see them
DespawnDistance = 500.0       ; long life = fewer respawn cycles
ClearSpawnArea = 1            ; REQUIRED - with 0, occupied spots fail silently
                               ; and the world looks stock

[PRESET]
preset = los_santos_balanced
```

The tuning philosophy: the streets should feel quietly richer than stock — recognizably Los Santos, not a car meet. If you want chaos, raise `max_traffic_vehicles` toward 10; everything else here holds.

## Opt-in knob: TrafficRegularBias

`ADVANCED TrafficRegularBias` (default `0` = stock behavior) sets the chance each traffic spawn uses an everyday class — compacts, sedans, SUVs, muscle, vans — regardless of zone. Stock selection is exotic-only in rich zones (sport classics and supers) and neighbor-copying elsewhere; at high bias values regular Online cars dominate traffic everywhere. Selection only: spawn rate, occupancy guards, and blacklist behavior are unchanged. Read by TrafficMP, written by hand into the ini.

## Default blacklist: mechanical fixes only

Ships in [`config/mp_blacklist.txt`](config/mp_blacklist.txt), 12 entries — every one a mechanical defect, not a taste call:

- **4 hover/fly vehicles** (`oppressor`, `oppressor2`, `deluxo`, `thruster`). NPC traffic AI cannot place or fly them — they land upside down, and they correlate with our crash telemetry: three game crashes, all identical access violations inside `ScriptHookV.dll` at offset `0x1c9fe`, during sessions with these in traffic.
- **2 amphibious transform cars** (`stromberg`, `toreador`) — transform states misbehave under AI drivers.
- **2 mega-trailers** (`terbyte`, `moc`) — clip road geometry, land flipped.

Everything else the mod spawns, it spawns by default in this fork too — **including the police vehicles**. The upstream author deliberately parks police-liveried models at lore-appropriate lots; finding a Gauntlet Interceptor staked out somewhere is part of the mod's character, so it stays.

### Taste recipe: no permanent cop stakeouts

If a lot near you draws the police class and two cruisers sitting there forever breaks your immersion (our original complaint: the lot by Lester's place), uncomment the police block at the bottom of `config/mp_blacklist.txt` (15 `pol*`/`riot2` models, listed and commented out). Because of this fork's fallback fix, those lots then spawn ordinary street cars instead of going dead — which is exactly why the fallback fix exists. One-line reversible, applies on the next **Insert** reload.

## Field notes from the test setup

- GTA V Enhanced (Steam), game build 24129078, ScriptHookV 3889, ScriptHookVDotNet nightly 3.9.0, built against that exact SHVDN3.dll.
- **ScriptHookVDotNet nightlies scan `scripts\` recursively.** An old version of any script DLL parked in a subfolder loads as a second live copy of the mod — you get two spawners, cars materializing inside each other, and physics explosions. Never keep DLLs in `scripts\` subfolders.
- Story Mode only. Never bring this (or any mod) into GTA Online.

## Install

1. Copy `All MP Vehicles in SP/bin/x64/Release/All MP Vehicles in SP.dll` (this fork's build) into your game's `scripts\` folder, replacing the existing one.
2. Copy `config/AllMpVehiclesInSp.ini` and `config/mp_blacklist.txt` into `scripts\` too.
3. Launch, or press **Insert** in game to reload scripts.

## Building from source

The csproj expects a `ScriptHookVDotNet3.dll` reference HintPath that won't exist on your machine — point it at your game's copy, or compile directly:

```
csc -target:library -platform:x64 -optimize+ -debug:pdbonly
    -out:"All MP Vehicles in SP.dll"
    -r:"<game folder>\ScriptHookVDotNet3.dll"
    -r:System.dll -r:System.Core.dll -r:System.Drawing.dll
    -r:System.Windows.Forms.dll -r:System.Xml.dll -r:System.Data.dll
    -r:Microsoft.CSharp.dll
    ParkedMP.cs SaveVehicles.cs VehList.cs TrafficMP.cs Properties\AssemblyInfo.cs
```

Upstream's `SaveVehicles.cs` uses C# 7 syntax (`out var`), so use a Roslyn compiler (Visual Studio 2022's `csc.exe`, or `dotnet`); the .NET Framework 4.x legacy compiler will reject it.
