### Configure XFCE screensaver

```
xfconf-query -c xfce4-session -p /general/LockCommand -s "i3lock -c 000000" --create -t string
```
