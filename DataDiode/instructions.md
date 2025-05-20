Step-by-step instructions:

1. **Virtual Machine Setup**
- Download NixOS ISO from official site
- Create new VM in VirtualBox with:
  - 2GB+ RAM
  - 20GB+ dynamic storage
  - Other Linux (64-bit) type
- Install NixOS using graphical installer
  - Enable sudo privileges during setup

2. **Base System Configuration**
- Edit `/etc/nixos/configuration.nix` to:
  - Enable container support
  - Disable DHCP and IP forwarding
  - Configure firewall
  - Add required packages (iptables, socat, etc.)
- Apply changes: `sudo nixos-rebuild switch`

3. **Container Creation**
- Create two containers:
  ```bash
  sudo nixos-container create sender
  sudo nixos-container create receiver
  ```
- Configure each container with:
  - Static IP addresses (10.0.1.2 and 10.0.2.2)
  - Specific firewall rules
  - PostgreSQL on receiver side

4. **Network Isolation Setup**
- Create two virtual bridges:
  - diode-br1 (10.0.1.1/24)
  - diode-br2 (10.0.2.1/24)
- Configure nftables rules to:
  - Allow traffic only from sender→receiver
  - Explicitly block reverse traffic
  - Set default policy to DROP

5. **Proxy Services Configuration**
- On sender container:
  - Set up socat to listen on 8080
  - Forward to receiver's 8081 port
- On receiver container:
  - Set up socat to listen on 8081
  - Insert received data into PostgreSQL
  - Create database table structure

6. **Testing Procedure**
- Verify unidirectional flow:
  ```bash
  curl -X POST -d "test" http://10.0.1.2:8080
  ```
- Check database for received data
- Verify reverse blocking:
  ```bash
  # From receiver container
  curl http://10.0.1.2:8080
  ```

7. **Security Hardening**
- Disable ping responses
- Add additional iptables drops
- Disable unnecessary services
- Enable logging of blocked packets

8. **Persistence Setup**
- Enable containers to start at boot:
  ```bash
  sudo systemctl enable systemd-nspawn@sender
  sudo systemctl enable systemd-nspawn@receiver
  ```

Key Technical Details:
- Network namespaces isolate container networks
- nftables rules enforce one-way flow at Layer 3
- Separate bridges prevent direct communication
- socat provides simple application proxy
- PostgreSQL demonstrates data storage

Troubleshooting Tips:
1. If connections fail:
- Check container status: `machinectl list`
- Verify bridge IPs: `ip addr show`
- Test basic connectivity between containers

2. If database isn't receiving:
- Check PostgreSQL logs in receiver
- Verify socat processes are running
- Test direct DB connection locally

3. For firewall issues:
- Check nftables rules: `nft list ruleset`
- Look for blocked packets in logs
- Temporarily enable logging for debugging

This implementation provides a true software-based data diode with:
- Physical layer simulation through network isolation
- Network layer enforcement via firewall rules
- Application layer proxying
- Storage layer on the secure side
- Complete prevention of reverse communication channels