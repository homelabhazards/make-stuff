# Headless OBS + DeckLink output on Linux — deployment notes

This documents a working headless OBS "playout" server: OBS Studio running with
**no desktop environment**, controlled entirely over SSH and obs-websocket,
rendering on a GPU and driving multiple SDI outputs from a Blackmagic DeckLink
card. It was built on Ubuntu Server with an Intel Arc Pro (Battlemage) GPU passed
through as an SR-IOV virtual function to a VM, plus a DeckLink Duo 2 (4× SDI).

The specifics (GPU model, card model) will vary; the shape of the setup is the
reusable part.

## Overview of the pieces

1. A GPU that OBS can get a **hardware GL context** on. Software GL (llvmpipe)
   works but is slow and CPU-heavy — avoid it.
2. **`cage`**, a headless Wayland kiosk compositor, to give OBS a GL surface
   without a desktop, X server, or physical display.
3. A **systemd service** running OBS under `cage` as a dedicated service user,
   with seat access via PAM.
4. The **Blackmagic Desktop Video** driver (DKMS) for the DeckLink card.
5. OBS + the **Decklink Output Filter** plugin.
6. **obs-websocket** for remote control, and this tool for binding outputs.

## GPU: get a real GL context headlessly

Confirm the GPU is bound by its kernel driver and exposes a render node:

```bash
lspci -nnk | grep -iA3 'VGA\|Display'
ls -l /dev/dri/by-path/
```

You want a `...-render -> ../renderD128` entry. The service user must be in the
`render` (and usually `video`) group to open it:

```bash
sudo usermod -aG render,video <serviceuser>
```

Verify hardware video works (VA-API) on the render node:

```bash
vainfo --display drm --device /dev/dri/renderD128
```

You should see decode/encode entrypoints for H.264/HEVC/AV1 as applicable. If
`vainfo` "failed to open the given device", it's almost always a group/permission
problem on the render node, not a driver problem.

### cage over SSH: the seat trap

`cage` wants a seat. Launched from an interactive SSH shell it fails with:

```
Timeout waiting session to become active
Failed to start a DRM session
Unable to create the wlroots backend
```

That's expected — there's no VT/seat in an SSH TTY. The fix is to run it from a
**systemd service** with `PAMName=login` and a TTY assigned, which grants the
seat. Don't try to launch `cage` interactively for real use.

Also, on an SR-IOV VF (or any headless GPU with no connected display) the DRM
backend has no connector to light up. Use the **headless wlroots backend** and
point it at the render node:

```ini
Environment=WLR_BACKENDS=headless
Environment=WLR_RENDERER=gles2
Environment=WLR_RENDER_DRM_DEVICE=/dev/dri/renderD128
```

## systemd unit

```ini
[Unit]
Description=Headless OBS (GPU) for DeckLink output
After=multi-user.target DesktopVideoHelper.service
Wants=DesktopVideoHelper.service

[Service]
User=obs
Group=obs
PAMName=login
TTYPath=/dev/tty7
UtmpIdentifier=tty7
UtmpMode=user
StandardInput=tty-fail
StandardOutput=journal
StandardError=journal
Environment=XDG_RUNTIME_DIR=/run/obs
Environment=WLR_BACKENDS=headless
Environment=WLR_RENDERER=gles2
Environment=WLR_RENDER_DRM_DEVICE=/dev/dri/renderD128
RuntimeDirectory=obs
RuntimeDirectoryMode=0700
# Clear any stale crash sentinel so OBS doesn't block on a safe-mode prompt:
ExecStartPre=/bin/sh -c 'rm -f /var/lib/obs/.config/obs-studio/.sentinel/run_* 2>/dev/null || true'
ExecStart=/usr/bin/cage -- obs --minimize-to-tray
Restart=on-failure
RestartSec=5
TimeoutStopSec=15

[Install]
WantedBy=multi-user.target
```

### The crash-sentinel gotcha

OBS drops a file in `~/.config/obs-studio/.sentinel/` on start and clears it on
clean exit. If a run is killed uncleanly, the leftover sentinel makes the next
start pop a **safe-mode recovery dialog** — which, headless, blocks forever with
no way to answer it. The log shows only:

```
Crash or unclean shutdown detected
```

The `ExecStartPre` above wipes stale sentinels so OBS always starts clean. Note
that `cage` is `$MAINPID`, so a plain stop signals cage, which can kill OBS
abruptly; clearing the sentinel on start is the robust fix regardless.

### Confirm the GL adapter

After starting the service, check the OBS log for the adapter line — this is how
you know you're on the GPU and not llvmpipe:

```bash
grep -i "Loading up OpenGL on adapter" \
  "$(ls -t ~/.config/obs-studio/logs/*.txt | head -1)"
```

You want your GPU named there, e.g. `... on adapter Intel ... Arc ... Graphics`,
**not** `llvmpipe`.

## DeckLink driver

Install kernel headers + build tools **before** the Desktop Video package, or the
DKMS module fails silently:

```bash
sudo apt install -y dkms build-essential linux-headers-$(uname -r)
sudo dpkg -i desktopvideo_*.deb || sudo apt install -f -y
lsmod | grep blackmagic
```

Check firmware — some cards (notably the Duo 2) won't appear in tooling with
stale firmware even when the driver is loaded:

```bash
DesktopVideoUpdateTool -l
```

The 4 channels of a Duo 2 appear as `/dev/blackmagic/io0..io3`.

If the DKMS build fails on a newer kernel with a `-Werror=strict-prototypes`
error, add to both `dkms.conf` files under `/usr/src/blackmagic*`:

```
MAKE="make KERNELRELEASE=$kernelver EXTRA_CFLAGS=-Wno-error=strict-prototypes"
```

## OBS + plugin

Install OBS (from the OBS PPA or your distro), then build the Decklink Output
Filter against it:

```bash
sudo apt install -y cmake ninja-build libobs-dev
git clone https://github.com/cg2121/obs-decklink-output-filter
cd obs-decklink-output-filter
cmake --preset ubuntu-x86_64 -DCMAKE_INSTALL_PREFIX=/usr
cmake --build build_x86_64
sudo cmake --install build_x86_64
```

> Watch for a packaging conflict: on some distros `libobs-dev` and a PPA
> `obs-studio` pull different `libobs` versions and apt will try to *remove*
> `obs-studio` to satisfy `libobs-dev`. Reinstall `obs-studio` afterward and keep
> the versions close.

Confirm the plugin loads:

```bash
grep -i "plugin loaded successfully" \
  "$(ls -t ~/.config/obs-studio/logs/*.txt | head -1)"
```

## obs-websocket

Built into OBS 28+. Enable it by editing
`~/.config/obs-studio/plugin_config/obs-websocket/config.json`:

```json
{ "server_enabled": true, "auth_required": true, "server_port": 4455 }
```

OBS generates a random `server_password` on first run; reuse it. It binds
`0.0.0.0:4455` — firewall it to your management network if the box shares a LAN
with untrusted hosts.

## Authoring scenes and binding outputs

Author scenes on a workstation OBS (with a GUI). Name one scene per output
(`io0`, `io1`, ...). Copy the collection JSON to the headless box (an SMB share
over the OBS config dir works well). Then use **this tool** to bind the DeckLink
outputs by scene name — see the top-level README.
