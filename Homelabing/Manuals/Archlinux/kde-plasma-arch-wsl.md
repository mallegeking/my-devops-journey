# KDE Plasma 6 on Arch Linux under WSL2

Setup runbook. Built and verified 26.08.2026 on `mrprivii-HomePC`.

Full Plasma 6 desktop running in a headless X session inside WSL, served over VNC to a viewer on the Windows host. WSLg cannot host a full desktop session, only individual GUI apps, which is why this goes through VNC instead.

**Final state:** Xvnc on display `:1`, port 5901, Plasma 6 with software rendering.

---

## Prerequisites

- Windows 11 with WSL2
- Arch Linux distro installed (`wsl --install archlinux`)
- A VNC client on Windows. TigerVNC Viewer (`vncviewer64-*.exe`) is a standalone binary, no installer needed.

---

## 1. Enable systemd

`/etc/wsl.conf`:

```ini
[boot]
systemd=true

[user]
default=sticks
```

Set `default` by username, not numeric UID. If the UID ever changes, a numeric entry breaks the launcher and drops you into root.

From PowerShell:

```powershell
wsl --shutdown
```

---

## 2. Base packages

```bash
sudo pacman -Syu
sudo pacman -S base-devel git sudo vim
```

---

## 3. Locale

Plasma floods the log with Qt warnings if no UTF-8 locale is generated. Do this before anything else.

```bash
sudo sed -i 's/^#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
sudo sed -i 's/^#de_DE.UTF-8 UTF-8/de_DE.UTF-8 UTF-8/' /etc/locale.gen
sudo locale-gen
echo 'LANG=en_US.UTF-8' | sudo tee /etc/locale.conf
```

---

## 4. Plasma and dependencies

```bash
sudo pacman -S plasma-meta plasma-x11-session xorg-server xorg-xinit \
               xcb-util-cursor tigervnc \
               konsole dolphin kate noto-fonts ttf-dejavu
```

Two packages that are easy to miss and both fatal if absent:

**`plasma-x11-session`** — since Plasma 6.4, Arch split kwin into `kwin-wayland` and `kwin-x11`, and moved the X11 session into its own package. Pacman will not pull it in automatically. Without it, `startplasma-x11` does not exist.

**`xcb-util-cursor`** — Qt 6.5 and later refuse to load the xcb platform plugin without it. The error reads `From 6.5.0, xcb-cursor0 or libxcb-cursor0 is needed`.

Verify:

```bash
which startplasma-x11
```

---

## 5. Session wrapper

TigerVNC 1.16 deprecated `~/.vnc/xstartup`. It now reads session definitions from `/usr/share/xsessions/` instead, and if you do not name one explicitly it picks whatever it finds first. On this machine that was i3.

Because `plasmax11.desktop` alone carries no environment, wrap it:

```bash
cat > ~/plasma-vnc.sh <<'EOF'
#!/bin/sh
unset WAYLAND_DISPLAY
export XDG_SESSION_TYPE=x11
export XDG_CURRENT_DESKTOP=KDE
export DESKTOP_SESSION=plasma
export QT_QPA_PLATFORM=xcb
export QT_QUICK_BACKEND=software
export KWIN_COMPOSE=Q
export LIBGL_ALWAYS_SOFTWARE=1
export PLASMA_USE_SYSTEMD=0
exec startplasma-x11
EOF
chmod +x ~/plasma-vnc.sh
```

What each variable is doing:

| Variable | Reason |
|---|---|
| `unset WAYLAND_DISPLAY` | WSLg leaks this in, Qt then tries Wayland first and fails |
| `QT_QPA_PLATFORM=xcb` | Force X11, do not let Qt guess |
| `QT_QUICK_BACKEND=software` | No GPU. Without this, Qt Quick throws `Failed to compile shader` and `Failed to build graphics pipeline state` |
| `LIBGL_ALWAYS_SOFTWARE=1` | Same reason, for GL clients |
| `KWIN_COMPOSE=Q` | XRender compositing instead of OpenGL |
| `PLASMA_USE_SYSTEMD=0` | Skip the systemd startup path. Safe fallback while `user@1000.service` is unreliable under WSL |

Register it. **The file must go in `/usr/share/xsessions/`.** TigerVNC does not look in `~/.local/share/xsessions/`.

```bash
sudo tee /usr/share/xsessions/plasma-vnc.desktop > /dev/null <<'EOF'
[Desktop Entry]
Name=Plasma VNC
Exec=/home/sticks/plasma-vnc.sh
TryExec=/home/sticks/plasma-vnc.sh
Type=XSession
EOF
```

---

## 6. VNC config

```bash
mkdir -p ~/.vnc
cat > ~/.vnc/config <<'EOF'
geometry=1920x1080
depth=24
localhost=no
session=plasma-vnc
EOF
```

The `session=` line is what stops it falling back to i3.

```bash
vncpasswd
```

---

## 7. Start and connect

```bash
vncserver :1
```

Confirm these two lines appear:

```
Using desktop session plasma-vnc
Starting desktop session plasma-vnc
```

