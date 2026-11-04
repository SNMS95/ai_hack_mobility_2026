## Crossing request message (`crossingrequest`)

A crossing request says: **"this traffic participant wants to cross this intersection, from this approach, at this time."** Crossing requests are the demand that the traffic light optimizer plans its green phases around. They are derived from sensor data, from traffic controller detectors, and from predictions about traffic arriving from neighbouring intersections.

A request is created when a participant is first noticed, updated whenever new information arrives (same `id`, newer `version`), and withdrawn when the participant has crossed or disappears.

### Example

```json
{
  "id": "crossing-Vehicle-1532",
  "requestTimestamp": 1617127198400,
  "crossingDuration": 2000,
  "creationTimestamp": 1617127198412,
  "version": 1617127201105,
  "onBehalfOf": { "id": "Vehicle-1532", "type": null, "modality": "CAR" },
  "intersection": { "id": "3rd_16th", "type": null },
  "incomingLink": { "id": "2.1", "type": null, "lane": null },
  "movements": [
    { "outgoingLink": { "id": "2.3", "type": null, "lane": null }, "probability": 0.8 },
    { "outgoingLink": { "id": "2.4", "type": null, "lane": null }, "probability": 0.2 }
  ],
  "entryZone": "61",
  "source": "SENSOR",
  "sourceId": "SENSOR-3rd_16th-964526-7",
  "reliability": 80,
  "stuck": false,
  "messageExpiryInterval": 0
}
```

### Fields

| Field                   | Type                                       | Meaning                                                                                                                                                                                                                                                                                                                                      |
|-------------------------|--------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `id`                    | string                                     | Unique ID of the request. All updates of the same request carry the same `id`. The prefix after `crossing-` tells you which engine made it: `Vehicle`, `Pedestrian`, `LRV`, `Firetruck`, `Starvation`, `Sweeper`.                                                                                                                            |
| `requestTimestamp`      | integer, milliseconds since 1970-01-01 UTC | The time the participant wants to cross the intersection. See the section below, because its meaning depends on `source`.                                                                                                                                                                                                                    |
| `crossingDuration`      | integer, milliseconds, may be null or 0    | How long the participant needs to clear the intersection once it starts moving. The optimizer must fit `requestTimestamp` plus this duration inside a green window.                                                                                                                                                                          |
| `creationTimestamp`     | integer, milliseconds since 1970-01-01 UTC | When the first version of this request was created. Never changes.                                                                                                                                                                                                                                                                           |
| `version`               | integer, milliseconds since 1970-01-01 UTC | When this version was created. Increases with each update. Use it to order updates of the same `id`.                                                                                                                                                                                                                                         |
| `onBehalfOf`            | object                                     | The participant the request is for. `id` is a stable ID for the participant, `type` is usually null, `modality` is the participant kind (same values as in the sensor data message: `CAR`, `TRUCK`, `BUS`, `CYCLE`, `PEDESTRIAN`, `LRV`, `EMERGENCY_VEHICLE`, and so on, plus `BUSTER` and `SWEEPER` for artificial requests, see `source`). |
| `intersection`          | object                                     | The intersection the request is for. `id` matches `intersection_id` in the sensor data. `type` is usually null.                                                                                                                                                                                                                              |
| `incomingLink`          | object                                     | The road link on which the participant approaches the intersection. `id` is the link name from the network configuration. `type` and `lane` are usually null.                                                                                                                                                                                |
| `movements`             | array                                      | Where the participant may go. Each entry has an `outgoingLink` (same shape as `incomingLink`) and a `probability`. The probabilities add up to 1. When the system does not know the destination, all possible exits are listed with equal probability.                                                                                       |
| `entryZone`             | string or null                             | The entry zone the participant is currently standing in, matching a zone name from the sensor data. Null when the participant is not yet at the intersection, for example when the request is a prediction of traffic coming from upstream.                                                                                                  |
| `source`                | string                                     | What the request is based on. See the table below.                                                                                                                                                                                                                                                                                           |
| `sourceId`              | string                                     | The ID of the participant as the source knows it, formatted as `SOURCE-location-id`. For `SENSOR` requests the middle part is the intersection and the last part is the sensor `object_id` followed by a counter.                                                                                                                            |
| `reliability`           | integer 0 to 100                           | How much the system trusts this request. Depends on the source. Predictions derived from other predictions get a lower score.                                                                                                                                                                                                                |
| `stuck`                 | boolean                                    | True when the participant did not move even though it already had enough green time. The optimizer gives such requests very low priority.                                                                                                                                                                                                    |
| `messageExpiryInterval` | integer, seconds                           | How long after `version` the request is considered expired. `0` means it does not expire by itself and will be explicitly withdrawn. Informational only.                                                                                                                                                                                     |

