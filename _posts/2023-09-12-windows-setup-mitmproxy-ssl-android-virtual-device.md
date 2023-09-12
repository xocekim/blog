---
title: Setup mitmproxy and android virtual device on windows to intercept SSL requests
date: 2023-09-12
categories:
- windows
tags:
- mitmproxy
- android
- ssl
---

Download [mitmproxy](https://mitmproxy.org/) into D:\tools and add to PATH
Download [Java JDK](https://jdk.java.net/) into D:\tools\jdk and add to PATH as JAVA_HOME\bin
Download [Android SDK Command Line Tools](https://developer.android.com/studio) and add to PATH as %ANDROID_HOME%\cmdline-tools\latest\bin (the SDK is picky about this)
Set ANDROID_HOME to D:\tools\android
Set ANDROID_USER_HOME to %ANDROID_HOME% to not clutter up our Windows environment
Set ANDROID_EMULATOR_HOME to %ANDROID_HOME%\emulator
Set ANDROID_AVD_HOME to %ANDROID_USER_HOME%\avd
Add ANDROID_EMULATOR_HOME to PATH
Add ANDROID_HOME\platform-tools to PATH

ANDROID_SDK_ROOT

```shell
# install platform-tools
sdkmanager "platform-tools" "platforms;android-29"
# install android 29 system-image
sdkmanager "system-images;android-29;google_apis;x86_64"
# accept licences
sdkmanager --licenses
# create avd based on pixel device
avdmanager create avd --name pixel-android-29 --package "system-images;android-29;google_apis;x86" --device "pixel"
```

Set ```hw.keyboard=yes``` in %ANDROID_USER_HOME%\avd\pixel-android-29.avd\config.ini to allow us to use our keyboard as an input device

We should now be able to boot up using ```emulator -avd pixel-android-29 -writable-system```

```shell
adb root
adb shell avbctl disable-verification
adb reboot
adb root
adb remount
```

Startup mitmweb with ```mitmweb --set confdir=D:\tools\mitmproxy-conf```

Configure the android certificate
```powershell
$certfile = "D:\tools\mitmproxy-conf\mitmproxy-ca-cert.cer"
$hash = openssl x509 -inform PEM -subject_hash_old -in $certfile | Select-Object -First 1
cp $certfile "$hash.0"
adb push "$hash.0" /system/etc/security/cacerts
adb shell "chmod 664 /system/etc/security/cacerts/$hash.0"
adb reboot
```