Then from Windows, point TigerVNC Viewer at:

```
localhost:5901
```

First launch takes twenty to thirty seconds. Software rendering with a cold Plasma cache is slow once and fast after.

---

## 8. Post-install

Baloo will try to index `/mnt/c` and eat the CPU:

```bash
balooctl6 disable
balooctl6 purge
```

In System Settings, disable screen locking and power management. Neither makes sense here, and the locker can leave you at a frozen screen with no way back in.

---

## Daily use

```bash
vncserver :1              # start
pkill -u sticks Xvnc      # stop
```

`vncserver -kill :1` may fail on this build with a usage error. `pkill` is reliable.

Check what is running:

```bash
ps -u sticks -o pid,cmd | grep -i vnc
```

Log file for any failure: `~/.vnc/mrprivii-HomePC:1.log`. The terminal output is much less useful.

---

## Known issues and workarounds

### user@1000.service fails with EBUSY

```
Failed to spawn executor: Device or resource busy
user@1000.service: Failed with result 'resources'
```

WSL2 does not cgroup-namespace its distros. Every distro's PID 1 targets the same path `/user.slice/user-$UID.slice/user@$UID.service/`. If Ubuntu and Arch both have a UID 1000 user, whichever boots second loses.

Workaround, and the simplest one: **boot Arch before Ubuntu and docker-desktop.** Quit Docker Desktop from the tray first, or it relaunches its distro behind you.

Recovery without a full restart, if the cgroup has since emptied:

```bash
sudo systemctl reset-failed user@1000.service
sudo systemctl start user@1000.service
```

Permanent fix would be renumbering one distro's user to UID 1001. Renumber **Ubuntu's**, not Arch's. See the warning below.

Not actually a blocker either way. `XDG_RUNTIME_DIR` is created by WSL's own init, `dbus-run-session` supplies the bus, and `PLASMA_USE_SYSTEMD=0` keeps Plasma from asking systemd for anything.

### Do not attempt usermod -u from inside your own session

`usermod` refuses to renumber a user with live processes, and WSL keeps a session leader alive for your user even when you log in as root. `loginctl terminate-user` does not clear it.

```
usermod: user sticks is currently used by process 190
```

What makes this dangerous is a partial application. `groupmod` and `chown -R` succeed while `usermod` fails, leaving files owned by a UID that does not exist and locking you out of your own home directory. Symptom is `Permission denied` on `ls`, `chmod`, everything, even as root on the files themselves.

Repair:

```bash
sudo chown sticks:sticks /home/sticks
sudo chmod 755 /home/sticks
sudo usermod -g 1000 sticks
sudo chown -R sticks:sticks /home/sticks
```

Then log out fully and back in. `id` caches credentials at login and will show stale values until you do.

### /tmp/.X11-unix is read-only

```
_XSERVTransmkdir: Mode of /tmp/.X11-unix should be set to 1777
_XSERVTransSocketCreateListener: failed to bind listener
```

WSLg bind-mounts it read-only from the Windows side. `chmod` fails and cannot be made to work.

Cosmetic. The X server falls back to TCP, which is what VNC uses anyway. Local X clients connect over `localhost:1` instead of a unix socket. Marginally slower, functionally the same.

To remove it entirely, disable WSLg in `/etc/wsl.conf`:

```ini
[wsl2]
guiApplications=false
```

Not worth doing unless something actually breaks.

### Bracketed paste mangling multi-line input

MoTTY and some other terminals produce `$'\E[200~ls': command not found` on pasted blocks. Commands silently merge or lose arguments, which is one way a stray `chown` argument ends up somewhere it should not.

```bash
bind 'set enable-bracketed-paste off'
```

Or paste one command at a time. Worth doing before anything involving `sudo chown`.

### Do not run xstartup contents in your shell

Pasting the session script directly into an interactive prompt inherits `DISPLAY` from that terminal. Under an SSH client with X11 forwarding that points at a Windows-side X server, which will not survive a full desktop starting up:

```
XIO: fatal IO error 2 on X server "localhost:11.0"
```

Always launch through `vncserver`, which creates its own display. Never set `DISPLAY` by hand.

---

## Log noise that is expected

All of these are normal in WSL. There is no hardware behind them.

```
Could not find any render nodes
Failed to initialize DRI3 extension
kcm_mouse: Not able to select appropriate backend
kf.bluezqt: Cannot open /dev/rfkill for reading!
Failed to load RealtimeKit property
org.kde.bolt.kded: Couldn't connect to Bolt DBus daemon
xkbcomp: Could not resolve keysym XF86...
xinit: XFree86_VT property unexpectedly has 0 items
```

---

## Tuning later

Once the desktop is stable, try removing `LIBGL_ALWAYS_SOFTWARE=1` and `QT_QUICK_BACKEND=software` from `plasma-vnc.sh`, one at a time. Put back whichever brings the shader errors with it.

`PLASMA_USE_SYSTEMD=0` can also come out if `user@1000.service` starts reliably. Change one thing per attempt. Changing two at once when something breaks is how you end up debugging from scratch.
