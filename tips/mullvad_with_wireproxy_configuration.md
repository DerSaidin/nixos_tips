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

Install wireproxy

```
  environment.systemPackages = with pkgs; [
    wireproxy
  ];
```

Create wireproxy user, and setup two wireproxy service running simultaneously for two configurations 

```
  users.groups.wireproxy = {};
  users.users.wireproxy = {
    isSystemUser = true;
    group = "wireproxy";
  };

  environment.etc = {
    "wireproxy/wireguard_1.conf" = {
        source = ./wireguard_1.conf;
    };
    "wireproxy/wireguard_2.conf" = {
        source = ./wireguard_2.conf;
    };
  };

  systemd.services.wireproxy_1 = {
    description = "1 WireGuard client that exposes itself as a socks5 proxy";
    wantedBy = [ "multi-user.target" ];
    after = [ "network.target" ];
    serviceConfig = {
      ExecStart = "${pkgs.wireproxy}/bin/wireproxy -c /etc/wireproxy/wireguard_1.conf";
      Restart = "always";
      User = "wireproxy";
      Group = "wireproxy";
    };
  };
  
  systemd.services.wireproxy_2 = {
    description = "2 WireGuard client that exposes itself as a socks5 proxy";
    wantedBy = [ "multi-user.target" ];
    after = [ "network.target" ];
    serviceConfig = {
      ExecStart = "${pkgs.wireproxy}/bin/wireproxy -c /etc/wireproxy/wireguard_2.conf";
      Restart = "always";
      User = "wireproxy";
      Group = "wireproxy";
    };
  };
```

### Wireguard config

The `source = ./wireguard_1.conf;` above means we should have
`/etc/nixos/wireguard_1.conf` to provide the content for
`/etc/wireproxy/wireguard_1.conf`

Go to https://mullvad.net/en/account/wireguard-config to generate
`/etc/nixos/wireguard_1.conf`.

Download the configuration to and save at `/etc/nixos/wireguard_1.conf`.

Because we have multiple configurations, we will change where the proxy is serving locally.

Add this to the end of `wireguard_1.conf`:

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

Generate the second one (use a separate device key for each one). 

Add this to the end of `wireguard_2.conf`:

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

### Firefox Create Proxy Profile

```
firefox -p
```

Create a new firefox profile. Called Proxy1.

Settings > Proxy Settings > Manual Proxy Configuration

If you set a SOCKS username and password, it will not work -- firefox cannot configure SOCKS authentication.

HTTP Proxy:  127.0.0.1   Port: 25345

(matching `[http] BindAddress` in `wireguard_1.conf`)

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
