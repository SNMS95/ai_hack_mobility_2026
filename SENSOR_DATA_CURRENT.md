## Sensor data message (`sensordatacurrent`)

Each message is a **snapshot of everything one sensor currently sees at one intersection**. A sensor sends a new snapshot several times per second. Every snapshot is complete: it lists all objects being tracked at that moment, not just the ones that changed. If an object was in the previous snapshot and is missing from the next one, the sensor has lost track of it.

### Example

```json
{
  "timestamp": 1617127200003,
  "intersection_id": "3rd_16th",
  "sensor_id": "cam-north",
  "objects": [
    {
      "object_id": "3rd_16th-964526",
      "modality": "CAR",
      "entry_zone": "61",
      "last_geoobject": {
        "timestamp": 1617127199974,
        "position": { "lat": 37.767421, "long": -122.389251 },
        "speed_ms": 10.738,
        "cog": -39.84,
        "zones": ["61"]
      }
    },
    {
      "object_id": "3rd_16th-964482",
      "modality": "PEDESTRIAN",
      "entry_zone": "608",
      "last_geoobject": {
        "timestamp": 1617127199974,
        "position": { "lat": 37.767086, "long": -122.388823 },
        "speed_ms": 0.198,
        "cog": 76.23,
        "zones": ["608"]
      }
    }
  ]
}
```

### Top level

| Field             | Type                                       | Meaning                                                                                                       |
|-------------------|--------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| `timestamp`       | integer, milliseconds since 1970-01-01 UTC | When this snapshot was taken.                                                                                 |
| `intersection_id` | string                                     | Which intersection the sensor covers. One sensor covers one intersection.                                     |
| `sensor_id`       | string                                     | Which sensor produced the snapshot. An intersection may have several sensors, each sending its own snapshots. |
| `objects`         | array                                      | All objects the sensor is tracking right now. May be empty.                                                   |

### Each entry in `objects`

| Field            | Type   | Meaning                                                                                                                                                                                                                                                                                                                                                                       |
|------------------|--------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `object_id`      | string | Tracking ID of the object. Stays the same while the sensor keeps tracking it. Treat it as unique, but be aware sensors occasionally reuse an ID for a new object after the old one leaves.                                                                                                                                                                                    |
| `modality`       | string | What kind of participant it is. One of `CAR`, `TRUCK`, `VAN`, `BUS`, `MOTORCYCLE`, `SCOOTER`, `ESCOOTER`, `CYCLE`, `CARGO_CYCLE`, `MOBILITY_SCOOTER`, `PEDESTRIAN`, `LRV` (tram or light rail), `EMERGENCY_VEHICLE`, `SNOWPLOW`, `TRAILER`, `UNKNOWN`. The sensor may change its mind as it sees an object better, so the modality of an object can change between snapshots. |
| `entry_zone`     | string | The zone through which the object **first entered** the intersection area. This is history, not current position. It keeps the same value for the whole life of the object, even after it has driven through. Use `last_geoobject.zones` for where it is now.                                                                                                                 |
| `last_geoobject` | object | The most recent observation of this object. See below.                                                                                                                                                                                                                                                                                                                        |

### `last_geoobject`

| Field       | Type                                       | Meaning                                                                                                                                                                                         |
|-------------|--------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `timestamp` | integer, milliseconds since 1970-01-01 UTC | When this observation was made. Usually slightly before the snapshot timestamp.                                                                                                                 |
| `position`  | object with `lat` and `long`               | GPS position in decimal degrees (WGS84).                                                                                                                                                        |
| `speed_ms`  | number, optional                           | Speed in metres per second. Values near zero mean the object is stationary, for instance waiting at a red light.                                                                                |
| `cog`       | number, optional                           | Course over ground, the direction of travel in degrees. Values may be negative, so normalise if you need a 0 to 360 range.                                                                      |
| `zones`     | array of strings                           | All zones the object is **currently** inside. Empty when it is between zones, for example in the middle of the intersection. An object can be in more than one zone at once when zones overlap. |

### About zones

Zones are named polygons drawn on the map around the intersection. The names are opaque strings such as `"61"` or `"608"`. Zone names are specific to one intersection. The important kinds are:

- **Entry zones**: the approach lanes before the stop line. An object in an entry zone is waiting to cross or about to cross.
- **Exit zones**: the lanes just after the intersection. An object in an exit zone has crossed and is leaving.
- Other zones may exist, for example crosswalks and waiting areas for pedestrians, or remote detection zones further up the road.

Which zone names are entry zones, exit zones and so on is defined in the network configuration of the intersection, not in this message. If you receive a network or zones file with the dumps, use it to interpret the names.

### Things to keep in mind when processing dumps

- Dumps are usually an array of many snapshots in time order. Diff consecutive snapshots from the **same** `sensor_id` to detect objects appearing, moving between zones, and disappearing.
- Some dumps contain extra fields per object, such as `exit_zone`. These are not part of the official message and the traffic system ignores them, but you may use them if present.
- Snapshots older than a few seconds are considered stale by the traffic system and are not acted on. The exact limit is a configuration setting.
- There is a separate message type, `sensorobjectfinal`, that is sent once when an object disappears. It contains the full history of the object: first and last seen times, entry and exit zone, the ordered list of zones it passed through, and every observation. The snapshot message described here contains no history, only the latest observation.