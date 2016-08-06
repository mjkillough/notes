# netctl-auto and resume

Taken from the Arch wiki[0] with some tweaks:

```
[Unit]
Description=restart netctl-auto on resume.
Requisite=netctl-auto@%i.service
After=suspend.target

[Service]
Type=oneshot
ExecStart=/usr/bin/systemctl restart netctl-auto@%i.service

[Install]
WantedBy=suspend.target
```

To enable:

```
sudo systemctl enable netctl-auto-resume@wlp3s0
```

[0] https://wiki.archlinux.org/index.php/Netctl#Problems_with_netctl-auto_on_resume