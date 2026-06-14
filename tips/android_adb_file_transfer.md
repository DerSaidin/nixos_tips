### Transfer Files to/from Android device

Settings > About Phone > tap build number until "you are a developer" appears.

Settings > System > Developer Options > USB Debugging on

```
nix-shell -p android-tools
# Run adb first to generate an adbkey.pub
adb
# Print the host's key
cat ~/.android/adbkey.pub | base64 --decode | md5sum
# Copy some files (pull does not support *)
adb pull $(adb shell ls /sdcard/DCIM/Camera/PXL_2024*) /local/destination/path/
```
