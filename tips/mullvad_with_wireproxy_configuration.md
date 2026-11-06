## Mullvad with Wireproxy

### Enable Mullvad

All the following configuration is in `/etc/nixos/configuration.nix`

```
  # Mullvad VPN
  # https://github.com/NixOS/nixpkgs/issues/113589
  networking.firewall.checkReversePath = "loose";
  networking.wireguard.enable = true;
  services.mullvad-vpn.enable = true;
```

Create wireproxy user, and setup two wireproxy services running simultaneously for
multiple configurations.

The services reference `${pkgs.wireproxy}` directly, so wireproxy does not need to be in
`environment.systemPackages`.

```
  users.groups.wireproxy = {};
  users.users.wireproxy = {
    isSystemUser = true;
    group = "wireproxy";
  };

  # WireGuard configs contain private keys, so they live outside the nix store
  # (root:root 0400 in /etc/wireproxy) and are handed to the service via LoadCredential.
  systemd.services.wireproxy_aaa = {
    description = "AAA WireGuard client that exposes itself as a socks5 proxy";
    wantedBy = [ "multi-user.target" ];
    after = [ "network.target" ];
    serviceConfig = {
      LoadCredential = "wg.conf:/etc/wireproxy/wireguard_aaa.conf";
      ExecStart = "${pkgs.wireproxy}/bin/wireproxy -c \${CREDENTIALS_DIRECTORY}/wg.conf";
      Restart = "always";
      User = "wireproxy";
      Group = "wireproxy";
    };
  };

  systemd.services.wireproxy_bbb = {
    description = "B WireGuard client that exposes itself as a socks5 proxy";
    wantedBy = [ "multi-user.target" ];
    after = [ "network.target" ];
    serviceConfig = {
      LoadCredential = "wg.conf:/etc/wireproxy/wireguard_bbb.conf";
      ExecStart = "${pkgs.wireproxy}/bin/wireproxy -c \${CREDENTIALS_DIRECTORY}/wg.conf";
      Restart = "always";
      User = "wireproxy";
      Group = "wireproxy";
    };
  };
```

You may want to replace the aaa bbb with whatever country you chose as the exit.

#### Why LoadCredential

The WireGuard configs contain private keys. Anything referenced from `configuration.nix` as a
path (e.g. `environment.etc."...".source = ./wireguard_aaa.conf;`) gets copied into
`/nix/store`, which is world readable. So instead:

* The configs are installed manually to `/etc/wireproxy`, owned `root:root` with mode `0400`
  (directory `0700`), so they never enter the nix store and only root can read them.
* `LoadCredential` makes systemd (running as root) read the file at service start and expose
  a private copy at `$CREDENTIALS_DIRECTORY/wg.conf`, readable only by that service's
  `wireproxy` user.
* The `\${CREDENTIALS_DIRECTORY}` is escaped in nix so it reaches systemd literally, and
  systemd expands it at runtime.

### Wireguard config setup

Go to https://mullvad.net/en/account/wireguard-config and generate a configuration.
Download it and save it (somewhere temporary, e.g. the current directory) as
`wireguard_aaa.conf`.

Because we have multiple configurations, we will change where the proxy is serving locally.

Add the following to the end of `wireguard_aaa.conf`:

```
# Socks5 creates a socks5 proxy on your LAN, and all traffic would be routed via wireguard.
[Socks5]
BindAddress = 127.0.0.1:25344

# Socks5 authentication parameters, specifying username and password enables
# proxy authentication.
#Username = user1
# Avoid using spaces in the password field
#Password = pass1

# http creates a http proxy on your LAN, and all traffic would be routed via wireguard.
[http]
BindAddress = 127.0.0.1:25345
```

Generate the second one for the second exit as `wireguard_bbb.conf`.
You must use a separate device key for each one.

Add the following to the end of `wireguard_bbb.conf`:

