### Nix Clean up System generations

```shell
# List generations
sudo nix-env -p /nix/var/nix/profiles/system --list-generations
# Alternatively
ls /nix/var/nix/profiles/

# Delete list of specific generations
sudo nix-env -p /nix/var/nix/profiles/system --delete-generations 36
```

### Issue: `nixos-rebuild switch` fails due to no space on /boot

`OSError: [Errno 28] No space left on device: '/nix/store/q3rf1fg9ssf9yldpb1aa3rjmf4grz715-initrd-linux-6.18.55/initrd' -> '/boot/EFI/nixos/q3rf1fg9ssf9yldpb1aa3rjmf4grz715-initrd-linux-6.18.55-initrd.efimy4in621.tmp.'`

```shell
# Check what’s taking up space
df -h /boot
du -sh /boot/EFI/nixos/* | sort -h

# Delete old system generations
# This keeps the last 3 generations. Use old instead of +3 to keep only the current one.
sudo nix-env -p /nix/var/nix/profiles/system --list-generations
sudo nix-env --delete-generations +3 --profile /nix/var/nix/profiles/system

# Remove their files from /boot without writing anything new
sudo /run/current-system/bin/switch-to-configuration boot

# Garbage-collect the store
sudo nix-collect-garbage

# Attempt nixos-rebuild again
sudo nixos-rebuild switch
```
