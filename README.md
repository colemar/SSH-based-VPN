## Overview
Tunnel IP packets in a SSH channel and get a point-to-point VPN.  
Optional: use the VPN to access the Internet from the client using the server public IP address.

### Build the point-to-point VPN
1. **Enable tunneling on server side:**
   - Log in as root or use `sudo`.
   - Edit `/etc/ssh/sshd_config` and add the following line:
     ```SSH Config
     PermitTunnel yes
     ```
   - Restart sshd service: `systemctl restart sshd`
   - Create a tunnel device owned by a regular user: `openvpn --mktun --dev tun1 --user username2`
   - Check tunnel device ownership: `ip -d link show tun1` (look for "user username2" in third line)
   - Assign IP address in minimal network range and bring up the tun1 interface:
   ```
   ip address add 10.1.0.2/22 dev tun1
   ip link set tun1 up
   ```
     
2. **Enable tunneling on client side:**
   - Log in as root or use `sudo`.
   - Create a tunnel device owned by a regular user: `openvpn --mktun --dev tun1 --user username1`
   - Check tunnel device ownership: `ip -d link show tun1` (look for "user username1" in third line)
   - Assign IP address in minimal network range and bring up the tun1 interface:
   ```
   ip address add 10.1.0.1/22 dev tun1
   ip link set tun1 up
   ```

4. **Turn on the ssh tunnel on client side:**
   - Log in as username1.
   - Start a background ssh session: `ssh -f -w 1:1 username2@<server-ip-address> true`
   - Check that the ssh process is still there: `pgrep -u username1 -a ssh`
   
5. **Check connectivity:**
   - On server: `ping 10.1.0.1`
   - On client: `ping 10.1.0.2`

At this point **tun1** on client side along with the **ssh channel** and **tun1** on server side form a **Virtual Private Network** 10.1.0.0/22 (range 10.1.0.0 to 10.1.0.3).

In order to make client's Internet traffic go through the VPN and appear as coming from the server IP, it is necessary to enable routing and masquerating (NAT) on the server and amend the routing table on the client.

### Access the Internet through the VPN

1. **Enable routing and masquerading on server side:**
   - Login as root or use `sudo`
   - Make a permanent setting (it survives reboot): edit /etc/sysctl.conf and add or modify the line `net.ipv4.ip_forward=1`
   - Avoid having to reboot: `sysctl -w net.ipv4.ip_forward=1`
   - Alternative to above command: `echo 1 > /proc/sys/net/ipv4/ip_forward`
   - Enable masquerading: `iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE`

2. **Amend routing table on client side:**
   - Login as root or use `sudo`.
   - Add host route for remote server (this is to protect the above ssh session's connection):
   ```
   ip route add <server-ip-address> via <gateway-ip-address> dev <if-name>
   ```
   where \<if-name\> (most likely `eth0`) and \<gateway-ip-address\> are the same as the existing default route (see `ip route list default`).
   - Add global network routes through the tunnel:
   ```
   route add 127.0.0.0/1 dev tun1
   route add 0.0.0.0/1 dev tun1
   ```
   
