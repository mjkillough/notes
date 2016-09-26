# Multiple monitors


## xrandr

Handy description of how to get `xrandr` to cope with multiple monitors with mixed DPI settings:

- http://askubuntu.com/questions/393400/

I've written a simple wrapper around xrandr to deal with the syntax of its command line:

- https://github.com/mjkillough/dotfiles/blob/master/scripts/Scripts/configure-monitors


## udev

You can use `udev` to automatically trigger the script, using a command line like:

```
ACTION=="change", SUBSYSTEM=="drm", ENV{DISPLAY}=":0", ENV{XAUTHORITY}="/home/mjk/.Xauthority" RUN+="/home/mjk/Scripts/configure-monitors udev"
```

... however, it seems to get called before `xrandr` has had a change to pick up the changes, so we can't query to see what monitors are connected.

The internet suggests sleeping, but also warns that sleeping in a `udev` trigger blocks other `udev` events from firing and this is bad. They suggest more complex solutions (like triggering a `systemd` service), but it isn't clear that these make it much simpler to wait for `xrandr` to get itself sorted, without resorting to sleeps.

(When I tried adding a simple sleep of 3 seconds, it didn't seem to help).