### Values of `source`

| Value           | Meaning                                                                                                                                                       |
|-----------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `SENSOR`        | A sensor at this intersection sees the participant, either in an entry zone (waiting here) or in an exit zone of the previous intersection (on its way here). |
| `REMOTE_SENSOR` | A sensor zone further up the road detected the participant.                                                                                                   |
| `VIRTUAL`       | Predicted from the planned departure of a participant at an upstream intersection. Nobody has seen it on this link yet.                                       |
| `NTCIP_PREEMPT` | A preempt request from the traffic controller, typically an emergency vehicle.                                                                                |
| `NTCIP_LOCAL`   | A local detector of the traffic controller, for example a transit vehicle tag reader at the intersection.                                                     |
| `NTCIP_REMOTE`  | A remote detector of the traffic controller, further up the road.                                                                                             |
| `NTCIP_CALL`    | The traffic controller reports a call on a phase that did not come from this system, for example a push button or a loop detector.                            |
| `STARVATION`    | Artificial request created to guarantee an approach gets green after waiting too long. The participant has modality `BUSTER`.                                 |
| `SWEEPER`       | Artificial request created to flush out remaining vehicles. The participant has modality `SWEEPER`.                                                           |

### What `requestTimestamp` means

The timestamp is always about crossing the intersection, never about reaching a zone. But how it is calculated differs:

- **Participant already at the intersection** (`SENSOR` with a non-null `entryZone`, and the NTCIP local sources): the timestamp is the time the participant **started waiting**. It is now or in the past. When the approach was already backed up, the system may set it earlier than the moment the participant was first seen, to credit it the queue wait. The difference between the actual departure and this timestamp is the delay the optimizer tries to minimise.
- **Participant still on its way** (`SENSOR` with null `entryZone`, `REMOTE_SENSOR`, `VIRTUAL`, NTCIP remote sources): the timestamp is the **predicted arrival** at the intersection, computed as the detection time plus the travel time of the remaining road. It is in the future. Such a request usually also has a non-zero `messageExpiryInterval`, since it becomes meaningless once the predicted time plus `crossingDuration` has passed.

When a participant that was predicted to arrive is then actually seen in an entry zone, the system updates the **same request** rather than making a new one: `source` becomes `SENSOR`, `entryZone` gets filled in, and `requestTimestamp` becomes the moment it entered the zone.

### Lifecycle and how to read the dumps

- A dump is a stream of versions. Group by `id`, sort by `version`, and you get the history of each request.
- A request is **withdrawn** by publishing an empty message on its topic. In a dump this may appear as an empty or null payload for that `id`, or the `id` may simply stop appearing. Treat the last non-empty version as the final state.
- Withdrawal happens when the participant leaves the entry zone (it crossed), when the sensor loses it, when a prediction is no longer plausible, or when the participant's destination changes to another intersection.
- Several participants can be waiting at once, so expect many open requests per intersection. For a car, one request. For a pedestrian at a corner serving two crossings, two requests with different `movements`.
- The MQTT topic carries the intersection and request IDs too: `trafficflowengine/1/trafficsituationengine/crossingrequest/<intersection id>/<request id>`.