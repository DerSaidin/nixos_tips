### Steam Fixing

```shell
nix-env --upgrade

nix-env -i steam

steam --reset

rm -rf ~/.local/share/Steam/package/beta
```

This can fix issues after a steam update or a nixos update.

