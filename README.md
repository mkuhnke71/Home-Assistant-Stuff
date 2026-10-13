# Home-Assistant-Stuff
# Node-RED Automatic Turn-Off Flow

## Overview

This Node-RED flow provides a reusable **Turn Off Lights - Perform Action** subflow for automatically turning off Home Assistant entities based on occupancy, minimum run time, absence time, and optionally illuminance. It can be used for lights as well as other entities that can be turned off, such as fans or similar devices.

The flow is designed to be conservative: it evaluates whether an entity should be turned off and otherwise leaves it unchanged. If the target was turned on manually, the flow extends the configured timing thresholds before turning it off.

## Requirements

- Node-RED
- Home Assistant
- `node-red-contrib-home-assistant-websocket`
- A configured Home Assistant server node
- **Enable Global Context Store** enabled on the Home Assistant server configuration, because the function reads Home Assistant states through global context
- Illuminance Integration
- Helper Entity - input_boolean.bool_motion_activated_lights, this is so you can turn on/off the logic in node-red

## Importing the Flow

1. Open Node-RED.
2. Select **Menu > Import**.
3. Import the supplied flow JSON.
4. Make sure the Home Assistant server nodes point to your Home Assistant instance.
5. Deploy the flow.

The export contains an example using `light.foyer` and `binary_sensor.presence_sensor_front_room_foyer_zone`, triggered every 30 seconds.

## Basic Usage

Place an instance of the **Turn Off Lights - Perform Action** subflow in your flow and connect a trigger to its input. A repeating Inject node is a simple way to periodically evaluate a room or device.

Configure the subflow instance by editing its environment properties.

### Subflow Configuration

| Setting | Purpose | Default |
|---|---|---:|
| `ENTITYID` | Home Assistant entity to evaluate and turn off | Required |
| `SENSOR` | Occupancy/motion sensor entity. Leave blank for time-only operation | Blank |
| `MIN_SEC_ABSENCE` | Minimum continuous absence required before turn-off, in seconds | `0` |
| `MIN_SEC_ON` | Minimum time the target must have been on before it is eligible for turn-off, in seconds | `0` |
| `OVER_X_MINUTES` | Occupancy-history window analyzed when a sensor is configured | `10` |
| `PRESENCE` | How historical occupancy intervals are summarized: `TOTAL`, `LONGEST`, `AVG`, or `LAST` | `LONGEST` in the subflow configuration |
| `IS_DARK_LUX` | Illuminance threshold. `0` disables the illuminance override | `0` |
| `REASON` | Optional text added to the decision description | Blank |
| `DEBUG` | Enables additional Node-RED debug information | `false` |

## Example Configuration

The included Foyer example uses:

```text
ENTITYID = light.foyer
SENSOR = binary_sensor.presence_sensor_front_room_foyer_zone
MIN_SEC_ABSENCE = 60
MIN_SEC_ON = 30
```

An Inject node runs the subflow every 30 seconds.

With this configuration, the foyer light is evaluated every 30 seconds. If it is on, the flow determines whether the room is occupied and whether the configured minimum times have been met before deciding whether to turn the light off.

## How the Decision Logic Works

### 1. Automation Enable Check

The flow first checks `input_boolean.bool_motion_activated_lights`. Processing continues only when this helper is `on`. This acts as a master enable/disable switch for motion-based automatic turn-off behavior.

### 2. Target State Check

The flow checks the entity identified by `ENTITYID`. If the entity is already off, there is nothing to turn off. If it is on, evaluation continues.

### 3. When an Occupancy Sensor Is Configured

If `SENSOR` contains an occupancy sensor entity ID:

1. The flow checks the sensor's current state.
2. If the room is currently occupied, the target is left on.
3. If the room is not occupied, Home Assistant history is retrieved for the configured `OVER_X_MINUTES` window.
4. The history is analyzed using the selected `PRESENCE` method.
5. The target becomes eligible to turn off when both of these conditions are satisfied:
   - It has been on for at least `MIN_SEC_ON` seconds.
   - The room has been continuously unoccupied for at least `MIN_SEC_ABSENCE` seconds.

The historical occupancy calculation is also added to the diagnostic output and description.

### 4. When No Occupancy Sensor Is Configured

Leave `SENSOR` blank for devices that should turn off based only on how long they have been on.

In this mode, `MIN_SEC_ON` acts as the turn-off threshold. Once the target has been on for at least that amount of time, it becomes eligible to turn off.

This is useful for devices such as fans or other loads where occupancy does not need to determine the shutoff time.

> **Important:** With no sensor and `MIN_SEC_ON = 0`, an entity that is on is immediately eligible to turn off when the subflow runs.

## Presence Calculation Modes

When a sensor is configured, historical occupancy intervals within `OVER_X_MINUTES` can be summarized four ways:

- **TOTAL**: total occupied time across all occupancy intervals.
- **LONGEST**: duration of the longest individual occupancy interval.
- **AVG**: average duration of the occupancy intervals.
- **LAST**: duration of the most recent occupancy interval.

These values provide context about recent activity. Actual turn-off eligibility uses the current continuous absence and minimum-on-time thresholds.

## Manual Turn-On Protection

The flow determines whether the target's latest state change was attributed to the configured Node-RED/Home Assistant user.

If it determines that the entity was turned on manually:

- `MIN_SEC_ABSENCE` is doubled.
- `MIN_SEC_ON` is doubled.
- If either resulting value is still `0`, that value is set to 10 minutes.

