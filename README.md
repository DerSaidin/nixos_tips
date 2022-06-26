### Cleanup unused packages

#### User

```shell
// List generations
$ nix-env --list-generations
$ ls -l /nix/var/nix/profiles/per-user/$LOGNAME/

// Delete list of specific generations
$ nix-env --delete-generations 3 4 8

// Delete all but latest 5
$ nix-env --delete-generations +5

// Delete all older than 30 days
$ nix-env --delete-generations 30d

$ nix-collect-garbage
```

#### System

```shell
// List generations
# nix-env -p /nix/var/nix/profiles/system --list-generations
// Alternatively
$ ls /nix/var/nix/profiles/

// Delete list of specific generations
# nix-env -p /nix/var/nix/profiles/system --delete-generations 36
```

### get dmesg from previous boot

```shell
sudo journalctl -b -1
```

https://github.com/NixOS/nixpkgs/issues/25820

### NixOS Select Kernel

```
  boot.kernelPackages = pkgs.linuxPackages_latest;

  $ sudo nixos-rebuild boot
  $ sudo reboot
```

https://nixos.wiki/wiki/Choose_your_kernel_on_NixOS

### NixOS Select Kernel Point Release

https://github.com/NixOS/nixpkgs/tree/master/pkgs/os-specific/linux/kernel

Select a major.minor version. Look at the history of its nix file.

https://github.com/NixOS/nixpkgs/commits/master/pkgs/os-specific/linux/kernel/linux-5.4.nix

Select desired version to extract the sha256 value.

https://github.com/NixOS/nixpkgs/commit/e3ba43b82638ed09e711f5f634c6a450dc7ccd53#diff-9d9e3b75979e37116ca5f19d6b76c928

Use the following configuration:
```
  boot.kernelPackages = pkgs.linuxPackagesFor (pkgs.linux_5_4.override {
    argsOverride = rec {
      src = pkgs.fetchurl {
            url = "mirror://kernel/linux/kernel/v5.x/linux-${version}.tar.xz";
            sha256 = "0mxhz3f0ayz0nggndbikp44kx307yxf16qzsv46hni6p8z1ffr0y";
      };
      version = "5.4.41";
      modDirVersion = "5.4.41";
      };
  });
```

For a different version you would need to substitute the following bits of the example above:

- "5_4"
- "v5.x" (there should be an 'x' in this, only change this for different major version)
- "0mxhz3f0ayz0nggndbikp44kx307yxf16qzsv46hni6p8z1ffr0y"
- "5.4.41"

NOTE: I had an old generation on 5.4.41. I rebuild a 5.4.41 generation with this approach. The newly build generation has other differences (many things on the nix-channel have been updated), and only the old generation build of 5.4.41 works for running a certain game. The unstable channel has 5.6 and 5.7 on it now, so I'm guessing other packages in unstable were not impacted by this kernel configuration and are not compatible.

### NixOS Deleting System Generations

```
nix-env --profile /nix/var/nix/profiles/system --list-generations

nix-env --profile /nix/var/nix/profiles/system --delete-generations 10
```

### Package Dependencies

What is in the current system?
```
nix-store -q --tree /run/current-system
```

How does the current system depend on this alacritty build?
```
nix why-depends -a /run/current-system  /nix/store/9l8xyaxk1mwbb9lcij4wf6plv7h91lg9-alacritty-0.4.3
```

Note: The current-system is a symbolic link to some /nix/store/... These commands are all operating on /nix/store/... objects.

### Debug build of package

```
nix-build -E 'with import <nixpkgs> { }; enableDebugging PACKAGENAME'
```

https://nixos.wiki/wiki/FAQ#How_can_I_compile_a_package_with_debugging_symbols_included.3F

### Configure XFCE screensaver

```
xfconf-query -c xfce4-session -p /general/LockCommand -s "i3lock -c 000000" --create -t string
```

### xterm configuration

http://www.linuxquestions.org/questions/debian-26/how-to-change-background-and-fonts-color-in-xterm-156290/page2.html

