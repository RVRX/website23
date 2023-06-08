# Intro, and Setting Up WireGuard
This is a guide on setting up a Kill-Switch for WireGuard on Linux, covering some niche network cases. The server used to test this was running Ubuntu 22.04.

If you are instead looking to setup, or have not setup WireGuard, I'd recommend this tutorial by Mullvad: [WireGuard on Linux terminal (easy)](https://mullvad.net/en/help/easy-wireguard-mullvad-setup-linux/).

> When tinkinering, make sure you know how to undo any commands typed into the `PostUp` and `PreDown` sections of your Wiregaurd config! If you typed something wrong and the tunnel connection cannot complete it will leave the config file half-run, without having yet called the PreDown commands. `iptables -S` and `-D` are your friend.
{.is-danger}


# The 'Kill-Switch'
As Wiregaurd defines it, a kill-switch is used to prevent the flow of unencrypted packets through the non-WireGuard interfaces on your device. You might see this option when downloading your WireGuard config from [Mullvad](https://mullvad.net/en), and [IVPN has a tutorial](https://www.ivpn.net/knowledgebase/linux/linux-wireguard-kill-switch/) explaining how to modify a config to implement a kill-switch.

Well someone copied somebody's homework, as they both apply the exact same modification to your WireGuard config (`/etc/wireguard/*.conf`), as do many other sources on the internet:

```bash
PostUp  =  iptables -I OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT && ip6tables -I OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT
PreDown = iptables -D OUTPUT ! -o %i -m mark ! --mark $(wg show  %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT && ip6tables -D OUTPUT ! -o %i -m mark ! --mark $(wg show  %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT
```

These lines should be added after the [Interface] section, but before and [Peer] section of your config. As it turns out, the first chunk of each line above is lifted directly from the `wg-quick` man page, and the second half is an addition to make the kill-switch work over IPv6 via `ip6tables`.

Let's break down the important parts of this kill-switch here...
1. First we have `PostUp` and `PreDown` at the start of each line. Pretty simple, those define script snippets to be executed by bash after the wg tunnel is brought up, and before it tears down respectively.
2. `iptables -I` inserts a rule, with `iptables -D` deleting the rule.
3. The rules are for outbound (`OUTPUT`) connections.
4. `! -o %i` Applies the rule to all network interfaces, except the WG tunnel's interface. `wg-quick` handily expands `%i` to the interface for this config file (usually the name of the file, so `wg0.conf` would correspond to the `wg0` interface.
5. `! --dst-type LOCAL` Applies the rule to all network traffic except "LOCAL". That is, an address that is local to the host we are working on. 127.0.0.1 for example. This **does not include other LAN IPs**.

So what have we done by adding this to our config? Effectively, if the connection to the VPN server ever drops, these iptable rules will prevent traffic from going through any of the other interfaces. A super handy feature, but it has some rather annoying unintended consequences.

## Unintended Consequences

If you are SSH'ed into a machine when you start up a tunnel with these rules set, your terminal will stop responding. Why? Well you just told your server that it isn't able to talk to you anymore, as you are not connected to the server through the WireGuard tunnel's network interface. The fix? Well it depends on your network architecture...

### For Most: IPtables rule exception
If you are most people you ***should be able to*** just change the kill-switch lines to include an exception for your local IP. This is done by adding `! -d 10.0.0.0/24` just before the `-j Reject` in the iptables command (where the IP/Mask is your local subnet). Just like the `! --dst-type LOCAL` indicated not (!) to include localhost IPs in the rule, this adds an exception for destination in the specified IP/subnet.

Like so:
```
PostUp = iptables -I OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL ! -d 10.0.0.0/24 -j REJECT && ip6tables -I OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT
PostDown = iptables -D OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL ! -d 10.0.0.0/24 -j REJECT && ip6tables -I OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT
```

But, as I said earlier, this won't work for everyone.

### What about if I have multiple subnets?: More iptables rules!
If you are like me and have different subnets, you will run into even more issues with this setup. Lucky us!

I ran into this issue as I am running a VPN into my LAN to allow for access to my devices when away. Now this shouldn't cause an issue for you unless devices connected to your home VPN are put on a different subnet, but this is pretty common for security reasons.

#### The Fix...

So, for whatever reason, you've got multiple subnets, and once you turn on the tunnel with the kill-switch, you can no longer talk to devices on the other subnet. Why? As I understand it (and I might not), if `AllowedIPs` in your config is set to all IPs (`0.0.0.0/0, ::/0`) then when you try and talk to a device on your LAN, but on a different subnet, the WireGuard interface will try and send this off to the remote VPN server, where it obviously will not find what it is looking for.

#### Part 1: Adding IP Route
The fix however, I am more confident on. First off, the gateway to the other subnets must be added to the machine's routing table. This will follow the syntax:
```
PreUp = ip route add [Subnet/Mask] via [Gateway IP] dev [normal, non wg interface]
PreDown = ip route del [Subnet/Mask] via [Gateway IP] dev [normal, non wg interface]
```

This should go **BEFORE** the kill-switch PreUp.

So if your machine is on the `10.0.2.0/24` subnet, normally uses the interface `ens18` (check `ip a` for your default interface) and you want to be able to access devices on the `10.0.1.0/24` subnet, you would need to tell the OS to  add `ip route add 10.0.1.0/24 via 10.0.2.1 dev ens18`.

#### Part 2: Adding iptables exception
Now your machine knows how to access these other subnets, but the iptables kill-switch in our config is still going to prevent us from talking to other devices over interfaces other than our WireGuard one. The fix for this is, well, to override those rules. The syntax follows:
```
PostUp = iptables -I OUTPUT -d [subnet/mask] -j ACCEPT
PostDown = iptables -D OUTPUT -d [subnet/mask] -j ACCEPT
```
This should go **AFTER** the kill-switch.


*Make sure to **not add an iptables rule to accept connections sourced from the server connecting to the VPN***. You will be effectively overriding the kill-switch** by saying 'if the source is this subnet', it can speak over the regular interface, as the machine itself is a device on this subnet.*


The final config will look something like:
```
[Interface]
...


# Add route to gateway so the server knows how to access IPs outside of its subnet
# This section has to come before the kill-switch!
## 10.0.1.1 gateway can be accessed through 10.0.2.1
PostUp = ip route add 10.0.1.0/24 via 10.0.2.1 dev ens18
PostDown = ip route del 10.0.1.0/24 via 10.0.2.1 dev ens18
## 192.168.3.1 gateway can be accessed through 10.0.2.1
PostUp = ip route add 192.168.3.0/24 via 10.0.2.1 dev ens18
PostDown = ip route del 192.168.3.0/24 via 10.0.2.1 dev ens18

# Killswitch, prevent all interfaces except wg
PostUp  =  iptables -I OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT && ip6tables -I OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT
PreDown = iptables -D OUTPUT ! -o %i -m mark ! --mark $(wg show  %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT && ip6tables -D OUTPUT ! -o %i -m mark ! --mark $(wg show  %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT

# Add firewall rule to allow access to LAN subnets
## Allow outbound to 10.0.1.0 - 10.0.1.255 subnet (primary)
PostUp = iptables -I OUTPUT -d 10.0.1.0/24 -j ACCEPT
PreDown = iptables -D OUTPUT -d 10.0.1.0/24 -j ACCEPT
## Allow outbound to 192.168.3.0 - 192.158.3.255 subnet (LAN VPN users)
PostUp = iptables -I OUTPUT -d 192.168.3.0/24 -j ACCEPT
PreDown = iptables -D OUTPUT -d 192.168.3.0/24 -j ACCEPT


[Peer]
...
AllowedIPs = 0.0.0.0/0, ::/0
...
```

Now obviously everyone's setup and subnets are a little different, and you will likely have some changes to make to the above. Note my server here is on the `10.0.2.X` subnet, so it's gateway is `10.0.2.1`.


## Caveats of even the best of kill-switches
Kill-switches are not the end-all be-all. There are a few more steps I'd recommend to make your setup whole.

First-off, and the most common, is **interface binding**. Many applications and services offer the ability to bind to a specific interface. That's very handy in our case because our WireGuard interface doesn't even exist when the tunnel is not up, and our kill-switch covers the case where it is up, but not connected. Really. Run `ip a` before and after connecting to a WireGuard VPN.

The other feature I'd recommend making use of is enabling WireGuard to start automatically on boot. To do this, run the following command, replacing `wg0` with your config name: `systemctl enable wg-quick@wg0`. Then, If you then have any other systemd services that start automatically, add `wg-quick@wg0` to the tasks `After=` section. Check out the systemd man-page or a guide for more info there.


# See Also / References
* `iptables` manpage
* `wg` manpage
* `wg-quick` manpage
* https://www.procustodibus.com/blog/2021/03/wireguard-allowedips-calculator/
* https://www.reddit.com/r/WireGuard/comments/yy9hde/ssh_into_machine_from_lan_stops_working_when/
* https://www.reddit.com/r/WireGuard/comments/auuvn5/problem_excluding_private_networks_in_allowedips/
* https://mullvad.net/en/help/easy-wireguard-mullvad-setup-linux/
* https://serverfault.com/questions/163111/allow-traffic-to-from-specific-ip-with-iptables
* https://www.digitalocean.com/community/tutorials/how-to-list-and-delete-iptables-firewall-rules
* https://airvpn.org/forums/topic/50601-kill-switch-settings-for-wireguard-on-ubuntu-20/

*I call this section "See Also" but really its just "See What Websites Firefox History Says I Visited When Debugging This Also"*