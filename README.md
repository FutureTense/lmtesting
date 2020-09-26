# lock-manager

Home Assistant Lock Manager integration for Z-Wave enabled locks. This integration allows you to control one (or more) Z-Wave enabled locks that have been added to your Z-Wave network.  Besides being able to control your lock with lock/unlock commands, you can also control who may lock/unlock the device using the lock's front facing keypad.  With the integration you may create multiple users or slots and each slot (the number depends upon the lock model) has its own PIN code.

Setting up a lock for the entire family can be accomplished in a matter of seconds.

For more information, please see the topic for this package at the [Home Assistant Community Forum](https://community.home-assistant.io/t/simplified-zwave-lock-manager/126765).

## Before installing

This package works with several Z-Wave Door locks, but only has been tested by the developers for Kwikset and Schlage.  It also supports an optional door sensor, and accepts a `cover` for a garage. If you aren't using the open/closed door sensor or garage cover, you can just hide or ignore the assoicated entities in the generated lovelace files. Likewise you can remove the garage cover. In fact, if wish to modify the lovelace used for all locks, you can edit the `lovelace.head` and `lovelace.code` files in the /config/packages/DOOR directory which are used to build the lovelace code for your lock.

### Each lock requires its own integration
Each physical lock requires its own integration.  When you create an integration, you give the lock a name, (such as *frontdoor*) and a directory will be created in /config/packages/ that holds the code for that lock.  So if you have a second physical lock, you will create a second integration (such as *backdoor*).  When you are finished, you will have the following directories:

    /config/packages/frontdoor
    /config/packages/backdoor


<details>
  <summary>If you're using multiple locks, please click here!</summary>
  
After you add your devices (Zwave lock, door sensor) to your Z-Wvave network via the inlusion mode, you should consider using the Home Assistant Entity Registry and rename each entity that belongs to the lock device and append `_LOCKNAME` to it. For example, if you are calling your lock `FrontDoor`, you will want to append \_FrontDoor to each entity of the lock device.  This isn't necessary, but it will make it easier to understand which entities belong to which locks.  This is especially true if you are using multiple locks.

`sensor.schlage_allegion_be469_touchscreen_deadbolt_alarm_level`
would become
`sensor.schlage_allegion_be469_touchscreen_deadbolt_alarm_level_frontdoor`

AND

`lock.schlage_allegion_be469_touchscreen_deadbolt_locked`
would become
`lock.schlage_allegion_be469_touchscreen_deadbolt_frontdoor`
</details>

## Installation

This integration can be installed manually, but the *supported* method requires you to use The [Home Assistant Community Store](https://community.home-assistant.io/t/custom-component-hacs/121727).  If you dont't already have HACS, it is [simple to install](https://hacs.xyz/docs/installation/prerequisites).

Follow [these instructions](https://hacs.xyz/docs/faq/custom_repositories) to add a custom repository to your HACS integration.  The repository you want to use is: https://github.com/FutureTense/lock-manager and you will want to install it as an `Integration`.


If all goes well, you will also see a new directory (by default `<your config directory/packages/lockmanager/>`) for each lock with `yaml` and a lovelace files. So if you add two integrations, one with FrontDoor and the other with BackDoor, you should see two directories with those names. Inside of each of those directories will be a file called `<lockname>_lovelace`. Open that file in a text editor and select the entire contents and copy to the clipboard.

> (Open file) Ctrl-A (select all) Ctrl-C (copy)

Open Home Assistant and open the LoveLace editor by clicking the elipses on the top right of the screen, and then select "Configure UI". Click the elipses again and click "Raw config editor". Scroll down to the bottom of the screen and then paste your clipboard. This will paste a new View for your lock. Click the Save button then click the X to exit.

You will also need to register the following plug-ins by pasting the following into the lovelace "resoures" section at the top of your lovelcace file.

    resources:
        - url: /community_plugin/lovelace-card-mod/card-mod.js
          type: module
        - url: /community_plugin/lovelace-fold-entity-row/fold-entity-row.js
          type: module
        - url: /community_plugin/lovelace-card-tools/card-tools.js
          type: js

The easiest way to add these plugins is using the [Home Assistant Community Store](https://community.home-assistant.io/t/custom-component-hacs/121727). You will need to go to "Settings" and add the following repositories in order to download the plugins:

- https://github.com/thomasloven/lovelace-auto-entities
- https://github.com/thomasloven/lovelace-card-tools

#### Additional Setup

Before your lock will respond to automations, you will need to add a couple of things to your existing configuration. The first is you need to add the following `input_booleans`:

```
allow_automation_execution:
  name: 'Allow Automation'
  initial: off

system_ready:
  name: 'System Ready'
  initial: off

```

and you will also need to add this `binary_sensor`

```
    allow_automation:
      friendly_name: "Allow Automation"
      value_template: "{{ is_state('input_boolean.allow_automation_execution', 'on') }}"

    system_ready:
      friendly_name: "System ready"
      value_template: "{{ is_state('input_boolean.system_ready', 'on') }}"
      device_class: moving
```

`binary_sensor.allow_automation` is used _extensivly_ throught the project. If you examine the code, you see it will refered to the `input_boolean.allow_automation_execution`. The reasons for a seperate input_boolean and binary_sensor are beyond the scope of this document. But the reason they exist is worth a discussion.

When Home Assistant starts up, several of this project's automations will be called, which if you have notificatins on will send unnecessary notifications. The allow_automations boolean will prevent those automations from firing. However, in order for this to work you stil need to add a few more things.

Add the following to you Home Asssistant Automations (not in this project!)

```
- alias: homeassistant start-up
  initial_state: true
  trigger:
    platform: homeassistant
    event: start
  action:
    - service: script.turn_on
      entity_id: script.customstartup

- alias: Zwave_loaded_Start_System
  initial_state: true
  trigger:
    - platform: event
      event_type: zwave.network_ready
    - platform: event
      event_type: zwave.network_complete
    - platform: event
      event_type: zwave.network_complete_some_dead
  action:
    - service: script.turn_on
      entity_id: script.system_cleanup
```

and likewise, add the following to your scripts:

```
system_cleanup:
  sequence:
    #- service: homekit.start If you use homekit, uncomment this statement
    - service: input_boolean.turn_on
      entity_id: input_boolean.system_ready
    - service: input_boolean.turn_on
      data:
        entity_id: 'input_boolean.allow_automation_execution'

customstartup:
  sequence:
    - service: input_boolean.turn_off
      data:
        entity_id: 'input_boolean.allow_automation_execution'
      # You can add other "startup" code here if you wish
```

This ensures that your input_boolean.allow_automation_exectuion is turned off at startup. After your Zwave devices have been loaded, script.system_cleanup is called, which will enable this boolean and allow your automations to execute. As it takes some time for your Zwave devices to actually load, use binary_sensor.system_ready sensor on the GUI so the end user knows when everything is ready to go.

#### Usage

The application makes heavy usage of binary*sensors. Each code slot in the system has it's own `Status` binary_sensor. Whenever the status of that sensor changes, the application will either \_add* or _remove_ the PIN associated with that slot from the lock. The `Status` sensor takes the results of the following binary_sensors and combines them using the `and` operator. Note, these sensors are not visible in the UI.

- Enabled
- Access Count
- Date Range
- Sunday
- Monday
- Tuesday
- Thursday
- Friday
- Saturday

**Enabled** This sensor is turned on and off by using the `Enabled` toggle in the UI. Anytime you modify the text of a PIN, this boolean toggle is turned off and must be manually turned on again to "activate" the PIN. By default, all of a slot's other UI elements are in the "always on" setting, so if you enable this boolean the PIN will be automatically added to the lock and remain that way until you turn the turn the toggle on again.

**Access Count** This sensor is controlled by the `Limit Access Count` toggle and the `Access Count` slider. If the toggle is turned on _and_ the value of the Access Count slider is greater than zero, this sensor will report true. Anytime this code slot is used to open a lock, the Access Count slider will be decremented by one. When the Access Count hits zero, this sensor is disabled and the PIN will be removed from the lock and will remain that way until you increase the Access Count slider or turn off the toggle.

**Date Range** This sensor, if enabled by it's boolean toggle will be enabled only if the current system date is between the dates specified in the date fields.

**Sunday - Saturday** These sensors are enabled by default. If you disable any of them, then the PIN won't work on that day. When enabled, the PIN will only work if the current system time falls between the specified time periods. If the periods are equal, then the PIN will work for the entire day.
