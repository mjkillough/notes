Notes on getting a nice boot experience.

# Silent boot

This kernel command line seems to be silent[0][1]:

```
quiet vt.global_cursor_default=0 rd.loglevel=0 systemd.show_status=false rd.udev.log-priority=0 udev.log-priority=0
```

To disable fsck messages[0]:
- Run each of:
    - `sudo systemctl edit systemd-fsck-root.service`
    - `sudo systemctl edit systemd-fsck\@.service`
- Specifying:

```
[Service]
StandardOutput=null
StandardError=journal+console
```

# Plymouth

## Enabling Plymouth

- Install Plymouth
- In `mkinitcpio.conf`:
    - Add `plymouth` to `HOOKS`
    - Add `i915` to `MODULES`
- Add `quiet splash` to kernel command line.

## Theming/configuring

- Install `aur/plymouth-theme-arch-charge`
- Modify `/etc/plymouth/plymouthd.conf`:

```
[Daemon]
Theme=arch-charge
ShowDelay=0
```

## Transition from Plymouth to X

In order to have a seamless transition to X you need to[3][4]:
- Run `plymouth deactivate`
- Start X (ideally with `-background none` - previously known as `-nr`)
- Run `plymouth quit`

Ensure that `plymouth-quit` is not run until after X has started. (Be aware that the various `displaymanager-plymouth.service` scripts on ArchLinux have `After=plymouth-quit.service`).

You should add the following to whichever service starts X (usually your DM):

```
Wants=plymouth-deactivate.service
After=plymouth-deactivate.service
```

(See [Auto-login to X](#auto-login-to-x) for an example).

## Debugging

To debug[2]:
- Edit:
    - `/usr/lib/systemd/system/plymouth-start.conf`
    - `/usr/lib/initcpio/hooks/plymouth`
- Add: `--debug --debug-file=/root/plymouth.log`.

I did not succeed in getting this to output to the file, but it seemed to write to `/var/log/bootlog.log` and `journalctl`.

# Auto-login to X

We can use notes from the Arch wiki[5] and `xlogin-git`[6], but we create our own services:

```
# /etc/systemd/system/x@.service
[Unit]
Description=X on %I
Conflicts=getty@tty1.service
Wants=plymouth-deactivate.service
After=systemd-user-sessions.service getty@tty1.service plymouth-deactivate.service
StopWhenUnneeded=true

[Service]
Type=forking
ExecStart=/usr/bin/x-daemon -background none -noreset -nolisten tcp %I
StandardOutput=null
StandardError=null
```

```
# /etc/systemd/system/xlogin@.service
[Unit]
Description=Direct X login for user %i
After=x@vt1.service systemd-user-sessions.service
Before=plymouth-quit.service
Requires=x@vt1.service

[Service]
User=%i
WorkingDirectory=~
TTYPath=/dev/tty1
PAMName=login
Environment=XDG_SESSION_TYPE=x11 DISPLAY=:0
ExecStart=/usr/bin/bash -l .xinitrc
StandardOutput=null
StandardError=null

[Install]
WantedBy=graphical.target
```

Note:
- X runs as root, rather than the user.
- As discussed in [Transition from Plymouth to X](#transition-from-plymouth-to-x):
    - We force these services to come after `plymouth-deactivate.service` but before `plymouth-quit.service`.
    - We specify `-background none`.
- We might be able to use automatic login to a virtual console[7] and `startx`, but this might introduce a flicker when transitioning from Plymouth. Not tested.

# Smooth fade from Plymouth/DM to Wallpaper

It's quite nice that OS X has a smooth transition from the boot splash to the desktop wallpaper. We can achieve this by mimicing what GDM does in [4].

Basically, follow [Transition from Plymouth to X](#transition-from-plymouth-to-x) and have a script scrape the contents of the root window and explicitly set them as the background of the root window. From there, set the background, ideally with a tool that allows fading in from the previous background.

[set-wallpaper](https://github.com/mjkillough/set-wallpaper/) is a script which does this. Add this to your `.xinitrc` or equivalent to get a smooth transition:

```
set-wallpaper --copy-root-window
set-wallpaper --image path-to-image --fade-secs 2
```

# References

- [0] https://wiki.archlinux.org/index.php/Silent_boot
- [1] http://www.friendlyarm.net/forum/topic/2998
- [2] https://bbs.archlinux.org/viewtopic.php?id=164117
- [3] https://blogs.gnome.org/halfline/2009/11/28/plymouth-%E2%9F%B6-x-transition/
- [4] https://lists.freedesktop.org/archives/plymouth/2014-March/000753.html
- [5] https://wiki.archlinux.org/index.php/Systemd/User#Automatic_login_into_Xorg_without_display_manager
- [6] https://aur.archlinux.org/packages/xlogin-git/
- [7] https://wiki.archlinux.org/index.php/Automatic_login_to_virtual_console
