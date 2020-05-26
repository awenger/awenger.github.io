---
layout: post
title:  "Clean android screenshots"
date:   2020-05-26 15:00:00 +0200
categories: android
excerpt: "Take screenshots and record the screen of connected android devices automatically with a clean statusbar and without any notifications"
---

As developers we frequently need to create screenshots and screen recordings to showcase new functionality and features we are working on. By default the results contain things like notifications, battery level, connectivity information or just the information that "USB debugging connected" in the status bar.

![Messy status bar in android screenshot](/assets/img/android-screenshot-cluttered.png)

I personally have the urge to clean all these things from the status bar before I start to record the screen. So far this was a manual process. And more often than not some important notifications also fell victim to this cleaning, resulting in forgotten tasks afterwards.

So once I stumbled upon Chris Banes's [tweet](https://twitter.com/chrisbanes/status/1258397404606484481) about "Demo Mode", I saw the potential to automate this process for myself and to improve my everyday android development experience. A small word of caution, my bash scripting experience is slightly above "somehow it works", so I'm looking forward to any suggestions about how to improve this solution. So here is what I'm currently using (tested on ubuntu 20.04):

The first two scripts transfer a phone that is connected via adb into the demo mode and back to normal again:

```sh
#!/bin/bash
# adb_presentation_start.sh

adb shell settings put global sysui_demo_allowed 1
adb shell am broadcast -a com.android.systemui.demo -e command enter
adb shell am broadcast -a com.android.systemui.demo -e command clock -e hhmm 1010
adb shell am broadcast -a com.android.systemui.demo -e command battery -e plugged false
adb shell am broadcast -a com.android.systemui.demo -e command battery -e level 100
adb shell am broadcast -a com.android.systemui.demo -e command network -e wifi hide
adb shell am broadcast -a com.android.systemui.demo -e command network -e mobile show -e datatype none -e level 4g
adb shell am broadcast -a com.android.systemui.demo -e command notifications -e visible false

```

```sh
#!/bin/bash
# adb_presentation_end.sh

adb shell am broadcast -a com.android.systemui.demo -e command exit

```
The [Demo Mode documentation](https://android.googlesource.com/platform/frameworks/base/+/HEAD/packages/SystemUI/docs/demo_mode.md) explains all possible commands and options to customize the status bar in demo mode.
The script currently uses a fixed time, hides the wifi state, shows very good mobile connectivity and a fully charged battery. And most importantly hides all notifications in the status bar!

The third script takes a screenshot of the phone that is connected via adb (in presentation/demo mode) and stores the image locally. The resulting screenshots are stored in a separate directory "adb-screenshots" to keep them separate from any Pictures on your machine

```sh
#!/bin/bash
# adb_screenshot.sh

TARGET_DIR="$HOME/Pictures/adb-screenshots/"
FILE_NAME="$(date +"%Y-%m-%d-%H:%M:%S").png"
SCRIPTS_DIR=$(dirname $0)

echo "waiting for device..."
adb wait-for-device

$SCRIPTS_DIR/adb_presentation_start.sh

mkdir -p $TARGET_DIR
adb exec-out screencap -p > $TARGET_DIR$FILE_NAME

$SCRIPTS_DIR/adb_presentation_end.sh

echo ""
echo "File: $TARGET_DIR$FILE_NAME"
xdg-open $TARGET_DIR$FILE_NAME & disown

```

The last script records the screen of a devices that is connected via adb in demo/presentation mode and stores the resulting video locally in the same "adb-screenshots" directory. This script is a bit more complex, as it needs to trap the `CTRL` + `c`, which is used to finish the recording, so that it can copy the resulting video after the recording finished:
```sh
#!/bin/bash
# adb_screenrecord.sh

TARGET_DIR="$HOME/Pictures/adb-screenshots/"
FILE_NAME="$(date +"%Y-%m-%d-%H:%M:%S").mp4"
SCRIPTS_DIR=$(dirname $0) 

function fetch_video()
{
  echo "retrieving file"
  sleep 1
  mkdir -p $TARGET_DIR
  adb pull /sdcard/$FILE_NAME $TARGET_DIR$FILE_NAME
  adb shell rm /sdcard/$FILE_NAME

  $SCRIPTS_DIR/adb_presentation_end.sh
  echo ""
  echo "File: $TARGET_DIR$FILE_NAME"
  xdg-open $TARGET_DIR$FILE_NAME & disown
}

echo "waiting for device..."
adb wait-for-device

$SCRIPTS_DIR/adb_presentation_start.sh

echo "ctrl+c to stop recording"
trap fetch_video SIGINT
adb shell screenrecord /sdcard/$FILE_NAME

```

With these scripts I'm now able to create clean screenshots and screen recordings with a single command. And the best thing is that it does not require the manual removal of notifications:
![Clean status bar in android screenshot](/assets/img/android-screenshot-clean.png)

Let me know if you have some suggestions to improve these scripts or if they helped you to create some beautiful screenshots and screen recordings.
