## Motion sensor night light

### Requirements

While creating a motion sensor controlled night light, I found out that the requirements
for implementation were a bit more complex than "turn on when motion is detected, turn off after
a bit of time". Here's a list of my requirements:

* When it's (getting) dark, then motion detection must trigger a night scene on a set of lights.
* These lights must be turned off automatically after a few minutes.
* When motion is detected during this period, the timer must restart. The lights must not be
  turned off automatically while I am still moving around in the room.
* When the lights are reconfigured, they must no longer be turned off. This means that if I walk
  into a room and the night light triggers, the lights should no longer automatically be turned
  off after I for example hit the light switch to turn the lights fully on.
* When the lights are turned off by the timer, it is okay when they are triggered right afterwards
  by motion
* When the lights are turned of in any other way, they must stay off during a cool off period.
  This means that when I turn off the lights using a switch that is located in the room, I should
  have enough time to leave the room, without the lights being turned on again by motion detection.
  
### Solution

First of all, I created a scene for the night light. In my configuration.yaml, I configured
the scene setup in such way that it loads scenes from individual yaml files in the `scenes/` 
subdirectory:

```yaml
scene: !include_dir_list scenes
```

The scene itself goes into a file like `scenes/roomx.night_light.yaml`, and could look
somewhat like this:

```yaml
name: Roomx night light
entities:
  light.light1:
    brightness: 5
    color_temp: 367
    state: 'on'
  light.light2:
    brightness: 5
    color_temp: 367
    state: 'on'
  light.light3:
    brightness: 5
    color_temp: 367
    state: 'on'
  light.light4:
    state: 'off'
  light.light5:
    state: 'off'
  light.light6:
    state: 'off'
```

I needed a few state variables to keep track of things. The `input_boolean` is perfect for this job.

```yaml
input_boolean:
  roomx_night_light_state:
    name: Whether or not the lights are activated as a night light in room X
    initial: 'off'
    icon: mdi:weather-night
  roomx_motion_cooldown:
    name: Whether or not to prevent motion detection from activating the lights in room X
    initial: 'off'
    icon: mdi:snowflake
```

The rest is all handled in automations. Here's the first one, which takes care of activating
the scene from above when movement is detected in the dark. It will only activate when
it's dark enough (by means of a lumen sensor), when all the lights in the room are currently
turned off and when the cooldown period (which will be setup below) is not active.

```yaml
- id: roomx_night_light_on
  alias: 'Roomx night light: Turn on when there's motion in the dark'
  trigger:
  - entity_id: binary_sensor.roomx_motion
    platform: state
    to: 'on'
  condition:
  - below: '4'
    condition: numeric_state
    entity_id: sensor.roomx_lumen
  - condition: state
    entity_id: light.all_lights_in_roomx
    state: 'off'
  - condition: state
    entity_id: input_boolean.roomx_motion_cooldown
    state: 'off'
  action:
  - data:
      entity_id: scene.roomx_night_light
    service: scene.turn_on
  - delay: '2'
  - data:
      entity_id: input_boolean.roomx_night_light_state
    service: input_boolean.turn_on    
```

The following automation takes care of turning off the lights automatically after a few minutes.
They will only be turned of when the input_boolean that tracks the night light state is still active.
After turning off the lights, the night light state is made inactive.

In the automation below this one, you will see an automation that turns on the cooldown boolean when
all lights are turned off. Because it's okay when the lights are turned on right after this automation
has triggered, this automation deactivates the cooldown. It does so after a delay of a few seconds,
because otherwise, because of the async nature of home-assistant, light off events (resulting from the
`light.turn_off` call) might still arrive afterwards, enabling the cooldown again.

```yaml
- id: roomx_night_light_off_after_timeout
  alias: 'Roomx night light: Turn off after a few minutes'
  trigger:
  - entity_id: binary_sensor.roomx_motion
    for: 00:05:00
    platform: state
    to: 'off'
  condition:
  - condition: state
    entity_id: input_boolean.roomx_night_light_state
    state: 'on'
  action:
  - data:
      entity_id: light.all_lights_in_roomx
    service: light.turn_off
  - data:
      entity_id: input_boolean.roomx_night_light_state
    service: input_boolean.turn_off
  - delay: 00:00:03
  - data:
      entity_id: input_boolean.roomx_motion_cooldown
    service: input_boolean.turn_off
``` 

The following automation disables the night light state when light settings
are changed during the night light state.

Note that I have individually added all the lights that are involved in room X in the trigger.
I did try to use my zigbee-based bridge group for this instead, but that one sometimes
reported a change without anything being changed (unless I gave all the lights the exact
same settings, possibly a bug). You might be able to simply use a group here. One thing I
want to try myself, is to setup a group for these lights in Home Assistant and use that
group here. I have a strong feeling that this would work correctly too.

```yaml
- id: roomx_disable_motion_cooldown_on_change
  alias: 'Roomx night light: turn off motion cooldown on light changes'
  trigger:
  - entity_id: light.light1
    platform: state
  - entity_id: light.light2
    platform: state
  - entity_id: light.light3
    platform: state
  - entity_id: light.light4
    platform: state
  - entity_id: light.light5
    platform: state
  - entity_id: light.light6
    platform: state
  condition:
  - condition: state
    entity_id: input_boolean.roomx_night_light_state
    state: 'on'
  action:
  - data:
      entity_id: input_boolean.roomx_motion_cooldown
    service: input_boolean.turn_off
```

This automation will activate the motion cooldown boolean, when all lights are turned off.
So for example when a person hits a light switch or uses the app to turn off the lights.
Within a short period of time, the cooldown boolean will prevent the lights from being
turned on again by motion (giving the person the time to leave the room).

```yaml
- id: roomx_night_light_activate_motion_cooldown_on_lights_off
  alias: 'Roomx night light: activate motion cooldown when all lights are turned off'
  trigger:
  - entity_id: light.all_lights_in_roomx
    platform: state
    to: 'off'
  condition: []
  action:
  - data:
      entity_id: input_boolean.roomx_motion_cooldown
    service: input_boolean.turn_on
```

This automation will deactivate the cool down boolean when one or more lights
are turned on in room X. There is no functional reason at this point to keep the
cool down enabled.

```yaml
- id: roomx_night_light_deactivate_cooldown_on_lights_on
  alias: 'Roomx night light: deactivate cool down when one ore more lights are turned on'
  trigger:
  - entity_id: light.all_lights_in_roomx
    platform: state
    to: 'on'
  condition: []
  action:
  - data:
      entity_id: input_boolean.roomx_motion_cooldown
    service: input_boolean.turn_off
```

And finally, this automation will dactivate the motion cooldown after the lights are keps off
for a short period of time.

```yaml
- id: roomx_night_light_deactivate_cooldown_after_timeout
  alias: 'Roomx night light: deactivate cool down after 30 seconds'
  trigger:
  - entity_id: light.all_lights_in_roomx
    for: 00:00:30
    platform: state
    to: 'off'
  condition: []
  action:
  - data:
      entity_id: input_boolean.roomx_motion_cooldown
    service: input_boolean.turn_off
```

