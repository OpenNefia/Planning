- Feature Name: `zones`
- Start Date: 02-25-2022
- RFC PR: [OpenNefia/Planning#0002](https://github.com/OpenNefia/Planning/pull/2)
- OpenNefia Issue: *(TBA)*

# Summary
[summary]: #summary

Elona allows players to create new buildings on the world map. These buildings can serve multiple different purposes as in-game time passes. For example, shops allow players to sell off unwanted items by leaving them in the building's map. Ranches allow the breeding of tamed creatures over time. Each building also comes with an associated tax cost to be paid monthly.

OpenNefia also needs such a system to be compatible with vanilla, but owing to an earlier suggestion, it may be possible to expand the building system into a generalized modding feature that allows these buildings to share a single map. This would be the "zones" feature proposed: a way to demarcate sections of the map as belonging to the player, as well as serving a specific function.

# Motivation
[motivation]: #motivation

The set of buildings that can be created in vanilla Elona is fairly limited. It would be a good option if modders were able to expand the system to add new deed and building types.

Having zones would allow more kinds of building configurations not possible in vanilla due to its engine design.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Area Creation

Let's walk through the process of adding a new building type, using the ranch as an example. With the proposed implementation, the ranch would consist of a single zone that covers the area inside the fencing where tamed creatures are bred. The shop register would be "assigned" to this zone, such that all register operations operate on it.

First, we need to define the entity prototype for the zone that will be created.

```yaml
- type: Entity
  id: MyMod.ZoneRanch
  parent: BaseZone
  components:
  # This component is responsible for the actual ranch logic.
  - type: ZoneRanch
```

Next, we need to create a new map blueprint and map prototype representing the ranch map.

```yaml
# /Maps/MyMod/Ranch.yml
meta:
  format: 1
  name: Ranch
grid: |
  ..........
  ..........
  ..######..
  ..#....#..
  ..#....#..
  ..#....#..
  ..##..##..
  ..........
  ..........
tilemap:
  .: Elona.LightGrass1
  '#': Elona.LightGrassBushes
entities:
- uid: 0
  components:
  - type: Map
  # Defines the zone entities to create when this map is instantiated.
  - type: MapGenZones
    zones:
    - zoneId: Ranch

      # Set the prototype of the generated zone entity.
      protoId: MyMod.ZoneRanch

      # Define the tiles this zone encompasses.
      # NOTE: Could be compressed with a specialized (de)serializer.
      tiles:
      - 4, 2
      - 4, 3
      - 4, 4
      - 4, 5
      - 5, 2
      - 5, 3
      - 5, 4
      - 5, 5
      - 6, 2
      - 6, 3
      - 6, 4
      - 6, 5
```

```yaml
# /Prototypes/MyMod/Map/Building.yml
- type: Map
  id: MyMod.MapRanch
  blueprintPath: /Maps/MyMod/Ranch.yml
```

Finally, we will create a new area entity prototype. This is the area that will be spawned when the deed for the new building is used on the world map.

```yaml
- type: Entity
  id: MyMod.AreaRanch
  parent: BaseArea
  components:
  - type: AreaStaticFloors
    floors:
      MyMod.FloorRanch: MyMod.MapRanch
  - type: AreaEntrance
    startingFloor: MyMod.FloorRanch
```

We can now create the area in-code. For zones to take effect, all maps that the zones are a part of must be generated already (such as by visiting them for the first time).

```csharp
var area = _areaManager.CreateArea(Protos.Area.MyMod_AreaRanch, parent: parentArea.Id);

// Because the map entity prototype has the `MapZones` component, 
// the following call will automatically generate new zones for it.
var map = _areaManager.GetOrGenerateMapForFloor(area.Id, new("MyMod.FloorRanch"));
```

At this point, the zone system can take advantage the map's zone.

## Zone Logic

Now we need to implement the logic of updating the ranch. Let's say that we can assign a character entity to the ranch zone acting as the breeder, and upon each week passing there is a chance for a new character to be spawned based on the active breeder's stats.

Define the `ZoneRanch` component as follows:

```csharp
[ComponentUsage(ComponentTarget.Zone)]
public sealed class ZoneRanchComponent : Component 
{
    public override string Name => "ZoneRanch";

    [DataField]
    public EntityUid? ActiveBreeder { get; set; } = null;
}
```

Let's assume we have some way of setting the breeder, such as through the shop register:

```csharp
var map = _mapManager.ActiveMap;
var putit = _gen.SpawnEntity(Protos.Chara.Putit, map.AtPos(Vector2i.One));
var zoneEntity = _zoneManager.ZonesInMap(map).Where(zone => _entityManager.HasComponent<ZoneRanchComponent>(zone.Owner));
zoneEntity.GetComponent<ZoneRanchComponent>().ActiveBreeder = putit.Owner;
```

Now we can implement the zone updating logic. Say that we will check the status of all ranch zones every month. This entity system will iterate all active ranches and update their status.

```csharp
public sealed class ZoneRanchSystem : EntitySystem
{
    [Dependency] private readonly IZoneManager _zoneManager = default!;
    [Dependency] private readonly IMapManager _mapManager = default!;
    [Dependency] private readonly IRandom _random = default!;
    [Dependency] private readonly IEntityGen _entityGen = default!;

    public override void Initialize()
    {
        SubscribeLocalEvent<MapComponent, MapOnMonthsPassedEvent>(HandleMonthsPassed);
    }

    private void HandleMonthsPassed(EntityUid uid, MapComponent mapComp, ref MapOnMonthsPassedEvent args)
    {
        foreach (var zone in _zoneManager.AllZones.Where(zone => EntityManager.HasComponent<ZoneRanchComponent>(zone.Owner))) 
        {
            var ranch = EntityManager.GetComponent<ZoneRanchComponent>(zone.Owner);

            // Load the map containing the active breeder entity.
            using (var handle = _mapManager.LoadMapTemporary(zone.MapId))
            {
                var meta = EntityManager.GetComponent<MetaDataComponent>(ranch.ActiveBreeder.Value);
                var spatial = EntityManager.GetComponent<SpatialComponent>(ranch.ActiveBreeder.Value);

                for (var i = 0; i < args.MonthsPassed; i++) 
                {
                    if (_random.OneIn(50)) 
                    {
                        if (EntityManager.IsAlive(ranch.ActiveBreeder) && meta.EntityPrototype != null) 
                        {
                            var map = handle.Map;
                            _entityGen.SpawnEntity(meta.EntityPrototype, spatial.WorldPosition);
                        }
                    }
                }
            }
        }
    }
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Zones are entities that are created in the global map (ID `-1`), such that they can be referenced at any point during gameplay.

`ZoneComponent` would be defined like so:

```csharp
[ComponentUsage(ComponentTarget.Zone)]
public sealed class ZoneComponent : Component
{
    public override string Name => "Zone";
    
    /// <summary>
    /// Map this zone is identified with.
    /// </summary>
    public MapId MapId { get; set; } = MapId.Nullspace;
    
    /// <summary>
    /// The tiles this zone encompasses in the map.
    /// </summary>
    public HashSet<Vector2i> Tiles { get; } = new();
}
```

It will be possible for zones to overlap one another.

The `MapGenZonesComponent` will create all global zones listed if they are missing.

```csharp
[ComponentUsage(ComponentTarget.Map)]
public sealed class MapGenZonesComponent : Component
{
    public override string Name => "MapGenZones";
    
    /// <summary>
    /// Map this zone is identified with.
    /// </summary>
    public List<MapGenZone> Zones { get; } = new();
}

[DataDefinition]
public sealed class MapGenZone
{
    /// <summary>
    /// ID of the zone. <see cref="ZoneId"/> is a struct wrapper around a <c>string</c>.
    /// </summary>
    [DataField]
    public ZoneId ZoneId { get; set; } = default!;

    /// <summary>
    /// Prototype ID of the generated zone entity.
    /// </summary>
    [DataField]
    public PrototypeId<EntityPrototype> ProtoId { get; set; } = default!;
    
    /// <summary>
    /// The tiles this zone will encompass in the map.
    /// </summary>
    public HashSet<Vector2i> Tiles { get; } = new();
    
    /// <summary>
    /// True if the zone has been generated already. Set after the map is generated
    /// for the first time.
    /// </summary>
    public bool IsGenerated { get; set; } = false;
}
```

# Drawbacks
[drawbacks]: #drawbacks

OpenNefia requires a building system of some kind to be compatible with vanilla, so there are no real drawbacks to adding it.

Zones, on the other hand, are additional complexity on top of the building system. However, much of the logic that would go into a zone system would also be a part of the building system, except the building system would just look at the entire area of a map instead of a subset of it. As such, the complexity isn't that much of a drawback.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Scope of Data

This is vanilla's logic for updating player-owned buildings:

```hsp
if ( adata(ADATA_ID, gdata(GDATA_AREA)) == AREA_SHOP ) {
	gosub *shop_update
}
if ( adata(ADATA_ID, gdata(GDATA_AREA)) == AREA_MUSEUM ) {
	gosub *museum_update
}
if ( gdata(GDATA_AREA) == AREA_HOME ) {
	gosub *house_update
}
if ( adata(ADATA_ID, gdata(GDATA_AREA)) == AREA_RANCH ) {
	// ...
}
```

In other words: if the area ID matches that of a building type, that building's update logic is run. Following from this, the data that indicates what areas exist and what their types are have to be at the level of areas at minimum, meaning "globally available".

Contrast this with storing the zone data in the map. This means that when zones are updated, one must load all maps from disk in order to see if their zones need updating. And in addition, there is *still* a need to at least indicate if an area has any zones that need checking before the maps are loaded.

## Area Metadata vs. Global Entities

Why not store zone data on the area entities, since those entities are already in the global map?

- An area can have more than one zone of the same type. This makes component-based storage of the data impractical, because only one component of a given type can be added to an area entity at a time.
- Eschewing the ECS system due to the single-component limitation means that areas have to implement their own "blackboard"-style storage for multiple zones. This goes against OpenNefia's design principles, where anything that potentially needs extension via mods should use the ECS system.

Also, using global entities for each zone makes simplifies iteration to all global entities with a `ZoneComponent`, in contrast to iterating each area to iterate its zone data. This also means zones do not have to be tied to an area.

## Additions from Zoning

The motivation for the additional "zones" feature expands the proposal for the buildings system:

- Vanilla's system is rather inflexible because it assumes that all the maps in an area are a part of the same building. By doing a direct port of vanilla's building implementation, it's far more complicated to have a "partitioning" system so that buildings can share a single map.
- Zoning is a desirable feature that can be implemented as a superset of the building system. Applying the concept to vanilla, a building would simply be a one-floor map with the entire area of the map counting as one zone.

One alternative is continuing with vanilla's implementation and letting mods add a zoning system on top. However, this significantly increases complexity, as the base engine and the zoning mod can constantly go out of sync.

# Prior art
[prior-art]: #prior-art

ON/Lua's building system was an attempted port of vanilla's logic. Unfortunately, it was difficult to extend, as there was no standardized format to add new "metadata" to an individual area. The fact that areas in ON/C# are ECS entities solves this problem, by allowing modders to add more components onto each area for extra state (such as tax information).

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- How will zones receive ECS events? In the provided `EntitySystem` example, the event for time passing is triggered on the current map entity. Will maps forward those events to all zones in the global map? Or will the global map also have those events triggered on it?

# Future possibilities
[future-possibilities]: #future-possibilities

This proposal could open the way for a town management system, where the player can purchase shops or other individual buildings and set up a player-owned map to their liking.