```
xterm*Background: black
xterm*Foreground: white
```

"xterm" must be all lowercase.

Now in a terminal issue
Code:

`xrdb -merge ~/.Xresources`


### xterm clipboard

https://unix.stackexchange.com/questions/276326/configure-xterm-to-use-ctrlshiftc-ctrlshiftv-for-copy-paste/450506

### xterm fonts

https://wiki.archlinux.org/index.php/Xterm#VT_Fonts_menu

### i3 get window class

```
xprop -spy
*click window*
```

### fsck encrypted disk

https://serverfault.com/questions/375090/using-fsck-to-check-and-repair-luks-encrypted-disk

### Current channel git URL

```shell
nix-instantiate --eval -E '(import <nixpkgs> {}).lib.version'
```

### Steam Fixing

```shell
nix-env --upgrade

nix-env -i steam

steam --reset

rm -rf ~/.local/share/Steam/package/beta
```

This can fix issues after a steam update or a nixos update.

### Update rust

```shell
nix-env -iA nixos.latest.rustChannels.stable.rust
```

### cannot open shared object file: No such file or directory

If you're in a nix-shell environment...

```shell
LD_LIBRARY_PATH=$CMAKE_LIBRARY_PATH
```

### Nix in development

https://ejpcmac.net/blog/about-using-nix-in-my-development-workflow/
  ([alternate link](https://medium.com/@ejpcmac/about-using-nix-in-my-development-workflow-12422a1f2f4c))


### AMD 2200G embedded graphics

(This should no longer be necessary for kernels after about 4.19 or so)

```
  boot.kernelPatches = [
      { name = "amdgpu-config";
        patch = null;
        extraConfig = ''
          DRM_AMD_DC_DCN1_0 y
        '';
      }
  ];
```

https://discourse.nixos.org/t/how-to-get-xserver-working-on-amd-raven-ridge/987

### Firefox use video decode NVIDIA

https://www.askwoody.com/forums/topic/firefox-video-decode-acceleration-how-did-i-miss-this/

`about:config`

Change:
```
media.ffmpeg.dmabuf-textures.enabled = true
media.ffmpeg.vaapi.enabled = true
media.ffvpx.enabled = false
```

Set environment variable:
```
// X11:
MOZ_X11_EGL=1

// For Wayland:
MOZ_ENABLE_WAYLAND=1
```

### Temporarily Open port in nixos firewall

One can edit configuration.nix to permanently open a port in firewall: https://nixos.org/manual/nixos/stable/#sec-firewall

e.g. by one of these options
```
networking.firewall.enable = false;
networking.firewall.allowedTCPPorts = [ 80 443 8080 ];
networking.firewall.allowedTCPPortRanges = [
  { from = 4000; to = 4007; }
  { from = 8000; to = 8100; }
];
```

But what if we want to use `python -m http.server 8080` for 10 min? How can we temporarily open a port?

```
// Need root to even read iptables
# iptables -L INPUT --line-numbers
Chain INPUT (policy ACCEPT)
num  target     prot opt source               destination
1    nixos-fw   all  --  anywhere             anywhere

// -I=insert INPUT rule table, position 1
# iptables -I INPUT 1 -p tcp --dport 8080 -j ACCEPT

// At this point port 8080 can accept connections.

// Rule ordering is important. If we used -A (append) instead of -I, the new rule would be after.
// If the existing nixos-fw is first it will match all traffic first, so the new rule will never apply.
# iptables -L INPUT --line-numbers
Chain INPUT (policy ACCEPT)
num  target     prot opt source               destination
1    ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:http-alt
2    nixos-fw   all  --  anywhere             anywhere

// Delete the rule allowing 8080
# iptables -D INPUT 1

// Back in initial state: everything to firewall for it to figure out what to allow.
# iptables -L INPUT --line-numbers
Chain INPUT (policy ACCEPT)
num  target     prot opt source               destination
1    nixos-fw   all  --  anywhere             anywhere
```
