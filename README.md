# Automating Smart Home Lights with Presence Sensors: A Comprehensive Solution

## **Introduction**

Motion activated lights have been something I've been looking for since my early days of experimentation with Home Assistant and smart lightbulbs, not only because they are cool and make a home really feel smart, but also attracted by the potential saving thanks to ensuring no light would have been forgotten on in the future.

However, what initially looks like a quick win (grab a PIR sensor, hook it up to a smart light, done) quickly turns out to be more challenging, as requirements add up. Such as:

1. **Quick response** - when lights do not switch on immediately, the user is left wondering if the system is working.
2. **Not switching on the light when the room is already lit** (by sunlight or other sources)
3. **Allowing manual override**, and not having the automation turn off the light despite it.
4. **Adapting the light brightness and colour to the time of the day** (or night)
5. **Not forcing us to move every few minutes** to keep the PIR detecting us.

Satisfying each of those requirements in turn brought up new problems, and my automations have grown in complexity in the last 3 years.

Here, I spare the details about the many attempts that lead me here, and share directly the solution which (at least for me) seems to have reached an almost perfect state.

## **Context**

A few details about my smart home environment, indeed based on Home Assistant, are useful to understand what follows.

* I organised my flat in areas: Kitchen/LivingRoom, Bedroom, Corridor, Bathroom.
* Each area has a night mode switch (boolean input), controlled by different automations.
* For each area, I have an Adaptive Lighting integration setup. It is an addon which adapts light brightness to time of the day. Setting up one integration per area allows me fine tuning the parameters for the specific room and lightbulbs.
* I am a confident and affectionate user of ESPhome.

## **The Sensors**

Initially, I experimented with DIY sensors composed of a PIR and photoresistors, but the problem of PIRs is that they can only detect moving bodies.

