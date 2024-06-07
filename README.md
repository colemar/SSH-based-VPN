## Overview
Tunnel IP packets in a SSH channel.

### Steps
1. **Enable tunnelling on server side:**
   - Edit `/etc/ssh/sshd_config` and add the following line:
     ```SSH Config
     PermitTunnel yes
     ```

2. **Change WSL Virtual Switch to External in Hyper-V:**
   - Open PowerShell as administrator and run:
     ```powershell
     Get-NetAdapter -Name * -Physical
     ```
     ![image](https://github.com/colemar/Win10WSL2UbuntuExternalIP/assets/3066000/90f16024-2e61-45cf-b4b2-de957505bde2)
     ```powershell
     Set-VMSwitch WSL -SwitchType External -NetAdapterName "Wired"
     ```
   - If the above commmand fails, try:
     ```powershell
     Set-VMSwitch WSL -SwitchType External -NetAdapterInterfaceDescription "Realtek PCIe GBE Family Controller #2"
     ```
   - If it fails again, do it via Hyper-V manager / Virtual Switch Manager:
![image](https://github.com/colemar/Win10WSL2UbuntuExternalIP/assets/3066000/f7a14b1b-5214-4e78-aa34-eb16a80ae66a)


4. **Modify the WSL Configuration:**
   - **Do not** use `networkingMode = bridged` in the `.wslconfig` file in your user profile directory (`$env:USERPROFILE/.wslconfig`).
   - Instead, use `networkingMode = NAT` (the default) or remove it altogether.

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

