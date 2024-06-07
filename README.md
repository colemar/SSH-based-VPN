## Overview
Tunnel IP packets in a SSH channel.

### Steps
1. **Enable tunnelling on server side:**
   - Login as root or use `sudo`.
   - Edit `/etc/ssh/sshd_config` and add the following line:
     ```SSH Config
     PermitTunnel yes
     ```
   - Restart sshd service: `systemctl restart sshd`
   - Create a tunnel device owned by a regular user: `openvpn --mktun --dev tun1 --user username2`
   - Check tunnel device ownership: `ip -d link show tun1` (look for "user username2" in third line)
     
2. **Enable tunnelling on client side:**
   - Login as root or use `sudo`.
   - Create a tunnel device owned by a regular user: `openvpn --mktun --dev tun1 --user username1`
   - Check tunnel device ownership: `ip -d link show tun1` (look for "user username1" in third line)
   
3. **Adjust route table on client side:**
   - Login as root or use `sudo`.
   - Add host route for remote server: `ip route add <server-ip-address> via <default-gateway-ip-address> dev <if-name> ` where <if-name> is most likely `eth0`.
   - Add global network routes.

6. **Enable systemd in the WSL Distribution and flush ip configuration at boot:**
   - Edit the `/etc/wsl.conf` file in your WSL distribution and add the following lines:
     ```plaintext
     [boot]
     systemd=true
     # remove NAT related ip configuration applied by WSL at boot
     command = ip address flush dev eth0
     [network]
     generateResolvConf = false
     ```

7. **Configure Network Addressing:**
   - For dynamic address configuration, ensure the following is present in `/etc/systemd/network/10-eth0.network`:
     ```plaintext
     [Match]
     Name=eth0
     [Network]
     DHCP=yes
     ```
   - For static address configuration, use:
     ```plaintext
     [Match]
     Name=eth0
     [Network]
     Address=192.168.x.xx/24
     Gateway=192.168.x.x
     DNS=192.168.x.x
     ```

8. **Link systemd Resolv.conf:**
   - Create a symbolic link to link resolv.conf from systemd:
     ```bash
     ln -sfv /run/systemd/resolve/resolv.conf /etc/resolv.conf
     ```

9. **Verification:**
   - Restart the WSL 2 instance and verify the network configuration with:
     ```bash
     ip addr show eth0
     ip route
     ```

