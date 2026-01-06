---
title: Wyze Doorbell with Thingino Configuration 
description: 15$ OpenSource video doorbell solution.
slug: thingino-doorbell-wvdb1-configuration
date: 2026-01-01 00:00:00+0000
categories:
    - Homelab
tags:
    - Thingino
    - Homelab
---

I want to record the video of my front door 7x24 and notify me when there is someone there. Also I want myself get notified if someone pressed the doorbell even if I'm not at home.

After some investigation, I bought a Wyze Video Doorbell Gen 1 (Model number: WVDB1).

Mainly because it's one of the most well supported Thingino video doorbell.

## Installing Thingino

Official wiki already have detailed instruction on how to install Thingino firmware.

Link:

<https://github.com/themactep/thingino-firmware/wiki/Camera:-Wyze-Doorbell-(V1)>

## Frigate 7x24 NVR

With current thingino firmware, the video stream is rotated 90 degree. It's not recommended that we rotate it from the device itself given limited resource. Also the model don't have a TF slot, which means we practically need sth like `Frigate` to do the rotation and recording.

My Frigate config looks like below:

```yaml
go2rtc:
  streams:
    camera1_hd:
      - ffmpeg:rtsp://thingino:thingino@[ip]:554/ch0#video=h264#rotate=90#audio=copy
    camera1_sub:
      - ffmpeg:rtsp://thingino:thingino@[ip]:554/ch1#video=h264#rotate=90#audio=copy
      - rtsp://thingino:thingino@[ip]:554/ch1
  webrtc:
    candidates:
      - 127.0.0.1:8555
cameras:
  camera1:
    live:
      streams:
        Detect View: camera1_sub
        HD View: camera1_hd
    enabled: true
    ffmpeg:
      inputs:
        - path: rtsp://localhost:8554/camera1_sub
          input_args: preset-rtsp-restream
          roles:
            - detect
            - audio
        - path: rtsp://localhost:8554/camera1_hd
          input_args: preset-rtsp-restream
          roles:
            - record
```

## Two way audio

Thingino firmware supports 2 way audio (audio input and output). We can make that work with Frigate. Above config I shared contains the key entries to make 2 way audio working.

Keypart 1: original RTSP as a fallback stream to make audio output working.

```yaml
    camera1_sub:
      - ffmpeg:rtsp://thingino:thingino@[ip]:554/ch1#video=h264#rotate=90#audio=copy
      # Two way audio required original rtsp protocol
      - rtsp://thingino:thingino@[ip]:554/ch1
```

Keypart 2: WebRTC is required for 2 way audio.

```yaml
  webrtc:
    candidates:
      - 127.0.0.1:8555
```

Keypart 3: audio tag to make Frigate aware and display relevant UI elements.

```yaml
        - path: rtsp://localhost:8554/camera1_sub
          input_args: preset-rtsp-restream
          roles:
            - detect
            - audio
```

Side note: if you found the output audio is disabed when you use microphone, it's normal, otherwise there will be huge echo.

Ref doc:
<https://nils.schimmelmann.us/2025-05-18-enabling-open-source-two-way-audio-in-thingino/>

## Configure and customize doorbell button behavior

It's a doorbell, so I want the doorbell button to actually work, which means I want it to execute some custom script when it get pressed.

The key service here is `thingino-button` service.

Check the status by `service thingino-button status`.

It is configured by `/etc/thingino-button.conf`

```sh
# User mappings
#KEY_0 RELEASE 0 /bin/command1
KEY_1 RELEASE 0 sh /etc/send-msg.sh
KEY_1 RELEASE 0 play /usr/share/sounds/doorbell_3.opus
#KEY_1 TIMED 0.1 play /usr/share/sounds/doorbell_3.opus
#KEY_1 RELEASE 0 doorbell_ctrl 00:11:22:33 15 1
```

Make sure for all the edit you make to the conf file, you need to restart service `service thingino-button restart` for it to apply.

After some testing, I found for this model, `KEY_1` here is the doorbell button.

I commented out the original command and replaced `doorbell_ctrl` command with my own script to trigger home assistant script.

```sh
#!/bin/sh

TOKEN="YOUR HOME ASSISTANT TOKEN"
HA_DOMAIN="YOUR HOME ASSISTANT DOMAIN"

echo "$(date '+%Y-%m-%d %H:%M:%S') - Triggered" >> /tmp/button.log

/bin/curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"entity_id": "script.send_critical_notification"}' \
  "https://$HA_DOMAIN/api/services/script/send_critical_notification"
```

## HomeAssistant Notification

For iOS, HomeAssistant app have the capability to send critical alert, which bypass all the mute and silent mode. This is exactly what I want to achieve.

In Home Assistant, I create a script that sends notifications in parallel to all family members.

```yaml

alias: send_critical_notification
description: "Unified Critical Alert: Telegram + 3x Mobile Devices"
sequence:
  - parallel:
      - action: notify.mobile_app_(your device 1)
        data:
          <<: *unified_msg
          data: &critical_settings
            push:
              sound:
                name: "Doorbell.caf" # iOS System Sound, need to be imported in app first
                critical: 1          # Bypasses silent switch
                volume: 1.0          # Max volume

      - action: notify.mobile_app_(your device 2)
        data:
          <<: *unified_msg
          data: *critical_settings
mode: parallel
```

This is the script get trigged by the script in Thingino.
