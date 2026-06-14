### Nix Clean up User generations

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