Eventually, I fell in love with [Everything Presence One](https://shop.everythingsmart.io/products/everything-presence-one-kit). This product contains a PIR, a mmWave sensor (able to detect you even when you don't move), a Light sensor, and other nice things like Temperature and Humidity. The sweetest part is that it is based on ESP32 and comes with ESPhome on-board.

For the sake of responsiveness, I learned over time that it is preferable to let the sensor itself process what it is capable of, and send HA the outcomes, instead of forwarding all data and waiting for HA to process it.

This is especially relevant for requirement 2 and the light sensor.

### **Light Sensing combined to PIRs and lights**

In my early attempts, I used to send HA the PIR and brightness reading, and configured the HA automation to conditionally turn on the light if the brightness was below a certain threshold (dark), and I ended up noticing the following frustrating pattern:

1. The user walks in the room, PIR detects, the room is dark and automation turns light on.
2. The room gets bright, and the brightness sensor reports this, at its next update interval.
3. The user does not move, PIR goes off, automation turns off the light.
4. User reaction is to move, to reactivate the lights. But as the brightness sensor has not yet sent its updated reading, the automation condition processes the lit room brightness and aborts the execution, keeping the user into darkness.

This gets even worse if you consider that most PIR sensors have some signal processing on-board, and once triggered stay on for a few seconds, or as long as they detect movement. As the motion light automation trigger is only triggered when the PIR status changes, user needs to wait for PIR to go off before being able to retrigger the automation.

### **EP1 solution**

The file `ep1corridoio.yaml` contains the code I run on my EP1 sensors, derived from the original code with some important modifications:

* I combined the PIR and mmWave sensors' outputs into a single internal templated presence sensor
* I added an input number which allows me specifying a brightness threshold in lux
* I created two template occupancy sensors to forward HA:
  * **Raw**: Which sends the status of the combined sensors as it is. This is also useful for other purposes, such as alerting me if something moves when I am away.
  * **Filtered**: Each time the Raw sensor goes on, this template takes an immediate reading of the brightness sensor, compares the reading against the threshold, and sends the event to HA only if the room is darker.

```yaml
- platform: template
  name: ${room} Occupancy Filtered
  id: occupancy_filtered
  device_class: occupancy
  lambda: |-
    static bool last_occupancy_state = false;
    static bool filtered_state = false;
    bool current_occupancy = id(occupancy_raw).state;
    
    if (current_occupancy && !last_occupancy_state) {
      filtered_state = id(bh1750_sensor).state < id(occupancy_brightness_threshold).state ? true : false;
    } else if (!current_occupancy) {
      filtered_state = false;
    }
    last_occupancy_state = current_occupancy;
    return filtered_state;
```

The key insight here is that **the brightness check happens at the exact moment the occupancy sensor triggers**, before the light has a chance to turn on and affect the brightness reading. This eliminates the race condition entirely.

The mmWave here is an important addition, but the logic around the light sensor could still be adopted with just a PIR.

## **The Automation**

My final solution consists of four pieces for each area: An automation to turn on the lights, an input boolean to keep track of automated light changes, an automation to revert the boolean off if the user makes manual changes, and an automation to turn the lights off when presence is not detected anymore.

### **Helpers**

Initially, I tried to have a single automation turning the lights on, and then waiting for the room empty to turn them off, but this approach leads to lots of complications and it is hardly manageable and reliable.

Finally, I realised an input boolean was all I needed. This, with time, turned out to be a much more useful idea than expected.

In short, this boolean only tells the turn_off automation if it's good to proceed. I want this to happen when the lights have been turned on by the motion automation (and optionally other automations you may have) but to not happen when I set my favourite light scene and I am waiting for guests.

Eventually, it is simple: **Any automation turning on the lights is also turning on the input boolean. Any user interaction is reverting the boolean off.**

For the Corridoio area, I created an input boolean called `input_boolean.motion_corridio`.

### **Turn On Automation**

This automation is triggered by the **Filtered** occupancy sensor. It has two important conditions:
1. The lights must currently be off (to avoid retriggering)
2. Someone must be home (to avoid triggering when away)

```yaml
alias: Motion Corridio ON
triggers:
  - trigger: state
    entity_id:
      - binary_sensor.ep1_corridio_occupancy_filtered
    to: 'on'
conditions:
  - condition: state
    entity_id: light.luci_corridoio
    state: 'off'
  - condition: state
    state: 'on'
    entity_id: binary_sensor.home_presence
actions:
  - action: adaptive_lighting.apply
    data:
      entity_id: switch.adaptive_lighting_dafault_adaptive_light
      lights:
        - light.luci_corridoio
      adapt_color: true
      turn_on_lights: true
  - action: input_boolean.turn_on
    target:
      entity_id: input_boolean.motion_corridio
```

The automation uses Adaptive Lighting to turn on the lights with appropriate brightness and color temperature for the time of day, then sets the helper boolean to indicate that the lights were turned on automatically.

### **Turn Off Automation**

This automation is triggered by the **Raw** occupancy sensor going off (meaning no presence detected for 30 seconds in the Corridoio example). The critical condition is checking that the helper boolean is on, which ensures we only turn off lights that were turned on automatically.

```yaml
alias: Motion Corridio OFF
triggers:
  - trigger: state
    entity_id:
      - binary_sensor.ep1_corridio_occupancy_raw
    to: 'off'
    for:
      hours: 0
      minutes: 0
      seconds: 30
conditions:
  - condition: state
    entity_id: input_boolean.motion_corridio
    state: 'on'
actions:
  - action: homeassistant.turn_off
    target:
      entity_id: light.luci_corridoio
  - action: input_boolean.turn_off
    target:
      entity_id: input_boolean.motion_corridio
```

Note that we use the **Raw** sensor here, not the Filtered one. This is because we want to turn off the lights regardless of brightness - we only care about brightness when turning lights *on*.

The delay (30 seconds for the hallway) can be adjusted per area based on typical usage patterns.

### **User Interaction Automation**

This is the magic that makes manual control work seamlessly. It detects when any light in the area changes state due to user interaction (by checking for a `user_id` in the state change context) and immediately resets the helper boolean.

```yaml
alias: Corridio - Reset helper if Light Turned On by User
triggers:
  - entity_id:
      - light.luci_corridoio
      - light.spot_1
      - light.spot_2
      - light.spot_3
    trigger: state
conditions:
  - condition: template
    value_template: '{{ trigger.to_state.context.user_id is not none }}'
actions:
  - action: input_boolean.turn_off
    target:
      entity_id: input_boolean.motion_corridio
```

This ensures that if you manually turn on the lights (via app, voice assistant, or physical switch), the motion automation won't turn them off when you leave. **Importantly, the helper is also reset if you change any parameter of the automatically turned on lights** (like the brightness or color), which allows you to regain manual control at any time.

Note that this automation doesn't prevent the turn-off behavior - if you want to stay in darkness, you can disable the turn-on automation. In my setup, night mode automatically disables motion-activated lighting in bedrooms, while tuning Adaptive Lighting to red for areas where I may walk at night (bathroom, corridor).

### **The Corridoio Special Case**

The hallway is an excellent example because it demonstrates how this system handles **multiple automations** controlling the same lights. In my setup, I have another automation that turns on the hallway light when the entrance door opens:

```yaml
alias: Luce Porta Ingresso
triggers:
  - trigger: state
    entity_id:
      - binary_sensor.porta_ingresso
    to: 'on'
actions:
  - action: light.turn_on
    data:
      brightness_pct: 100
      kelvin: 4010
      transition: 1
    target:
      entity_id: light.spot_1
  - action: input_boolean.turn_on
    target:
      entity_id: input_boolean.motion_corridio
```

Notice that this automation **also sets the helper boolean**. This means the turn-off automation will work correctly whether the lights were turned on by motion detection or by the door opening. The system is flexible and composable.

## **Conclusion**

This solution has been running reliably in my home for months now, and it satisfies all five requirements I started with:

1. ✅ **Quick response** - The filtered sensor eliminates processing delays
2. ✅ **Brightness awareness** - Lights only turn on when actually needed
3. ✅ **Manual override** - User interactions are respected via the helper boolean
4. ✅ **Time-adaptive** - Adaptive Lighting handles brightness and color
5. ✅ **No constant movement needed** - mmWave sensors detect static presence

The key insights that made this work:

* **Process brightness checks on the sensor**, not in Home Assistant
* **Separate filtered and raw sensors** for different purposes (on vs off)
* **Use a helper boolean** to track automation state, not complex conditions
* **Detect user interaction** via the state change context
* **Keep automations simple and composable** - multiple automations can set the same boolean

## **Future Improvements**

A logical next step would be to create blueprints for these automations, making it even easier to replicate this setup across multiple areas with minimal configuration. This would allow others to adopt this approach with just a few clicks.

I hope this helps others who have struggled with the same challenges. Feel free to adapt this approach to your own setup!
