### NixOS Select Kernel

```
  boot.kernelPackages = pkgs.linuxPackages_latest;

  $ sudo nixos-rebuild boot
  $ sudo reboot
```

https://nixos.wiki/wiki/Choose_your_kernel_on_NixOS

### NixOS Select Kernel Point Release

See currently available kernel versions:

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

