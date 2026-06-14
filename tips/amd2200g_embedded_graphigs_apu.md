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
