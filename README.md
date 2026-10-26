# linux-cyberark-psm-remoteapp

Open CyberArk PSM RemoteApp sessions (the `.rdp` files the PVWA hands out) from
Linux on a Wayland compositor (niri), with working mouse, popups, keyboard focus
and clipboard.

CyberArk only officially supports Linux through the HTML5 gateway. This is
the workaround for when that isn't enabled.

## Usage

Download the `.rdp` from the PVWA and run:

```sh
psm              # uses the newest *.rdp in ~/Downloads
psm file.rdp     # or a specific file
```

Or click the `.rdp` in the browser. `psm.desktop` registers it as the
handler and runs it in a kitty window so the logs are visible.

Each `psm` run starts its own nested Weston. Closing the app, the Weston
window or the terminal ends everything.

## Install

Requirements (nix):

```sh
nix profile add nixpkgs#freerdp nixpkgs#weston nixpkgs#clipnotify
```

Plus `wl-clipboard`, `xclip`, `ss` (iproute2) and `kitty` for the desktop entry.
FreeRDP must be **3.32.0 or newer** (first release with RemoteApp in the SDL client).

```sh
ln -s "$PWD/psm" ~/.local/bin/psm
cp psm.desktop ~/.local/share/applications/
xdg-mime default psm.desktop application/x-rdp application/x-remmina
update-desktop-database ~/.local/share/applications
```

On Ubuntu `.rdp` is typed `application/x-remmina` because Remmina claims the
extension, so both types are mapped. That means `.remmina` files will also open
with psm. To undo that, run
`xdg-mime default org.remmina.Remmina.desktop application/x-remmina`.

If you have the apt `freerdp2-*` / `freerdp3-wayland` clients installed, remove
only those three packages. Removing the `libfreerdp*` libraries also removes
remmina-plugin-rdp, gnome-connections and gnome-remote-desktop.

## What the .rdp file contains

```
full address:s:<psm host>
username:s:localhost\PSM@<guid>          one-time token, no password
alternate shell:s:PSM@<guid>
remoteapplicationmode:i:1
remoteapplicationprogram:s:||PSMInitSession
remoteapplicationname:s:PSM-<Component>
enablecredsspsupport:i:0                 no NLA, so /sec:rdp or /sec:tls
```

The token is single-use and expires fast, so every attempt needs a fresh download.

## The command that works

```sh
DISPLAY=<weston X display> SDL_VIDEODRIVER=x11 sdl-freerdp \
  /v:<host> /u:'localhost\PSM@<guid>' /p:'' \
  /app:program:'||PSMInitSession',name:PSM-<Component> \
  /sec:rdp /cert:tofu +clipboard -gfx /drive:Documents,~/Documents
```

`/drive:` shares that one folder with the server. It shows up as
`\\tsclient\Documents` and as "Documents on <your hostname>" in the remote
app's file dialogs, if the PSM policy allows drive redirection. FreeRDP marks
dot-files as hidden on the Windows side.

## What we learned (in the order we hit it)

1. **Strip `\r`.** `.rdp` files are CRLF. A trailing `\r` on the token makes
   PSM reject it, and you land on the PSM server's Windows logon screen. Don't
   log in there with your AD account, because that bypasses PSM.

2. **`/p:''` is required.** FreeRDP only sends the auto-logon flag when a
   password is set. With an empty password PSM logs in as `<domain>\PSMConnect`
   by itself.

3. **Full desktop + alternate shell doesn't work.** `/shell:PSM@<guid>`
   connects, but PSM logs the session off after about 4 s
   (`ERRINFO_LOGOFF_BY_USER`). These components only allow RemoteApp. An admin
   could set `DisableRemoteApp=Yes` per connection component.

4. **Use a RemoteApp-capable client.** `wlfreerdp3` has no RemoteApp code and
   is deprecated. `sdl-freerdp` gained RemoteApp in 3.32.0
   ([FreeRDP#13133](https://github.com/FreeRDP/FreeRDP/pull/13133), still
   marked experimental). `xfreerdp` (X11) also has it.

5. **Native Wayland RemoteApp is broken in practice.** On Wayland the client
   can't know where its windows are, so mouse clicks land at the wrong remote
   coordinates. Popups open as separate compositor windows that don't get
   keyboard focus (only Esc works). Multi-monitor offsets make it worse
   ([FreeRDP#13507](https://github.com/FreeRDP/FreeRDP/pull/13507), fixed for
   3.32.1). The workaround of maximising the remote window (Alt+Space, X) and
   fullscreening the local one works for one window only.

6. **niri's XWayland (xwayland-satellite) isn't enough either.** With
   `SDL_VIDEODRIVER=x11` or `xfreerdp` directly on niri, windows drew once and
   then froze. Edge stayed blank white, and the PSM login splash stayed stuck on
   screen.

7. **Nested Weston fixes everything.** `weston --xwayland` runs as one window
   on niri and gives RemoteApp a real stacking X window manager. Window
   positions, popups, focus and input all work, for both Edge and DBeaver. The
   white Edge window was the client's problem, not the server's.

8. **`SDL_VIDEODRIVER=x11` stays.** `WAYLAND_DISPLAY` still points at niri, so
   without it SDL3 skips Weston and goes native Wayland on niri. Pointing it
   at Weston's Wayland socket doesn't help, because the X11 backend is what
   knows window positions.

9. **cage and gamescope are not alternatives.** cage forces every window
   fullscreen or maximised and positions dialogs itself. gamescope shows one
   focused window. RemoteApp needs the compositor to honour server-given
   positions.

10. **Weston has its own clipboard.** It doesn't share a clipboard with the
    host compositor. `psm` bridges text both ways:
    - host → Weston: `wl-paste --watch xclip -t text/plain`
    - Weston → host: `clipnotify` (event-driven, so the RDP server isn't polled)
      then `xclip -o | wl-copy`, skipped when the text is already on the host
      clipboard, which prevents ping-pong.

11. **xclip must offer `text/plain`, not its default `UTF8_STRING`.**
    sdl-freerdp picks the first text mime the owner offers, and winpr can't
    convert `UTF8_STRING`. The symptom is
    `No synthesizer for format CF_RAW --> CF_UNICODETEXT` and pasting does
    nothing.

12. **Ctrl+C inside the remote app is blocked** (PSM/app keyboard hook).
    Right-click → Copy works. Ctrl+V works.

13. **`set -euo pipefail` gotcha.** A `grep` with no match inside `$(...)`
    exits the script, and the EXIT trap then kills Weston 2 ms after start
    (`caught signal 15`). Hence the `|| true` in the display-detection loop.

14. **Don't start Weston with `--fullscreen` on niri.** With it, windows inside
    Weston (even `xclock -update 1`) only redraw while the mouse moves. Without
    it, Weston keeps committing frames to niri on its own.

## Keyboard

The remote session may use a different layout (e.g. Danish). On Danish, `\` is
AltGr + the key left of Backspace and `@` is AltGr + 2. `/kbd:layout:0x409`
forces US, if PSM respects the client layout.

## niri notes

- Weston sets no app-id. Match it with `title="^Weston Compositor"`.
- niri can't put floating windows above a focused fullscreen window
  ([niri#2798](https://github.com/niri-wm/niri/discussions/2798)).
  Maximize-to-edges is the only alternative.

## Not done / ideas

- The clipboard bridge carries text only (no images or files).
- If the PAM admins enable the HTML5 gateway or set `DisableRemoteApp=Yes`, a
  plain full-desktop session would work without any of this.
