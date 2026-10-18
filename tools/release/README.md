# Making a release

The release archives are built **by hand**, on a machine that has the ROM set: they hold the game's
translated code, so CI never builds or holds one. They land in `build/release/` and are uploaded to
the repository's Releases page by hand:

```sh
make                      # translate the game's code (needs rom/)
make launcher             # the launcher
tools/release/windows.sh  # build/release/Progear-windows-x64.zip
tools/release/linux.sh    # build/release/Progear-linux-x86_64.{tar.gz,AppImage}
gh release create vX.Y.Z build/release/* build/release/SHA256SUMS.txt --title ... --notes ...
```

| Script | Builds | What it holds |
| --- | --- | --- |
| `windows.sh` | `Progear-windows-x64.zip` | the two programs, the five DLLs they need, a `README.txt` and `licenses/` |
| `linux.sh` | `...-linux-x86_64.tar.gz`, `...-linux-x86_64.AppImage` | the two programs, the README and the licence; the AppImage adds the icon, the desktop entry and an `AppRun` |
| `sums.sh` | `SHA256SUMS.txt` | hashes every archive in `build/release/` (run by the others) |
| `make-icon.py` | `progear.png` | the icon, drawn from arithmetic — no artwork from the game is used |

Neither archive holds a ROM set, a key, a BIOS or game artwork: the player brings their own set.
`windows.sh` also checks every DLL the two programs import against the list of ones the archive
carries, so an archive cannot go out missing one.

## Build the Linux programs on an old base

The Linux programs take **glibc from the machine that runs them**, so the older the glibc they are
built against, the more machines can run them. Ubuntu 22.04 (glibc 2.35) is the one to use: it
covers every SteamOS from 3.4 on — the Steam Deck, Game Mode included — and every desktop Linux
since 2022. Built on Ubuntu 24.04 instead, the same sources come out asking for `GLIBC_2.38`
(glibc's C23 `strtol` redirect, which libstdc++ turns on) and then refuse to start on SteamOS 3.5
and on Debian 12.

A chroot is enough; no containers are needed.

```sh
# 1. An old base. The toolkit's CMake asks for 3.28+ and 22.04 ships 3.22, so a newer CMake comes
#    from Kitware's own tarball (it is only a build tool: it does not touch the programs' glibc).
sudo apt-get install -y debootstrap
sudo debootstrap --variant=minbase --components=main,universe jammy /srv/jammy \
     http://archive.ubuntu.com/ubuntu
sudo mkdir -p /srv/jammy/tmp && sudo chmod 1777 /srv/jammy/tmp
sudo cp /etc/resolv.conf /srv/jammy/etc/resolv.conf
for m in proc sys dev dev/pts; do sudo mount --bind /$m /srv/jammy/$m; done

# 2. The compiler and the development packages SDL 3 is built against.
sudo chroot /srv/jammy /bin/bash -c '
  apt-get update -qq && apt-get install -y -qq --no-install-recommends \
    build-essential ninja-build python3 curl file ca-certificates zlib1g-dev \
    libx11-dev libxext-dev libxrandr-dev libxcursor-dev libxi-dev libxss-dev libxtst-dev \
    libxkbcommon-dev libwayland-dev libdecor-0-dev libegl-dev libgl-dev \
    libpulse-dev libasound2-dev libudev-dev libdbus-1-dev libpipewire-0.3-dev
  curl -fsSL -o /tmp/cmake.tar.gz \
    https://github.com/Kitware/CMake/releases/download/v3.31.6/cmake-3.31.6-linux-x86_64.tar.gz
  mkdir -p /opt/cmake && tar -xzf /tmp/cmake.tar.gz --strip-components=1 -C /opt/cmake'

# 3. Build, with the project bind-mounted so the archives land in this tree. work/<game>/gen (the
#    translated code) and rom/ have to be there already: `make` and `make explore` made the first,
#    and the ROM is yours.
sudo mount --bind "$PWD" /srv/jammy/build
sudo chroot /srv/jammy /bin/bash -c '
  export PATH=/opt/cmake/bin:$PATH
  cd /build && tools/release/linux.sh'
```

SDL 3 is built as a static library from a checksum-checked source, so the programs need only the
system's usual libraries. To see what an archive asks for:

```sh
objdump -p build/release-linux/game/progear | sed -n 's/^ *NEEDED *//p' | sort -u
objdump -T build/release-linux/game/progear | grep -o 'GLIBC_[0-9.]*' | sort -uV | tail -n 1
```
