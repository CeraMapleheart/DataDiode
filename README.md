# Simulating a Data Diode with NixOS Containers

This guide will walkthrough creating a unidirectional data flow between two NixOS containers without external hardware. Use VirtualBox on Windows machine to create this setup.

## Prerequisites

1. Windows machine with VirtualBox installed
2. Sufficient RAM (at least 4GB recommended)
3. At least 20GB free disk space

## Step 1: Set Up NixOS in VirtualBox

1. **Download NixOS ISO**:
   - Go to [https://nixos.org/download.html](https://nixos.org/download.html)
   - Download the "Graphical ISO image" for the latest stable version

2. **Create Virtual Machine**:
   - Open VirtualBox and click "New"
   - Name: `NixOS-DataDiode`
   - Type: Linux
   - Version: Other Linux (64-bit)
   - Memory: At least 2048MB
   - Create a virtual hard disk (VDI, dynamically allocated, at least 20GB)

3. **Install NixOS**:
   - Start the VM and select the ISO when prompted
   - Follow the installation instructions (you can use the graphical installer)
   - Make sure to install with sudo privileges enabled

## Step 2: Configure NixOS for Container Setup

After installation, we'll configure the base system to support our data diode containers.

1. **Update your configuration.nix**:
   Edit `/etc/nixos/configuration.nix` to include container support:

   ```nix
   { config, pkgs, ... }:
   {
     boot.enableContainers = true;
     networking.useDHCP = false;
     networking.firewall.enable = true;

     # Disable IP forwarding
     boot.kernel.sysctl."net.ipv4.ip_forward" = 0;
     boot.kernel.sysctl."net.ipv6.conf.all.forwarding" = 0;

     # Required for containers
     systemd.services."systemd-nspawn@" = {
       enable = true;
       serviceConfig = {
         ExecStart = [
           ""
           "${pkgs.systemd}/lib/systemd/systemd-nspawn --quiet --keep-unit --boot --link-journal=try-guest --network-zone=diode %I"
         ];
       };
     };

     environment.systemPackages = with pkgs; [
       iptables
       nftables
       tcpdump
       socat
       curl
     ];
   }
   ```

2. **Apply the configuration**:
   ```bash
   sudo nixos-rebuild switch
   ```

## Step 3: Create the Containers

We'll create two containers: `sender` (ingress) and `receiver` (egress).

1. **Create sender container**:
   ```bash
   sudo nixos-container create sender --config '
     { config, pkgs, ... }:
     {
       networking.firewall.enable = true;
       networking.firewall.allowedTCPPorts = [ 8080 ]; # Proxy port
       networking.interfaces.eth0.ipv4.addresses = [ { address = "10.0.1.2"; prefixLength = 24; } ];
       networking.defaultGateway = { address = "10.0.1.1"; interface = "eth0"; };
     }
   '
   ```

2. **Create receiver container**:
   ```bash
   sudo nixos-container create receiver --config '
     { config, pkgs, ... }:
     {
       networking.firewall.enable = true;
       networking.firewall.allowedTCPPorts = [ 8081 ]; # Proxy port
       networking.interfaces.eth0.ipv4.addresses = [ { address = "10.0.2.2"; prefixLength = 24; } ];
       networking.defaultGateway = { address = "10.0.2.1"; interface = "eth0"; };
       
       services.postgresql = {
         enable = true;
         initialScript = pkgs.writeText "init-db" ""
           CREATE USER diode WITH PASSWORD 'diode';
           CREATE DATABASE diode_data OWNER diode;
         "";
       };
     }
   '
   ```

## Step 4: Configure Network Isolation

We'll create a virtual network bridge to connect the containers with unidirectional flow.

1. **Edit host configuration.nix**:
   Add this to your host's `/etc/nixos/configuration.nix`:

   ```nix
   networking.bridges = {
     diode-br1 = {
       interfaces = [];
     };
     diode-br2 = {
       interfaces = [];
     };
   };

   networking.interfaces = {
     "diode-br1" = {
       ipv4.addresses = [ { address = "10.0.1.1"; prefixLength = 24; } ];
     };
     "diode-br2" = {
       ipv4.addresses = [ { address = "10.0.2.1"; prefixLength = 24; } ];
     };
   };

   networking.nftables = {
     enable = true;
     ruleset = ''
       table inet filter {
         chain input { type filter hook input priority 0; policy drop; }
         chain forward {
           type filter hook forward priority 0; policy drop;
           
           # Allow only from sender to receiver
           iifname "diode-br1" oifname "diode-br2" ct state established,related accept
           iifname "diode-br1" oifname "diode-br2" ct state new accept
           
           # Explicitly block reverse direction
           iifname "diode-br2" oifname "diode-br1" drop
         }
         chain output { type filter hook output priority 0; policy accept; }
       }
     '';
   };
   ```

2. **Apply the configuration**:
   ```bash
   sudo nixos-rebuild switch
   ```

3. **Connect containers to bridges**:
   ```bash
   sudo machinectl start sender
   sudo machinectl start receiver
   sudo machinectl enable sender
   sudo machinectl enable receiver
   ```

## Step 5: Set Up Proxy Services

We'll use `socat` as a simple proxy to demonstrate the data flow.

1. **On sender container**:
   ```bash
   sudo nixos-container root-login sender
   ```

   Inside the sender container:
   ```bash
   cat > /etc/systemd/system/sender-proxy.service <<EOF
   [Unit]
   Description=Sender Proxy Service
   After=network.target

   [Service]
   ExecStart=${pkgs.socat}/bin/socat TCP-LISTEN:8080,fork,reuseaddr TCP:10.0.2.2:8081
   Restart=always
   User=nobody

   [Install]
   WantedBy=multi-user.target
   EOF

   systemctl enable --now sender-proxy.service
   exit
   ```

2. **On receiver container**:
   ```bash
   sudo nixos-container root-login receiver
   ```

   Inside the receiver container:
   ```bash
   cat > /etc/systemd/system/receiver-proxy.service <<EOF
   [Unit]
   Description=Receiver Proxy Service
   After=network.target postgresql.service

   [Service]
   ExecStart=${pkgs.socat}/bin/socat TCP-LISTEN:8081,fork,reuseaddr EXEC:'${pkgs.postgresql}/bin/psql -h localhost -U diode diode_data -c "INSERT INTO data (content) VALUES ($$'"'"'$$`cat`$$'"'"'$$);"',su=diode
   Restart=always
   User=nobody

   [Install]
   WantedBy=multi-user.target
   EOF

   # Create the database table
   psql -U diode diode_data -c "CREATE TABLE IF NOT EXISTS data (id SERIAL PRIMARY KEY, content TEXT, received_at TIMESTAMP DEFAULT NOW());"

   systemctl enable --now receiver-proxy.service
   exit
   ```

## Step 6: Test the Data Diode

1. **Test unidirectional flow (should work)**:
   From your host machine:
   ```bash
   curl -X POST -d "test message" http://10.0.1.2:8080
   ```

   Check if data was received:
   ```bash
   sudo nixos-container root-login receiver
   psql -U diode diode_data -c "SELECT * FROM data;"
   exit
   ```

2. **Test reverse flow (should fail)**:
   Try to connect from receiver to sender:
   ```bash
   sudo nixos-container root-login receiver
   curl http://10.0.1.2:8080
   exit
   ```

   This should timeout or be rejected.

## Step 7: Additional Hardening

Add these to your host's `configuration.nix` for additional security:

```nix
networking.firewall = {
  allowPing = false;
  logReversePathDrops = true;
  extraCommands = ''
    iptables -A INPUT -i diode-br2 -j DROP
    iptables -A OUTPUT -o diode-br2 -j DROP
  '';
};

# Disable all unnecessary services
services.udisks2.enable = false;
services.upower.enable = false;
services.avahi.enable = false;
```

## Final Notes

1. The setup uses simple `socat` proxies - in a production environment, you'd want more robust proxy software.
2. The database is a simple PostgreSQL setup - consider adding more security around it.
3. For monitoring, consider adding logging to track data flow.
4. The nftables rules ensure unidirectional flow at the network level.

To start all containers on boot:
```bash
sudo systemctl enable systemd-nspawn@sender
sudo systemctl enable systemd-nspawn@receiver
```

This gives a complete software-based data diode implementation using NixOS containers with strict unidirectional flow enforced at the network level.
