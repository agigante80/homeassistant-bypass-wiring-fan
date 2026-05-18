# Automations

This folder contains generic, parameterised Home Assistant automations for a bypass-wired smart ceiling fan setup.

## `wall_switch_smart_toggle.yaml`

Smart-toggle behaviour for the wall switch that physically lives on the bypass-wired conmutador / 3-way chain.

**Behaviour**

| Light state | Fan state | Action when the wall switch flips |
|---|---|---|
| off | off | turn the light on (default useful action) |
| on  | off | turn the light off |
| off | on  | turn the fan off (light stays off) |
| on  | on  | turn both off (kill-all) |

## How to install

1. Open `wall_switch_smart_toggle.yaml`.
2. Replace the three placeholders with your own entity IDs:
   * `<YOUR_WALL_SWITCH_ENTITY>` -> the Zigbee relay's switch entity for the bypass-wired channel. Example: `switch.sonoff_mini_zb2gs_ceiling_fan_switch_2`.
   * `<YOUR_FAN_ENTITY>` -> your fan motor entity. Example: `fan.smart_ceiling_fan_living_room`.
   * `<YOUR_FAN_LIGHT_ENTITY>` -> the fan's integrated light entity. Example: `light.smart_ceiling_fan_living_room`.
3. Append the YAML block to your `automations.yaml` (or paste it into the HA UI: **Settings -> Automations & Scenes -> Create automation -> Edit in YAML**).
4. Reload automations: **Developer tools -> YAML -> Reload Automations** (or call the `automation.reload` service).
5. Flip the wall switch and confirm the behaviour matches the table above.

## Prerequisites on the Zigbee relay

For the wall switch's state change to actually trigger this automation, the relay must be configured correctly:

* **External trigger mode**: set to `Edge trigger`. On Sonoff MINI-ZB2GS under ZHA, this is the `select.*_external_trigger_mode_2` entity (or `_1` if you wired to S1).
* **Detach relay**: keep at `All channels disabled`. Detached mode is not used in this setup because the load is physically bypassed, not driven by the relay's output.

## Adapting the automation

The YAML has inline comments for common variants. Quick reference:

| You want | Change to make |
|---|---|
| Fan with no integrated light | Remove the `light.*` actions; default action becomes `fan.turn_on` at a chosen percentage. |
| Always start fan at lowest speed | Add `data: {percentage: 17}` (or your fan's lowest speed) to the `else:` branch and replace `light.turn_on` with `fan.turn_on`. |
| Two loads on the same wall switch | Add another `condition: state` block under `condition: or` and another `turn_off` action under `then:`. |
| Latest flip wins (no debounce) | Change `mode: single` to `mode: restart`. |

## Testing without the wall switch

You can simulate a wall-switch flip from a terminal:

```bash
curl -s -X POST \
  -H "Authorization: Bearer ${HA_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"entity_id":"<YOUR_WALL_SWITCH_ENTITY>"}' \
  "${HA_URL}/api/services/switch/toggle"
```

The state change on the switch entity triggers the automation the same way a real wall-switch flip does.
