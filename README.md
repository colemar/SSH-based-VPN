## Overview
Tunnel IP packets in a SSH channel and get a point-to-point VPN.  
Optional: use the VPN to access the Internet from the client using the server public IP address.

### Build the point-to-point VPN
1. **Enable tunneling on server side:**
   - Log in as root or use `sudo`.
   - Edit `/etc/ssh/sshd_config` and add the following line: `PermitTunnel yes`
   - Restart sshd service: `systemctl restart sshd`
   - Create a tunnel device owned by a regular user: `ip tuntap add dev tun1 mode tun user username2`
   - Alternative to above ip command: `openvpn --mktun --dev tun1 --user username2`
   - Check tunnel device ownership: `ip -d link show tun1` (look for "user username2" in third line)
   - Assign IP address in minimal network range and bring up the tun1 interface:
     ```
     ip address add 10.1.0.2/30 dev tun1
     ip link set tun1 up
     ```
     
2. **Enable tunneling on client side:**
   - Log in as root or use `sudo`.
   - Create a tunnel device owned by a regular user: `ip tuntap add dev tun1 mode tun user username1`
   - Alternative to above ip command: `openvpn --mktun --dev tun1 --user username1`
   - Check tunnel device ownership: `ip -d link show tun1` (look for "user username1" in third line)
   - Assign IP address in minimal network range and bring up the tun1 interface:
   ```
   ip address add 10.1.0.1/30 dev tun1
   ip link set tun1 up
   ```

4. **Turn on the ssh tunnel on client side:**
   - Log in as username1.
   - Start a background ssh session: `ssh -f -w 1:1 username2@<server-ip-address> true`
   - Check that the ssh process is still there: `pgrep -u username1 -a ssh`
   
5. **Check connectivity:**
   - On server: `ping 10.1.0.1`
   - On client: `ping 10.1.0.2`

At this point **tun1** on client side along with the **ssh channel** and **tun1** on server side form a **Virtual Private Network** 10.1.0.0/30 (range 10.1.0.0 to 10.1.0.3).

In order to make client's Internet traffic go through the VPN and appear as coming from the server IP, it is necessary to enable routing and masquerating (NAT) on the server and amend the routing table on the client.

### Access the Internet through the VPN

1. **Enable routing and masquerading on server side:**
   - Log in as root or use `sudo`
   - Make a persistent setting (it survives reboot): edit /etc/sysctl.conf and add or modify the line `net.ipv4.ip_forward=1`
   - Avoid having to reboot in order to make the above setting effective: `sysctl -w net.ipv4.ip_forward=1`
   - Alternative to sysctl: `echo 1 > /proc/sys/net/ipv4/ip_forward`
   - Enable masquerading: `iptables -t nat -A POSTROUTING -o <if-name> -j MASQUERADE`. Here \<if-name\> (usually `eth0`) is the physical device attached to the external network (see `ip route list default`).
   - Optional: configure forwarding rules. By default, iptables will forward all traffic unconditionally. You probably want to restrict inbound traffic from the internet, but allow all outgoing:
     ```
     # Allow traffic from internal to external
     iptables -A FORWARD -i tun1 -o eth0 -j ACCEPT
     # Allow returning traffic from external to internal
     iptables -A FORWARD -i eth0 -o tun1 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
     # Drop all other traffic that shouldn't be forwarded
     iptables -A FORWARD -j DROP
     ```
   - Persist these iptables settings after reboot: `apt install iptables-persistent`

2. **Amend routing table on client side:**
   - Log in as root or use `sudo`.
   - Add an host route through the physical network for remote server (this is to protect the above ssh session's connection). Here \<if-name\> (e.g. eth0) and \<gateway-ip-address\> are the same as the existing default route trough the physical network (see `ip route list default`): 
     ```
     ip route add <server-ip-address> via <gateway-ip-address> dev <if-name>
     ```
   - Add global network routes through the tunnel:
     ```
     ip route add 128.0.0.0/1 dev tun1
     ip route add 0.0.0.0/1 dev tun1
     ```
     A single `ip route add default dev tun1` also seems to work, altough having two more specific routes as above ensures they override the existing default route through the physical network.

3. **Check that Internet access is through the VPN:**
   - Log in on client as regular user.
   - Please note that 1.1.1.1 is a public routable IP address assigned to Cloudflare.
   - `ip route get 1.1.1.1` → `1.1.1.1 dev tun1 ...`
   - `ping 1.1.1.1` → `64 bytes from 1.1.1.1: icmp_seq=1 ...`
   - `curl ipinfo.io/ip` → \<server-ip-address\>
   
