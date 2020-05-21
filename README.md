### get dmesg from previous boot

```shell
sudo journalctl -b -1
```

https://github.com/NixOS/nixpkgs/issues/25820

### AMD 2200G embedded graphics

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

### NixOS Select Kernel

```
  boot.kernelPackages = pkgs.linuxPackages_latest;

  $ sudo nixos-rebuild boot
  $ sudo reboot
```

https://nixos.wiki/wiki/Choose_your_kernel_on_NixOS

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
