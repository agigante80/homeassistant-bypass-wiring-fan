# Home Assistant bypass-wiring for a smart ceiling fan

Make a Tuya / WiFi smart ceiling fan controllable from Home Assistant **without losing the look or feel of the original wall switches**, by adding a Zigbee in-wall relay in bypass-wiring mode. The wall switches keep working mechanically, the fan canopy is always powered (so its WiFi chip never drops), and Home Assistant gets full smart control plus a wall-switch event it can react to.

This repo documents one specific install (SONOFF MINI-ZB2GS Zigbee relay + Ovlaim 132 cm DC ceiling fan in a Spanish two-way / *conmutador* circuit), but the technique is **brand-agnostic** and the included Home Assistant automation is parameterised so you can adapt it to your own house.

![Before: standard European two-way switch circuit](diagrams/01-before-conmutador.png)
![After: bypass mode with Zigbee relay](diagrams/02-after-bypass-wiring.png)

> Both PNG diagrams are exported from a single [Excalidraw source file](diagrams/wiring-diagrams.excalidraw). To customise them for your own setup (different brand, different room, different language), upload the `.excalidraw` file to [excalidraw.com](https://excalidraw.com/) or open it in the Excalidraw desktop app, edit, and re-export PNG. Pull requests with diagram improvements are welcome.

## The problem this solves

If you replace a "dumb" ceiling lamp with a WiFi smart ceiling fan, you hit a paradox:

* The fan's smart canopy needs **constant power** so its WiFi/Tuya chip stays on the network. The moment a wall switch cuts power, the canopy reboots and Home Assistant loses the entity.
* But people in the house still expect the wall switch on the wall to **do something useful**.
* In Spanish/European two-way (*conmutador*) circuits and US 3-way circuits, the wall switches sit between live and the lamp and break the circuit. They are not stateless push buttons; they are full single-pole-double-throw switches.

The naive options are bad:

* Tape the wall switch in the "on" position. Works, but the switch is now a lie.
* Replace the wall switch with a smart switch and put the fan downstream. The fan loses constant power and drops off WiFi every time someone flips the switch.
* Hardwire the fan past the switch and lose the wall control entirely.

## The solution

Add a small Zigbee in-wall relay (SONOFF MINI-ZB2GS in this build) at the **ceiling junction box**, not at the wall switch. Wire it like this:

* **Permanent live** lands on the relay's L terminal. The fan canopy receives its own permanent Live, independently of the relay's load output. That is the essence of bypass wiring: the load is wired *around* the relay, not *through* it. Where the fan's Live actually comes from is a separate question with multiple right answers (see "About the SONOFF MINI-ZB2GS dual L terminals" below).
* **Neutral** is shared between the relay and the fan canopy.
* **The wall switch chain** (Spanish *conmutador* / UK two-way / US 3-way) lands on the relay's **S input**, not on the load. The relay's L1/L2 load outputs stay capped and unused.
* The relay is set to **edge-trigger mode**, so every wall-switch flip toggles the relay's reported state in Home Assistant.
* A Home Assistant automation listens to the relay's switch entity and decides what to do with the fan and light.

The fan canopy is therefore **always powered**. The wall switches still feel mechanical, still snap up and down, still look identical to the neighbours' switches. Home Assistant sees a state change every time someone touches the switch and runs whatever logic you want, including: "if anything is on, turn it all off; else turn the light on" (the [included automation](automations/wall_switch_smart_toggle.yaml)).

The widely-used name for this layout is **bypass wiring** (Home Assistant community), also called **decoupled switch wiring** (Zigbee2MQTT community) or **smart-bulb wiring** when applied to bulbs. The terminology table at the bottom of this README compares it with a related-but-different firmware feature called **detached / decoupled mode**.

## Hardware

| Part | Used in this build | What to look for if you sub it out |
|---|---|---|
| **Zigbee relay** | [SONOFF MINI-ZB2GS](https://www.amazon.es/dp/B0FV2T1TZK) ("MINI DUO") | Any 2-channel Zigbee relay rated 10 A+, neutral required, edge-trigger mode supported. SONOFF ZBMINIR2 (1 channel) works for single-circuit loads. MOES MS-104BZ is functionally equivalent. |
| **Smart ceiling fan** | [Ovlaim 132 cm DC fan with integrated light](https://www.amazon.es/dp/B08RMN3YRN) (Tuya WiFi canopy) | Native Tuya or Smart Life WiFi (not an RF gateway). DC motor preferred for quietness. 230 V / 50 Hz for EU, 110 V / 60 Hz for US. |
| **Zigbee coordinator** | SONOFF Universal Zigbee 3.0 USB Dongle Plus | Any HA-supported Zigbee coordinator (ConBee, SkyConnect, etc.). |
| **Home Assistant** | HA Core 2026.4.x with ZHA + [tuya_local (HACS)](https://github.com/make-all/tuya-local) | Any modern HA. ZHA or Zigbee2MQTT for the relay; tuya_local for fully-local control of the fan canopy. Cloud Tuya integration works too if you do not mind a cloud round-trip. |

Total cost in mid-2026 from Amazon.es: ~ 18 EUR per relay + ~ 165 EUR for the fan + a one-time ~ 30 EUR for a Zigbee dongle if you do not already have one.

> Bypass wiring needs **permanent live + neutral** at the ceiling junction box. In Spanish / European *conmutador* houses, the wall-switch boxes typically only see live and travellers; the live, neutral and load wires only meet at the ceiling rosette or junction box. That is the right place for the relay, not the wall.

## Wiring

The full wire-by-wire mapping for a Spanish two-way *conmutador* circuit is documented in the diagrams. In short:

1. **Power off** at the breaker. Confirm with a multimeter.
2. **Bring permanent live to the junction box.** In most installs this means tapping the live side of the existing lighting circuit before the wall switches. If you do not feel comfortable doing this, an electrician's involvement is short and cheap.
3. **Run the relay's terminals** per the diagram:
   * Permanent live -> `L`. Route Live to the fan canopy from any convenient source: a separate breaker run, an external wago in the junction box, or (on the SONOFF MINI-ZB2GS specifically) by tapping the second `L` terminal which is internally bridged to the first. All three options yield the same bypass behaviour.
   * Neutral -> `N` and also to the fan canopy neutral.
   * Switched-live coming back from the wall-switch chain -> `S2` (or `S1`, your choice).
   * `L1` and `L2` load outputs: **leave capped and unused**.
4. **Connect the fan canopy** to permanent live + neutral, not via the relay. The fan now has 230 V at all times.
5. **Power on**, pair the relay to your Zigbee network, and pair the fan to its Tuya app + Home Assistant.

### About the SONOFF MINI-ZB2GS dual L terminals

The SONOFF MINI-ZB2GS has 7 screw terminals in the order `L L N L1 L2 S1 S2`. The two L terminals are **internally bridged**: both are Live inputs and together they act as a built-in junction inside the relay's screw block. This is a **wiring-convenience feature**, not a bypass-mode feature. Sonoff describes it in their support docs and reviewers consistently call it out: in a typical smart-switch install you need to split the incoming Live wire so that it powers the relay **and** the wall switches' COM terminals. The second L lets you do that without an external wago, saving space in a crowded junction box.

In a bypass install (this project), the second L is sometimes repurposed as a convenient tap point for the always-on load's Live wire (the fan canopy in our case). That works, but it is **optional**:

* You can tap the second L on the Sonoff to feed the load. Convenient when the load lives in the same junction box.
* You can run a separate Live wire from the breaker to the load. Cleaner if the load is on a different circuit or in a different location.
* You can splice externally with a wago. The Sonoff is then identical to any other single-L relay.

All three are valid. **Bypass wiring works on any 2-gang relay** (MOES MS-104BZ, ZBMINI Extreme, Shelly, Aqara, etc.), regardless of whether it has the dual-L convenience. The only thing that defines a bypass install is: the load's Live comes from somewhere other than the relay's L1/L2 output.

Full pinout details and Sonoff's own wiring diagrams are in [`manuals/sonoff_mini_zb2gs_user_manual_EN_V1.0.pdf`](manuals/sonoff_mini_zb2gs_user_manual_EN_V1.0.pdf).

### Safety notes

* The relay must sit **after** an MCB or RCBO rated for 16 A or less (Sonoff requirement).
* `S1` and `S2` only accept the switched-live coming back from a wall switch. **Never** connect them to neutral or earth.
* Earth (green/yellow) conductors are **not drawn** in the included diagrams for clarity. They are present in the real install, run from the breaker earth bar to the fan chassis and to any metal lamp fixture.
* For your jurisdiction, an electrician's involvement may be legally required. This README is engineering documentation, not professional electrical advice.

## Home Assistant automation

The included [`automations/wall_switch_smart_toggle.yaml`](automations/wall_switch_smart_toggle.yaml) is a generic, parameterised version of the smart toggle that runs in this house. Behaviour:

* Trigger: state change of the Zigbee relay's switch entity (any direction).
* If the fan **or** its integrated light is currently on, turn both off.
* Otherwise, turn the integrated light on.

Replace the three placeholders (`<YOUR_WALL_SWITCH_ENTITY>`, `<YOUR_FAN_ENTITY>`, `<YOUR_FAN_LIGHT_ENTITY>`) with the matching entity IDs from your install. The automation file has inline comments explaining each.

### Why this logic

It is the smallest possible behaviour that gives a satisfying "wall switch" feel:

| Light state | Fan state | What flipping the switch does |
|---|---|---|
| off | off | turn the light on (default useful action) |
| on  | off | turn the light off |
| off | on  | turn the fan off |
| on  | on  | turn both off (kill-all) |

You never need two flips to get the room dark, you never need to remember whether the fan was on, and the wall switch always does something visible.

### Variants

* **Always start the fan at a known speed**: replace the `light.turn_on` action with `fan.turn_on` + `light.turn_on` and set `data: {percentage: 17}` for the lowest speed on a 6-speed fan.
* **No integrated light**: drop the `light.*` actions entirely; the toggle then runs fan-on / fan-off.
* **Multiple loads on the same wall switch**: extend the `condition: or` block and add matching `turn_off` actions.

The automation uses `mode: single` so that a double-flip in less than a second is debounced (the second flip is dropped). If you prefer the latest-flip-wins behaviour, change to `mode: restart`.

## What is in this repo

```
.
├── README.md                          this file
├── LICENSE                            MIT, applies to the diagrams, README, and automation YAML only
├── NOTICE.md                          third-party content (PDFs) and their separate licenses
├── automations/
│   └── wall_switch_smart_toggle.yaml  generic, parameterised automation
├── diagrams/
│   ├── 01-before-conmutador.png       starting state: standard European two-way circuit
│   ├── 02-after-bypass-wiring.png     final state: bypass wiring with Zigbee relay + Tuya WiFi fan
│   └── wiring-diagrams.excalidraw     editable source for both PNGs (open at excalidraw.com)
└── manuals/                           official vendor PDFs, included for offline reference
    ├── sonoff_mini_zb2gs_user_manual_EN_V1.0.pdf
    ├── sonoff_mini_duo_quick_guide_V1.1.pdf       (multi-language quick guide)
    ├── ovlaim_52in_809_instruction_manual.pdf     (full installation + remote manual)
    └── ovlaim_smart_life_app_setup.pdf            (Tuya/Smart Life pairing flow)
```

## Resources

### Official PDFs (included locally + sources)

* SONOFF MINI-ZB2GS User Manual V1.0 EN: [local copy](manuals/sonoff_mini_zb2gs_user_manual_EN_V1.0.pdf) | [source: TinyTronics](https://www.tinytronics.nl/product_files/007652_sonoff-mini-zb2gs-zigbee-switch-user-manual.pdf)
* SONOFF MINI DUO Quick Guide V1.1 (multi-language): [local copy](manuals/sonoff_mini_duo_quick_guide_V1.1.pdf) | [source: SONOFF support](https://support.sonoff.tech/mini-zb2gs-usermanual/)
* Ovlaim 52-inch / Model 809 Instruction Manual: [local copy](manuals/ovlaim_52in_809_instruction_manual.pdf) | [source: manuals.plus](https://manuals.plus/asin/B08NSKDM5R)
* Ovlaim Smart Life app setup guide: [local copy](manuals/ovlaim_smart_life_app_setup.pdf) | source: shipped with the product

### External (not bundled)

* SONOFF MINI-ZB2GS product page: [Amazon.es B0FV2T1TZK](https://www.amazon.es/dp/B0FV2T1TZK) | [SONOFF support](https://support.sonoff.tech/mini-zb2gs-usermanual/)
* Ovlaim 132 cm DC fan product page: [Amazon.es B08RMN3YRN](https://www.amazon.es/dp/B08RMN3YRN)
* SONOFF smart-switch wiring guide (single-pole, 3-way, no neutral): [SONOFF blog](https://sonoff.tech/en-us/blogs/news/smart-switch-wiring-what-every-homeowner-electrician-should-know)
* SmartHomeScene independent review of MINI DUO and MINI DUO-L: [SmartHomeScene](https://smarthomescene.com/reviews/sonoff-mini-duo-mini-duo-l-zigbee-smart-switches-review/)
* Home Assistant `tuya_local` integration: [github.com/make-all/tuya-local](https://github.com/make-all/tuya-local)

## License

This project's original content (the README, the two diagrams, and the automation YAML) is released under the [MIT License](LICENSE). You can copy, modify, and redistribute freely as long as you keep the copyright notice. The bundled vendor PDFs in `manuals/` are **not** covered by the MIT license: they are the respective property of SONOFF (Shenzhen Sonoff Technologies Co., Ltd.) and Ovlaim, and are included verbatim for offline convenience. See [NOTICE.md](NOTICE.md) for the full attribution.

---

## Terminology: bypass vs decoupled vs detached vs smart-bulb wiring

These four terms appear interchangeably online but they do not all mean exactly the same thing. The distinction matters when you read forum threads, watch tutorials, or copy a wiring diagram from one source and an automation from another.

| Term | Type | What it means | Where you'll see it |
|---|---|---|---|
| **Bypass wiring** / **Bypass mode** | Wiring choice (hardware) | Load is fed from permanent live, not via the relay's load output. Relay's load output is left disconnected. The relay only sees the wall switch on its S input and reports its state. | Home Assistant community, Reddit, this project. Most generic and descriptive name. |
| **Decoupled switch wiring** | Wiring choice (hardware) | Synonym for bypass wiring, framed from the wall-switch's point of view: the switch is *decoupled* from the load and only acts as an input to HA. | Zigbee2MQTT community, smart-bulb tutorials. |
| **Smart-bulb wiring** | Wiring choice (hardware) | Subset of bypass wiring where the always-on load is specifically a smart bulb (which needs constant power to keep its WiFi chip online). Same topology, different vocabulary. | Smart-bulb retailer guides, Hue/LIFX community. |
| **Signal-only** / **S-only wiring** | Wiring choice (hardware) | Niche, precise label. Says the relay sees only an S-input signal, no current flows through its load output. Same setup as bypass. | Electrician-leaning posts. |
| **Detached relay mode** / **Detach relay** | Firmware feature (software toggle on the relay) | **Different concept.** The load **is** wired to the relay's L1/L2 output, but the relay's firmware ignores the S input and keeps the output on regardless of wall-switch state. HA controls the load via Zigbee only. Used when you cannot physically rewire to bypass. | SONOFF docs ("Detach relay" entity in HA), eWeLink app. |
| **Decoupled mode** | Firmware feature (software toggle on the relay) | Zigbee2MQTT's name for the same firmware feature as "detached relay mode" above. Easy to confuse with the wiring term *decoupled switch wiring*; they achieve a similar end result via different mechanisms. | Zigbee2MQTT device pages. |
| **Switch-only mode** | Firmware feature (software toggle on the relay) | Tasmota's term for the same firmware feature. | Tasmota docs. |
| **Edge-trigger mode** | Firmware feature on the relay | Independent of the above. Says the relay generates a Zigbee event on every voltage transition on its S input (rising and falling edge), rather than tracking absolute S level. Needed for a *conmutador* / 3-way wall-switch chain to act as a toggle input. Sonoff exposes this as `select.*_external_trigger_mode`. | SONOFF, Zigbee2MQTT, ZHA. |
| **Stateless switch** | Behavioural label | A wall switch that emits events but does not hold an on/off state of its own. Your wall switch becomes effectively stateless under bypass wiring. | Home Assistant ecosystem. |
| **Pulse / momentary / latching** | Physical switch type | Describes the *physical* style of the wall switch wired to S. Latching = standard rocker that stays in position (most European wall switches). Momentary = push-button that springs back. Edge-trigger mode handles both. | Electrician's vocabulary, Sonoff manuals. |

The big one to remember: **bypass wiring** (physical) and **detached relay mode** (firmware) can deliver similar end behaviour, but only one of them is happening in this project. If your relay does not expose detached mode, or if you want a brand-agnostic solution that does not depend on a specific firmware version, **bypass wiring** is the safe pick. That is what is documented here.