This gives manually activated devices more time before automation switches them off.

## Illuminance Override

Set `IS_DARK_LUX` to a number greater than zero to enable an illuminance-based turn-off check.

If the target is still on, was activated automatically, and the current illuminance is above the configured threshold, the flow can turn the target off.

By default the function uses:

```text
sensor.illuminance
```

The decision function also supports `msg.illuminance_entity` if a different illuminance entity is supplied upstream.

## Groups

When the decision is to turn an entity off, the flow includes a **Recursively Expand Groups** function. If the configured entity represents a Home Assistant group, nested groups are recursively expanded into individual entities.

The expansion logic:

- Handles nested groups.
- Avoids infinite recursion caused by circular group membership.
- Removes duplicate individual entities.
- Preserves the incoming message on each generated entity message.
- Marks entities that could not be resolved from Home Assistant state data.
- Adds entity number/count information so completion can be detected.

Useful generated properties include:

```text
msg.entity_id
msg.group_source
msg.entity_number
msg.entity_count
msg.unresolved
```

## Completion Output

After group/entity processing, the **Completion Check** function only forwards the final processed entity message. The resulting message includes:

```javascript
msg.topic = "group_turn_off_complete";
msg.complete = true;
```

`msg.payload` has this general form:

```json
{
  "status": "complete",
  "source": "original entity or group",
  "entity_count": 3,
  "last_entity": "light.example",
  "completed_at": "ISO timestamp"
}
```

Use the subflow output when another automation needs to wait until turn-off processing has completed.

## Important Output Properties

The decision logic can add the following properties to the message:

| Property | Description |
|---|---|
| `msg.action` | Decision: `off` or `leave` |
| `msg.descr` | Human-readable explanation of the decision |
| `msg.currently` | Current normalized target state |
| `msg.fullauto` | Indicates whether the most recent target change is attributed to the automation user |
| `msg.on_for_seconds` | How long the target has currently been on |
| `msg.Occupied` | Normalized current occupancy: `on` or `off` |
| `msg.SecondsSinceOccupancy` | Seconds since the occupancy sensor last became unoccupied |
| `msg.SecondsofOccupancy` | Historical presence duration calculated using `PRESENCE` |
| `msg.OccupancyCount` | Number of occupancy intervals found in the history window |
| `msg.current_lux` | Current lux when the illuminance logic is evaluated successfully |
| `msg.error` | Fail-safe error description when evaluation cannot safely continue |

## Home Assistant Logbook

When turn-off processing occurs, the flow writes explanatory information to the Home Assistant logbook using `logbook.log`. This makes it easier to see why the automation acted.

The description generated by the flow can include information such as:

- how long the entity has been on;
- how long the room has been unoccupied;
- whether minimum timing thresholds have been met;
- recent presence calculation;
- whether the entity was manually turned on;
- whether illuminance caused the turn-off.

## Debugging

Set:

```text
DEBUG = true
```

to enable the flow's diagnostic paths and Debug sidebar output.

Useful debug nodes include:

- **Auto Off Action**
- **Leave Action**
- **Turn Off**
- **Completion Output**
- **Error Catch - Auto Turn Off**

For normal operation, leave `DEBUG` set to `false`.

## Fail-Safe Behavior

The decision function defaults to `msg.action = "leave"`. If required Home Assistant state data is missing, an entity cannot be found, a configured sensor has an invalid state, or history cannot be evaluated safely, the logic reports an error rather than intentionally selecting an `off` action.

Check the Node-RED Debug sidebar and the error catch output when troubleshooting.

## Recommended Setup Pattern

For each room or device:

1. Add a periodic Inject node or other trigger.
2. Connect it to an instance of **Turn Off Lights - Perform Action**.
3. Set `ENTITYID`.
4. Set `SENSOR` if occupancy should affect shutoff.
5. Set `MIN_SEC_ON`.
6. Set `MIN_SEC_ABSENCE` when using occupancy.
7. Adjust `OVER_X_MINUTES` and `PRESENCE` if desired.
8. Set `IS_DARK_LUX` only if illuminance should be considered.
9. Test with `DEBUG = true`.
10. After verifying the decisions, set `DEBUG = false`.

## Troubleshooting

### Nothing happens

Check that `input_boolean.bool_motion_activated_lights` is `on`, the trigger is firing, the subflow is deployed, and `ENTITYID` is valid.

### Home Assistant global state is unavailable

Open the Home Assistant server configuration in Node-RED and make sure **Enable Global Context Store** is enabled.

### Sensor is enabled but evaluation fails

Make sure `SENSOR` contains a valid Home Assistant entity ID, for example:

```text
binary_sensor.office_occupancy
```

### Device turns off too quickly

Increase `MIN_SEC_ON` and/or `MIN_SEC_ABSENCE`. Also verify that the occupancy sensor reports the expected state.

### Device does not turn off after manual activation

This can be expected. Manual activation causes the timing thresholds to be doubled and imposes a minimum 10-minute value when a configured threshold is zero.

### History-related errors

Verify that the occupancy sensor exists, Home Assistant history is available for it, and `OVER_X_MINUTES` is a valid positive history window.

## Safety / Testing Recommendation

Before using the subflow on important devices, run it with `DEBUG = true` and conservative timing values. Review `msg.action` and `msg.descr` in the Debug sidebar to confirm that the automation is making the expected decisions.
