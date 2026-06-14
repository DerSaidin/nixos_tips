### Clean up System generations

```shell
// List generations
# nix-env -p /nix/var/nix/profiles/system --list-generations
// Alternatively
$ ls /nix/var/nix/profiles/

// Delete list of specific generations
# nix-env -p /nix/var/nix/profiles/system --delete-generations 36
```