```

# Socks5 creates a socks5 proxy on your LAN, and all traffic would be routed via wireguard.
[Socks5]
BindAddress = 127.0.0.1:25354

# Socks5 authentication parameters, specifying username and password enables
# proxy authentication.
#Username = user2
# Avoid using spaces in the password field
#Password = pass2

# http creates a http proxy on your LAN, and all traffic would be routed via wireguard.
[http]
BindAddress = 127.0.0.1:25355
```

Note that the ports are different.

### Install the Wireguard configs

Install both configs into `/etc/wireproxy`, readable only by root:

```
sudo install -d -m 0700 /etc/wireproxy
sudo install -m 0400 -o root -g root wireguard_aaa.conf wireguard_bbb.conf /etc/wireproxy
```

Then remove the downloaded copies (they contain the private keys):

```
rm wireguard_aaa.conf wireguard_bbb.conf
```

Do not put these files in `/etc/nixos` or reference them from `configuration.nix`, or they
will be copied into the world-readable nix store.

Apply the configuration with `sudo nixos-rebuild switch`. If you later change a config in
`/etc/wireproxy`, restart the service so it reloads the credential:

```
sudo systemctl restart wireproxy_aaa wireproxy_bbb
```

### Firefox Create Proxy Profile

```
firefox -p
```

Create a new firefox profile. Called Proxy1.

Settings > Proxy Settings > Manual Proxy Configuration

If you set a SOCKS username and password, it will not work -- firefox cannot configure SOCKS authentication.

HTTP Proxy:  127.0.0.1   Port: 25345

(matching `[http] BindAddress` in `wireguard_aaa.conf`)

Go to mullvad.net to test

### Run firefox profile

```
man wofi.7
```

> drun - searches $XDG_DATA_HOME/applications and $XDG_DATA_DIRS/applications for desktop files and allows them to be run by selecting them.

`echo $XDG_DATA_DIRS` shows that this includes:

* `/home/$LOGNAME/.nix-profile/share` (exists, but is read only)
* `/home/$LOGNAME/.local/state/nix/profile/share` (does not exist -- its free realestate)

```
mkdir -p /home/$LOGNAME/.local/state/nix/profile/share/applications
cd /home/$LOGNAME/.local/state/nix/profile/share/applications
```

Make a desktop application thingy file, based on `/run/current-system/sw/share/applications/firefox.desktop`

```
[Desktop Entry]
Actions=new-private-window;new-window;profile-manager-window
Categories=Network;WebBrowser
Exec=firefox -P Proxy1
GenericName=Web Browser
Icon=firefox
MimeType=text/html;text/xml;application/xhtml+xml;application/vnd.mozilla.xul+xml;x-scheme-handler/http;x-scheme-handler/https
Name=Firefox Proxy1
StartupNotify=true
StartupWMClass=firefox_proxy1
Terminal=false
Type=Application
Version=1.5

[Desktop Action new-private-window]
Exec=firefox -P Proxy1 --private-window %U
Name=New Private Window

[Desktop Action new-window]
Exec=firefox -P Proxy1 --new-window %U
Name=New Window
```

Key changes:

* Add `-P Proxy1` to ALL exec
* Change `Name` (added ` Proxy1`) 
* Remove `profile-manager-window action`

###  Firefox Proxy Profile Further Setup

#### Theme

I like to pick a theme with distinct colors, so it is obvious if I'm using the VPN or not.

#### WebRTC Leaks

https://mullvad.net/en/check

If WebRTC leaks, a simple fix is by disabling it:

`media.peerconnection.enabled =	false`

#### Geo Location to match VPN exit

Go to about:config

* `geo.enabled = true`
* `geo.provider.network.url = data:application/json,{"location": {"lat": 40.7590, "lng": -73.9845}, "accuracy": 27000.0}`

  * Use Google Maps to find coordinates to roughly match your exit node.

* `geo.provider.testing = true`
* `geo.provider.use_geoclue = false`

May need to clear cookies or restart firefox.

#### Privacy Setting  (optional)

Settings > Privacy & Security

Browser Privacy = Strict / Custom

Always use private browsing mode = true

HTTPS-ONly mode = all windows
