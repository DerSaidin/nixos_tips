### Nix Package Dependencies

What is in the current system?
```
nix-store -q --tree /run/current-system
```

How does the current system depend on this alacritty build?
```
nix why-depends -a /run/current-system  /nix/store/9l8xyaxk1mwbb9lcij4wf6plv7h91lg9-alacritty-0.4.3
```

Note: The current-system is a symbolic link to some /nix/store/... These commands are all operating on /nix/store/... objects.

