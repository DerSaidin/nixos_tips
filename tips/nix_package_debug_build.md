### Debug build of package

```
nix-build -E 'with import <nixpkgs> { }; enableDebugging PACKAGENAME'
```

https://nixos.wiki/wiki/FAQ#How_can_I_compile_a_package_with_debugging_symbols_included.3F

