My experience of trying out difference Linux distributions over the years has made
me want to document everything I have done to make a new installation as usable as
possible. This was the first time that I tried a hands-on/ minimal
distribution like Arch and committed to using it as a daily-driver for my work.

This here is a collection of every issue that I found interesting and worth
talking about. This is not meant to (nor can it ever) replace the official
[Arch Wiki](https://wiki.archlinux.org/title/Main_page) in any way.
I just sometimes found myself scouring the Internet to find answers to some of the
more weirder issues and this is an attempt to gather all that information in one place.

## Setup
- **Device:** Dell XPS-15 9570
- **OS:** Arch Linux (Kernel 6.14)
- **Window Manager:** i3 (X11)

## Troubleshooting

### Update pacman package installation mirrors
I needed to do this in 2 main situations:
- When a new Arch Installation occurs and one needs to fetch the
latest mirrors before performing any package downloads/ system upgrades.
- Sometimes packages would fail to install due to the download URLs returning a "404"
```
sudo reflector --verbose --latest 5 --country <country> --age 6 --sort rate --save /etc/pacman.d/mirrorlist
```
- Finally, sync databases/ perform a system upgrade with `yay -Syyu`

### Delete yay and pacman generated artifacts
- _pacman_ and _yay_ don't clear cache by default and can store many old versions
of packages:
    - `yay -Ps` to get the status of installed packages and cache disk space usage
    - `yay -Yc` removes unused dependencies
    - `yay -Sc` aggressively cleans cache for both _aur_ and _pacman_ packages
- For an intelligent/ less-aggressive solution:
    1. `sudo pacman -S pacman-contrib` to get `paccache` utility
    2. `paccache -d` to dryrun a cache purge
    3. `paccache -r` to delete old cache (saving last 3 versions by default)
    4. Can be [set-up as a hook](https://wiki.archlinux.org/title/Pacman#Cleaning_the_package_cache) that runs on every `pacman` transaction

### Control media playback via keyboard
- Get the [playerctl package](https://github.com/altdesktop/playerctl)
- `playerctl next`, `playerctl previous`, `playerctl play-pause` will control playback
- If you use Xorg, `XF86AudioNext`, `XF86AudioPrev`, `XF86AudioPlay` keys can then be
bound to the above commands to control playback via keyboard

### Improve microphone audio quality
While my laptop microphone would sound clear, I found it to be super sensitive in most
situations - picking up a lot of background noise and making it borderline unusable.
I installed and started using [this noise suppression config](https://github.com/werman/noise-suppression-for-voice) after a suggestion from Reddit and I have had a
much better experience ever since.

### Stop Discord/ Zoom/ Meet from changing microphone gain
It took me ages to figure out why my microphone volume kept getting changed without
my knowledge, causing my voice to either disappear completely or become way too loud
and distorted. This made it impossible for me to take any meetings or calls on linux
without using earphones that had built-in microphone.

Turns out, WebRTC (the protocol that all of these voice/ video conferencing apps use)
allows applications to automatically control the mic gain. Fortunately, we can toggle
AGC (Automatic Gain Control) off in some applications individually, as well as tell
our audio server (PipeWire in my case) to block AGC changes by any of these applications:

- [Disable audio post-processing in Firefox](https://wiki.archlinux.org/title/Firefox/Tweaks#Disable_WebRTC_audio_post_processing)
- Disable automatic volume adjustment in Zoom:
    1. Goto `Zoom -> Settings -> Audio`
    2. In the `Microphone` section, unselect `"Automatically adjust microphone volume"`
- Disable automatic gain control in Discord:
    1. Goto `Discord -> Settings -> Voice and Video`
    2. Under `Voice Processing`, toggle off `"Automatic Gain Control"`
- Lastly, I added a rule in PipeWire (Under `pulse.rules` in `.config/pipewire/pipewire-pulse.conf.d/pipewire-pulse.conf`) to block changes to the Input Source volume. The exact commit
with this change be found in my dotfiles [here](https://github.com/m0mosenpai/dotfiles/commit/8a5d37ab53439afee4647b9a357e1044fd18d193#diff-e6066ea2fbd370146b5ae4c6d39d3ad20069ed2aef44d5277e190b3dc81d818d)
    ```
    {
        matches = [
            { application.process.binary = "Discord" }
            { application.process.binary = "chrome" }
            { application.process.binary = "firefox" }
        ]
        actions = { quirks = [ block-source-volume ] }
    }
    ```
### autorandr fails to move all windows and switch to a single-display setup on unplugging an external display
Using [xrandr](https://wiki.archlinux.org/title/Xrandr) every single time you connect
or disconnect an external display can be very annoying. [autorandr](https://github.com/phillipberndt/autorandr) saves your display config in a profile and also automatically switches to it when you connect to it.

However, I found that disconnecting my display did not make it automatically switch
back to a single-display setup. This would result in a ghost i3 desktop to exist,
even if I moved all my windows back to my single laptop display.

Turns out, this was because once you disconnect your external display, `autorandr`
tries to find a saved profile matching the config of just your single-display setup.
So a neat little trick to fix this is to create a new profile in `autorandr` with
just your laptop display. This way when you disconnect the external display, `autorandr`
is always able to find a config to fall back to.

### Wi-fi drops connection intermittently
Another issue that plagued me for the longest time was my inconsistent wi-fi connection.

At the time of installing Arch, I had set up my wi-fi via `iwd`. Some time in the
future, I also ended up installing `wpa_supplicant` and `dhcpd` (for DHCP).
And then someone on the internet said that `NetworkManager` is the best so I ended up
installing that too. And soon after, I started experiencing connection drops.

What I didn't understand then was the fact that `iwd`, `wpa_supplicant` and `dhcpd`
were all separate backends to authenticate and connect to wireless networks. And
since all of them did the same thing at the same time, it was causing a race condition -
with the dormant one trying to replace the active one.`NetworkManager` on the other
hand was a client that would use either `wpa_supplicant` or `iwd` as one of the backends
to connect to a network. It also had the most amount of features/ support built-in
(one wouldn't need to install `dhcpd` separately to enable DHCP for example. You could enable it
throught `NetworkManager` itself). [This table](https://wiki.archlinux.org/title/Network_configuration#Network_managers) is pretty useful in understanding the differences between all
of them.

Therefore, I needed to decide a _client_ that I would
use to connect to a network and a _backend_ that would then handle the connection.

So I ended up going with `NetworkManager` + `iwd` combo. I uninstalled `wpa_supplicant`
and `dhcpd`, making sure to move any custom network configuration over to `iwd`.
And from then on, I strictly used `NetworkManager` to connect to all wi-fi networks
and voila, no connection drops ever.

### Changing regions does not automatically update the Time Zone

I was visiting my home country some time back and was not happy to find out that
my PC's time did not automatically update. One would ideally want their computer
to be able to figure out where you are (when it connects to the internet) and update
the time according to your timezone.

`NetworkManager` allows you to dispatch scripts based on when, where or which network
you connect to. So I found someone who was kind enough to already have made such a
script. I placed it in `/etc/NetworkManager/dispatcher.d` and made sure the dispatcher
service was running and enabled on system startup:

```bash
#!/bin/bash
iface=$1
action=$2

if [[ $iface != lo && $action == up ]]; then
    tz=$(tzupdate -s 1 -p 2>/dev/null)
    if [[ -n $tz && -r /usr/share/zoneinfo/$tz ]]; then
        timedatectl set-timezone $tz
    fi
fi
```

Arch wiki also suggests [an alternate script](https://wiki.archlinux.org/title/System_time#Update_timezone_every_time_NetworkManager_connects_to_a_network) to do this.
