# Integration of Cambridge Audio devices into Home Assistant over RS232 via Node-RED

## Introduction

There are a large number of legacy media devices that are not network-enabled smart devices and therefore cannot be integrated into a platform like [Home Assistant](https://www.home-assistant.io/) natively. However, a lot of these devices do offer other ways of controlling them remotely via some other mechanism like an RS-232 port. Using additional hardware to connect the RS-232 port to the network, commands can be sent to a device. We then leverage HA's `Node-RED` add-on and the [Node-RED Companion](https://github.com/zachowj/hass-node-red) integration to implement the respective device's communication protocol and represent status information and functions as HA entities.

These can then be used to build custom dashboards directly or set up [Universal Media Player](https://www.home-assistant.io/integrations/universal/) entities in HA.

## Hardware

There are different options for bridging RS-232 into the network. Manufacturers like [Waveshare](https://www.waveshare.com) and USR offer industrial-grade options like the [USR-N520](https://www.pusr.com/products/2-ports-serial-to-ethernet-servers-usr-n520.html) 2-port Serial-to-Ethernet converter, which allows for the connection of two separate media devices simultaneously. There are also a number of 1-port options available. These devices can either open a simple TCP/UDP port that directly passes data on via RS-232, or they can also support MQTT, which could be used in conjunction with HA's MQTT integration.

Other, potentially cheaper, DIY hardware solutions are also possible, for instance using an ESP32-based controller or a Raspberry Pi in combination with a fitting RS-232 module as offered by Waveshare. This hardware can also be used with `EPSHome`, which is well integrated into HA.

The rest of document assumes that the user has connected the media devices to some sort of Ethernet-to-RS-232 bridge that is reachable under a fixed IP-address on the network and that has one or more TCP ports open that each correspond to exactly one RS-232 port for which all communications will be bidirectionally passed on.
 
##  Protocol: AV Receiver Azur 551R

The Cambridge Audio Azur 551R is an AV receiver with an integrated tuner and the option to connect a wide variety of audio and video sources. Associated documents, including a description of the RS-232 communication protocol, can be found on the manufacturer’s [website](https://supportarchive.cambridgeaudio.com/hc/en-us/articles/200926612-Azur-551R-V1-V2-User-Manual-IR-Codes-Technical-Specification).

The device accepts commands that largely follow what the actual remote control offers in terms of functionality. One issue is that the device does not send status updates on its own, and the protocol knows no way of querying status information without sending a command first. This creates the problem that the device state tracked by HA may diverge from the actual device’s if it is being operated by remote control or the physical control elements on the device itself, e.g. changing the input or volume. The state will be synchronised again the next time a command for the respective functionality is sent via HA and a status update is received.

The implementation of the communication protocol is in the file `flow_azur_551r.json`, which can be imported into Node-RED once set up within HA. The flow uses one central TCP connection that is being established once the first command is being sent. Since there are no unsolicited updates, it is not necessary to keep the connection alive at all times, but responses to a command will be received and routed appropriately.

It implements almost all of the functionality that is implemented in the protocol, primarily with simple button and select entities. The responses are mostly ignored, because of the aforementioned issue with staying in sync, as well as the fact that they are often not particularly relevant. The flow could be expanded with appropriate entities if desired. 

Another issue has been the fact that the actual behaviour of the device and the protocol spec differ in some ways, requiring workarounds. For instance, the response to changing the surround sound mode should arrive with group `9` and command number `3`, but responses are arriving with both `2` and `3` for some reason. Additionally, the reported levels for things like volume, bass, and treble can contain a space between the minus sign and the actual number when a negative value is being returned. This has to be removed. There may be other inconsistencies, and this may also depend on the version of the hardware itself.

In any case, when adding additional status information, one should verify that the actual format of responses is as specified and expected.

## Protocol: Blu-Ray Player Azur 752BD

The Cambridge Audio Azur 752BD is a regular Blu-Ray player. Associated documents, including a description of the RS-232 communication protocol, can be found on the manufacturer’s [website](https://supportarchive.cambridgeaudio.com/hc/en-us/articles/200995791-Azur-752BD-User-Manual-IR-Codes-Technical-Specifications-RS232-Protocol).

The communication protocol of this device is markedly different from the Azur 551R AV receiver, even though these are devices from a similar generation. The 752BD does send status updates on its own, and it also offers a number of commands to query its status without sending a command. This enables us to ensure that its status is always synchronised with its corresponding HA entities.

The Node-RED flow can be found in the file `flow_azur_752bd.json`. The flow uses one central TCP connection that is being established upon deployment. It can be used to send commands and will receive any prompted and unprompted status updates coming from the device.

All of the relevant status information is being parsed and represented as entities. However, since there are a lot of potential commands, many of which also accept a number of arguments, there are no entities for sending the individual commands. Instead, a ‘command buffer’ text input entity (`text.cambridge_audio_bluray_command_buffer) has been added that can be used to send arbitrary commands directly to the device by setting its value. This can be used with dashboards, scripts, and input helper entities via automation.

## Dashboard

A prepared dashboard that supports both of the aforementioned devices can be found in the file `dashboard.yaml`. It consists of one section for each device, both of which are structured to resemble the physical remote controls.

The section for the Blu-Ray players requires an input helper entity for sending the command for switching the input:

```
input_select:
  cambridge_audio_bluray_input_sources:
    options:
      - Blu-Ray Player
      - HDMI-FRONT
      - HDMI-BACK
      - ARC-HDMI-OUT 1
      - ARC-HDMI-OUT 2
      - OPTICAL
      - COAXIAL
    icon: mdi:audio-input-rca
```

Then add an automation to send the actual command upon changing the helper entity’s value:

```
alias: "Set Blu-Ray input"
description: "Sends the command for setting the Blu-Ray player's source"
triggers:
  - trigger: state
    entity_id:
      - input_select.cambridge_audio_bluray_input_sources
conditions: []
actions:
  - action: text.set_value
    metadata: {}
    data:
      value: >-
        SIS {{ state_attr('input_select.cambridge_audio_bluray_input_sources',
        'options').index(states('input_select.cambridge_audio_bluray_input_sources')) 
        }}
    target:
      entity_id: text.cambridge_audio_bluray_command_buffer
mode: single
```

Finally, add a second automation to keep the value of the helper entity in sync with any status updates coming from the device itself:

```
alias: "Update Blu-Ray input status"
description: "Updates the state of the Blu-Ray source selector helper"
triggers:
  - trigger: state
    entity_id:
      - sensor.cambridge_audio_blu_ray_input_source
conditions: []
actions:
  - action: input_select.select_option
    metadata: {}
    data:
      option: "{{ states('sensor.cambridge_audio_blu_ray_input_source') }}"
    target:
      entity_id: input_select.cambridge_audio_bluray_input_sources
mode: single
```

This should be all that is needed to have a working input selector on the dashboard. If desired, the same principle can be used for adding additional control elements.

## Final words

This document is a description of how to use Node-RED in Home Assistant to add support for legacy devices without native smart home support. The same approach can be used for other Cambridge Audio devices, although the communication protocol can differ widely. It can also be used to add support for other devices that have a RS-232 port. More generally, it can also be used for anything that can, directly or indirectly, be controlled via a network port.
