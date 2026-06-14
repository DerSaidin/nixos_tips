### Temporarily Open port in nixos firewall

One can edit configuration.nix to permanently open a port in firewall: https://nixos.org/manual/nixos/stable/#sec-firewall

e.g. by one of these options
```
networking.firewall.enable = false;
networking.firewall.allowedTCPPorts = [ 80 443 8080 ];
networking.firewall.allowedTCPPortRanges = [
  { from = 4000; to = 4007; }
  { from = 8000; to = 8100; }
];
```

But what if we want to use `python -m http.server 8080` for 10 min? How can we temporarily open a port?

```
// Need root to even read iptables
# iptables -L INPUT --line-numbers
Chain INPUT (policy ACCEPT)
num  target     prot opt source               destination
1    nixos-fw   all  --  anywhere             anywhere

// -I=insert INPUT rule table, position 1
# iptables -I INPUT 1 -p tcp --dport 8080 -j ACCEPT

// At this point port 8080 can accept connections.

// Rule ordering is important. If we used -A (append) instead of -I, the new rule would be after.
// If the existing nixos-fw is first it will match all traffic first, so the new rule will never apply.
# iptables -L INPUT --line-numbers
Chain INPUT (policy ACCEPT)
num  target     prot opt source               destination
1    ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:http-alt
2    nixos-fw   all  --  anywhere             anywhere

// Delete the rule allowing 8080
# iptables -D INPUT 1

// Back in initial state: everything to firewall for it to figure out what to allow.
# iptables -L INPUT --line-numbers
Chain INPUT (policy ACCEPT)
num  target     prot opt source               destination
1    nixos-fw   all  --  anywhere             anywhere
```
