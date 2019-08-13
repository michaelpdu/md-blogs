# How to root Nexus 6P?

- Make sure you have enable unlock OEM in dev options in your device.
- Refer to XDA document [https://forum.xda-developers.com/nexus-6p/general/guides-how-to-guides-beginners-t3206928](https://forum.xda-developers.com/nexus-6p/general/guides-how-to-guides-beginners-t3206928)

-------------------

1. How to enter into bootloader?
- through adb command
```
adb reboot bootloader
```
- hardware shortcut (power key + volume down in powered off status)

2. How to unlock bootloader?
```
fastboot flashing unlock
```

3. How To Install A Custom Recovery On Your Device
Download the latest TWRP Recovery and use following command to flash custom recovery.
```
fastboot flash recovery <filename>.img
```
Use the volume keys to scroll and power key to select the Reboot Bootloader option. Once the phone has booted back into the bootloader you can use the volume keys to scroll and the power key to boot into your newly flashed recovery. It's now safe to disconnect your usb cable.

4. How to root (实测在上面一步完成以后，已经有root，安装Magisk以后，多了更多的管理能力，可以对shell进行授权)
- Download the latest root version (Magisk, SuperSU) of your choosing to your phone
- Boot into TWRP recovery and enter the install menu.
- Navigate to where you have the root zip stored on your internal storage and select it.
- Swipe to install.
- Once the zip has installed you'll have an option to wipe cache/dalvik and an option to reboot system. Wipe the cache/dalvik, hit the back button, and hit the reboot system button. That's it.

-------------------
How to flash a new system?
- Download offical firmware from [https://developers.google.com/android/images#angler](https://developers.google.com/android/images#angler)
- go to bootloader
- run flash-all
