## Overview
Tunnel IP packets in a SSH channel.

### Steps
1. **Enable tunneling on server side:**
   - Log in as root or use `sudo`.
   - Edit `/etc/ssh/sshd_config` and add the following line:
     ```SSH Config
     PermitTunnel yes
     ```
   - Restart sshd service: `systemctl restart sshd`
   - Create a tunnel device owned by a regular user: `openvpn --mktun --dev tun1 --user username2`
   - Check tunnel device ownership: `ip -d link show tun1` (look for "user username2" in third line)
   - Assign network range and bring up the tun1 interface:
   ```
   ip address add 10.1.0.2/24 dev tun1
   ip link set tun1 up
   ```
     
2. **Enable tunneling on client side:**
   - Log in as root or use `sudo`.
   - Create a tunnel device owned by a regular user: `openvpn --mktun --dev tun1 --user username1`
   - Check tunnel device ownership: `ip -d link show tun1` (look for "user username1" in third line)
   - Assign network range and bring up the tun1 interface:
   ```
   ip address add 10.1.0.1/24 dev tun1
   ip link set tun1 up
   ```

4. **Turn on the ssh tunnel on client side:**
   - Log in as username1.
   - Start a background session: `ssh -f -w 1:1 username2@<server-ip-address> true`
   - Check that the ssh process is still there: `pgrep -u username1 -a ssh`
   
5. **Adjust route table on client side:**
   - Login as root or use `sudo`.
   - Add host route for remote server (this is to protect the above ssh session):
   ```
   ip route add <server-ip-address> via <gateway-ip-address> dev <if-name>
   ```
   where <if-name> is most likely `eth0` and <gateway-ip-address> is the same as the existing default route (see `ip route list default`).
   - Add global network routes through the tunnel:
   ```
   route add 127.0.0.0/1 dev tun1
   route add 0.0.0.0/1 dev tun1
   ```

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

