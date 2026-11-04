The data comes from the traffic-management system of a Technolution customer and is provided as Parquet files.

**The data is available only during the hackathon and must be deleted afterwards.**

**1. File layout**

The `data` folder contains one subfolder per dataset, with one Parquet file per day for **September and October 2026**:

```text
data/
  crossingRequests/
    _schema.txt           human-readable column list
    2026-09-01.parquet
    2026-09-02.parquet
    ...
    2026-10-31.parquet
  ntcipCommunicationStatus/
  ntcipControllerState/
  requestPhaseStates/
  scheduledDepartures/
  sensorDataCurrent/
```

Every file is a plain Parquet file (zstd compressed, rows sorted by `timestamp`). The `_schema.txt` in each folder lists the columns and their types; `_schema.arrow` is the same schema in binary form and can be ignored.

Approximate size per dataset, all days together:

| Dataset                    | Size   |
|----------------------------|--------|
| `sensorDataCurrent`        | 22 GB  |
| `scheduledDepartures`      | 2.7 GB |
| `crossingRequests`         | 2.5 GB |
| `requestPhaseStates`       | 311 MB |
| `ntcipControllerState`     | 169 MB |
| `ntcipCommunicationStatus` | 48 MB  |

**2. Reading the files**

Examples:

```python
# polars: lazy scan over all days, filter before loading into memory
import polars as pl

lf = pl.scan_parquet("data/ntcipControllerState/*.parquet")
df = lf.filter(pl.col("controllerId") == "rings.blazer_2").collect()
```

```python
# pandas / pyarrow: one day
import pandas as pd

df = pd.read_parquet("data/crossingRequests/2026-10-01.parquet")
```

```sql
-- DuckDB: all days of a dataset
SELECT intersection_id, count(*) FROM 'data/crossingRequests/*.parquet' GROUP BY 1;
```

Several columns are nested (lists of structs), for example `phases` in `ntcipControllerState`, `departures` in `scheduledDepartures` and `objects` in `sensorDataCurrent`. Use `explode`/`unnest` (polars, DuckDB) or `json_normalize` (pandas) to flatten them.

**3. Datasets**

All datasets have a `timestamp` column with millisecond precision. The `controllerId`, `intersectionId` or `intersection_id` column contains the intersection name, consisting of the names of the two intersecting roads and a suffix. This suffix is a unique number identifying the intersection and is also used in the SUMO configuration. For example, `rings.blazer_2` refers to the intersection of Rings Rd and Blazer Pkwy, with `2` as its unique identifier.

- `crossingRequests`: Information about an approaching or waiting vehicle or pedestrian at a traffic light in a specific `entryZone`.
- `ntcipCommunicationStatus`: Status of the signal controller in the central system. `controlledByTfe = true` means the central system is active at this intersection.
- `ntcipControllerState`: Status of a signal controller as reported by the controller itself. The top-level `status` is the controller mode (`FREE`, `COORDINATION` or `FLASH`). The `phases` list always contains all 16 phases, although not all are used. Per phase, `status` is `ACTIVE`, `INACTIVE` or `NEXT`, `vehStatus` is `RED`, `YELLOW` or `GREEN`, and `pedStatus` is `WALK`, `FLASHING_DONT_WALK` or `SOLID_DONT_WALK`.
- `requestPhaseStates`: Instructions from the central system to the signal controller. `currentStates` has 32 entries, one `VEHICLE` and one `PEDESTRIAN` entry per phase 1 to 16, each with a state such as `STOP_AND_REMAIN`, `PROTECTED_MOVEMENT_ALLOWED` or `PROTECTED_CLEARANCE`.
- `scheduledDepartures`: Expected departure information from the central system for a specific intersection. Optimizers at the next intersection can use this to predict approaching traffic.
- `sensorDataCurrent`: Vehicles and pedestrians detected at an intersection. Each record holds a list of `objects`. Per object, `entry_zone` identifies the zone through which the object approached the intersection (empty when not yet known), and `last_geoobject` holds the latest position, speed and heading together with `zones`, the zones in which the object is currently detected.

**Road network**

The SUMO configuration folder contains the configuration for all intersections and the links between them.

Approach zones are numbered automatically: `i1x` for the first approach (`x` = lane number), `i2x` for the second approach, and so on.