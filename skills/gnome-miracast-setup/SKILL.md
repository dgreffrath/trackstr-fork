---
name: gnome-miracast-setup
description: Activate Miracast (Wi-Fi Display) casting from GNOME to smart TVs and wireless displays using gnome-network-displays. Use when: (1) casting a GNOME desktop to a TV or Miracast adapter, (2) fixing "No Miracast channels available", (3) checking whether a Wi-Fi chip supports Wi-Fi Direct (P2P).
---

# GNOME Miracast Setup

Miracast casting on GNOME runs through the **Network Displays** app (`gnome-network-displays`). It creates a Wi-Fi Direct (P2P) connection through NetworkManager + wpa_supplicant and streams video/audio with PipeWire/GStreamer. GNOME Settings has no built-in Miracast toggle — the app is the supported path.

## When to Use This Skill

- Casting a GNOME desktop to a Miracast sink (smart TV in "Screen Mirroring" mode, Microsoft Wireless Display Adapter, Roku/Android-TV mirroring)
- Troubleshooting "No Miracast channels available" or a TV that never appears
- Verifying that a Wi-Fi chip/driver supports Wi-Fi Direct before configuring anything

## Verify the machine supports it

Run these checks first; all must pass:

```bash
iw phy | grep -E '\* P2P-(client|GO|device)'  # driver supports Wi-Fi Direct
nmcli device | grep -i p2p                    # NM exposes a wifi-p2p virtual device
nmcli --version                               # NetworkManager >= 1.16
wpa_supplicant -v | head -1                   # wpa_supplicant >= 2.9 (iwd does NOT work)
```

If the `iw phy` output has no P2P lines, the Wi-Fi chip/driver lacks Wi-Fi Direct (common with Broadcom and Ralink hardware, including many laptop cards by those vendors). The fix is hardware: a Wi-Fi Direct-capable USB adapter (e.g., mt7921u or rtl8821cu based) or a wired HDMI mirroring dongle.

## Installation

```bash
# Debian/Ubuntu
sudo apt-get install gnome-network-displays

# Fedora
sudo dnf install gnome-network-displays

# Arch (AUR)
yay -S gnome-network-displays
```

Flatpak builds of "Network Displays" are also on Flathub. Pull the network-displays app from the distro package where possible so PipeWire/GStreamer codecs (openh264 or x264; fdkaacenc/faac/avenc_aac for AAC audio) resolve automatically.

## Activate (per casting session)

1. **Put the sink into mirroring mode first.** TVs only announce themselves on Miracast when enabled (TV remote: "Screen Share", "Sources → Screen Mirroring", "Smart View", etc. — naming varies by brand).
2. Launch the app:

```bash
gnome-network-displays &
```

It can also be started from the Activities overview ("Network Displays").

3. Select the TV in the list → **Connect**. On Wayland a screen-share portal prompt appears; pick the monitor to cast.
4. For audio, in **Settings → Sound** choose the output device named **Network-Displays** (Miracast audio requires using this sink; some receivers only accept uncompressed LPCM, which is not supported).

## Fix "No Miracast channels available"

The most common blocker is an unset regulatory domain (`country 00: DFS-UNSET`), which leaves no legal channel for the P2P group owner:

```bash
iw reg get                       # if it prints "country 00: DFS-UNSET", fix regdom
sudo iw reg set US               # runtime fix; use YOUR ISO 3166 country code
echo 'options cfg80211 ieee80211_regdom=US' | sudo tee /etc/modprobe.d/regdom.conf  # persists across reboots
nmcli radio wifi off && nmcli radio wifi on   # re-apply, then restart the app
```

## Troubleshooting

**TV never appears:** keep the TV in mirroring mode while the search runs; power-cycle the TV (many TVs stop advertising after the Wi-Fi radio idles — pull power for a minute); restart the app (it rescans on launch).

**Chromecast sinks:** those are discovered over the LAN via mDNS, not P2P — TV and PC must be on the same network, and `avahi-daemon` should be active with UDP 5353 not firewalled.

**Connect fails during channel negotiation:** wpa_supplicant before 2.10 had P2P channel-list bugs; upgrade to 2.10+ (2.11 is fine).

**Wayland/X11:** works under Wayland via the PipeWire screen-cast portal; an X11 frame-grab fallback exists on X11. A toast "No compatible sink" usually means the receiver rejected the H.264/AAC profile — try the other codec env knobs documented in the project README (`NETWORK_DISPLAYS_H264_ENC`, `NETWORK_DISPLAYS_AAC_ENC`).

**Verbose logging:** `G_MESSAGES_DEBUG=all gnome-network-displays` prints discovery, P2P, and stream negotiation details.
