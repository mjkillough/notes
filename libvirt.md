# Escaping Super+key key combinations

My WM (qtile) uses Super+key to switch between virtual desktops. I want these key combinations to work even when I have a guest focused in virt-manager.

Set the left-Super to be the "escape"/"grab" key in virt-manager and configure it not to take keyboard focus until the window is next focused:

- `dconf write /org/virt-manager/virt-manager/console/grab-keyboard false`
- `dconf write /org/virt-manager/virt-manager/console/grab-keys \"65515\"`

Disable the left-Super in Windows, so that it never opens the Start Menu:

- Add the following to the registry:
  - Key: HKLM\SYSTEM\CurrentControlSet\Control\Keyboard Layout\
  - Value: Scancode Map
  - Type: REG_BINARY
  - Data: 00000000000000000200000000005BE000000000
- Reboot

Links:

- https://bugs.freedesktop.org/show_bug.cgi?id=93249
- https://support.microsoft.com/en-us/kb/216893
- http://download.microsoft.com/download/1/6/1/161ba512-40e2-4cc9-843a-923143f3456c/scancode.doc
- http://www.howtogeek.com/howto/windows-vista/disable-caps-lock-key-in-windows-vista/