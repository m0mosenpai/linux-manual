### Update pacman package installation mirrors
- This is needed when a new Arch Installation occurs and one needs to fetch the
latest mirrors before performing any package downloads/ system upgrades.
- URLs can return a "404" while package installation. Updating mirrors can help.
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

### Fix Play/ Pause/ Next/ Prev Keyboard Keys
