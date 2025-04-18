# K3s Ubuntu Lab Setup Guide - Part 1

## Introduction

Welcome to the K3s Ubuntu Lab setup guide! This document provides step-by-step instructions for deploying a highly available K3s cluster using Ubuntu 22.04 VMs, managed by a dedicated service VM (`k8-svc`) acting as a router, load balancer, and service provider for the internal cluster network.

**Goal:** To create a functional K3s environment accessible from your main LAN (`192.168.1.0/24`), leveraging an existing Pi-hole DNS (`192.168.1.53`) and custom certificates for the `komebacklabs.lan` domain.

**Prerequisites:**

*   An Ubuntu 22.04 VM designated as `k8-svc`.
*   Access to your hypervisor to configure VM network interfaces.
*   SSH access to the `k8-svc` VM.
*   Basic familiarity with Linux command line and networking concepts.

---


## Part 1: Preparing the `k8-svc` VM

The `k8-svc` VM is the cornerstone of this setup, bridging your main LAN and the internal K3s network. We'll start by configuring its network interfaces and enabling routing capabilities.

### Step 1.1: Configure Network Interfaces

The `k8-svc` VM requires two network interfaces: one connected to your main LAN and one connected to the isolated network where your K3s nodes will reside.

**Objective:** Assign static IP addresses to both interfaces and ensure they are configured correctly for their respective networks, making the configuration persistent.

**Procedure:**

1.  **Identify Interface Names:** Log in to your `k8-svc` VM via SSH and identify the names of your two network interfaces using the command:
    ```bash
    ip addr show
    ```
    Look for interfaces like `eth0`, `eth1`, `ens34`, `ens35`, etc. Note which one is currently connected to your main LAN (it likely has an IP in the `192.168.1.x` range if it got one via DHCP initially) and which one is intended for the internal K3s network. For this guide, we'll use your specified names:
    *   `ens34`: Connects to Main LAN (`192.168.1.0/24`)
    *   `ens35`: Connects to Internal K3s Network (`192.168.10.0/24`)

2.  **Disable Cloud-Init Network Configuration (IMPORTANT):** You observed that `/etc/netplan/50-cloud-init.yaml` is managed by cloud-init, meaning manual changes might be overwritten. To prevent this and make our static configuration persistent, we must first disable cloud-init's network management feature.
    ```bash
    # Create the directory if it doesn't exist
    sudo mkdir -p /etc/cloud/cloud.cfg.d/

    # Create the config file to disable network management by cloud-init
    sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg > /dev/null <<EOF
    network: {config: disabled}
    EOF
    ```
    *   **Explanation:** This command creates a configuration file (`99-disable-network-config.cfg`) that tells cloud-init to stop managing network settings on subsequent boots. Your manual Netplan configurations will now persist.

3.  **Configure Static IPs (Netplan):** Now that cloud-init won't interfere, we can safely configure Netplan. Edit the existing YAML file in `/etc/netplan/` (which you identified as `50-cloud-init.yaml`) to contain the desired static configuration for *both* interfaces (`ens34` and `ens35`).

    **Action:** Replace the *entire content* of `/etc/netplan/50-cloud-init.yaml` with the following:

    ```yaml
    # /etc/netplan/50-cloud-init.yaml
    network:
      version: 2
      ethernets:
        ens34: # Interface connected to Main LAN
          dhcp4: no
          addresses:
            - 192.168.1.89/24 # Static IP for k8-svc on Main LAN
          routes:
            - to: default
              via: 192.168.1.1 # Your Main LAN Router/Gateway IP
          nameservers:
            addresses: [192.168.1.53] # Your Pi-hole IP
        ens35: # Interface connected to Internal K3s Network
          dhcp4: no
          addresses:
            - 192.168.10.1/24 # Static IP for k8-svc on Internal K3s Net (Gateway for K3s nodes)
          # No gateway route needed here
    ```

    *   **Explanation:**
        *   `dhcp4: no`: Disables DHCP for these interfaces as we're setting static IPs.
        *   `addresses`: Defines the static IP address and subnet mask (using CIDR notation, `/24` is equivalent to `255.255.255.0`).
        *   `routes`: For the LAN interface (`ens34`), we define the default gateway (`192.168.1.1`) for outbound internet traffic.
        *   `nameservers`: For the LAN interface (`ens34`), we explicitly set the Pi-hole (`192.168.1.53`) as the DNS server for the `k8-svc` VM itself. The internal interface (`ens35`) doesn't need DNS configured here; we'll set up a forwarder later.

4.  **Apply Netplan Configuration:**
    ```bash
    sudo netplan apply
    ```
    This command applies the changes. You might lose SSH connection briefly if you were connected via an IP that changed; reconnect using the new static IP (`192.168.1.89`).

5.  **Verify Configuration:** Check the IP addresses again:
    ```bash
    ip addr show ens34
    ip addr show ens35
    ```
    Verify the routes:
    ```bash
    ip route show
    ```
    You should see the default route via `192.168.1.1` associated with `ens34`.

### Step 1.2: Enable IP Forwarding

For the `k8-svc` VM to act as a router between the two networks, we need to enable IP forwarding in the kernel.

**Objective:** Allow the Linux kernel to forward packets received on one interface to another.

**Procedure:**

1.  **Edit sysctl configuration:**
    ```bash
    sudo nano /etc/sysctl.conf
    ```
2.  **Uncomment IP Forwarding:** Find the line `#net.ipv4.ip_forward=1` and remove the leading `#` to uncomment it:
    ```ini
    net.ipv4.ip_forward=1
    ```
3.  **Save and Close:** Save the file (Ctrl+O in nano, then Enter) and exit (Ctrl+X).
4.  **Apply the setting immediately:**
    ```bash
    sudo sysctl -p
    ```
    This reloads the sysctl settings from the file. You should see `net.ipv4.ip_forward = 1` printed to the console.

### Step 1.3: Configure NAT (Masquerading)

Now that forwarding is enabled, we need to configure Network Address Translation (NAT), specifically masquerading. This allows K3s nodes on the private `192.168.10.0/24` network to access the main LAN and the internet using the `k8-svc` VM's main LAN IP address (`192.168.1.89`).

**Objective:** Rewrite the source IP address of packets leaving the `k8-svc` VM (from the internal network) so that return traffic knows how to get back.

**Procedure:**

1.  **Install iptables-persistent:** This utility helps save and restore iptables rules across reboots.
    ```bash
    sudo apt update
    sudo apt install iptables-persistent -y
    ```
    During installation, it might ask if you want to save current IPv4 and IPv6 rules. You can say Yes to both, even though we haven't added our rule yet (we'll save it next).

2.  **Add the Masquerade Rule:** Use `iptables` to add the rule.
    ```bash
    sudo iptables -t nat -A POSTROUTING -o ens34 -s 192.168.10.0/24 -j MASQUERADE
    ```
    *   **Explanation:**
        *   `-t nat`: Specifies the NAT table.
        *   `-A POSTROUTING`: Appends the rule to the POSTROUTING chain (applied just before packets leave the machine).
        *   `-o ens34`: Matches packets going *out* of the specified interface (`ens34`).
        *   `-s 192.168.10.0/24`: Matches packets with a *source* IP address from the internal K3s network.
        *   `-j MASQUERADE`: Jumps to the MASQUERADE target, which automatically replaces the source IP with the IP of the outgoing interface (`ens34`).

3.  **Save the Rules:** Use `iptables-persistent` to save the current rules so they load on boot:
    ```bash
    sudo netfilter-persistent save
    ```

4.  **Verify (Optional):** You can view the current NAT rules:
    ```bash
    sudo iptables -t nat -L POSTROUTING -v -n
    ```
    You should see your MASQUERADE rule listed.

---

This completes the initial network setup for the `k8-svc` VM. It can now route traffic between the two networks and allow internal nodes to reach external resources. The next part will cover installing and configuring the DHCP server.
---

## Part 2: Configure DHCP Server (`isc-dhcp-server`)

The `k8-svc` VM will run a DHCP server to automatically assign IP addresses and network configuration to the K3s nodes joining the internal `192.168.10.0/24` network. We'll use the standard `isc-dhcp-server`.

**Objective:** Install and configure `isc-dhcp-server` to provide static IP leases based on MAC addresses for K3s nodes.

**Procedure:**

1.  **Install DHCP Server:**
    ```bash
    sudo apt update
    sudo apt install isc-dhcp-server -y
    ```

2.  **Prepare Configuration File:** The main configuration file is `/etc/dhcp/dhcpd.conf`. We need to define our internal subnet and specify the static IP assignments for our K3s nodes.

    **Action:** Replace the entire content of `/etc/dhcp/dhcpd.conf` with the following configuration. **Crucially, you must edit this configuration later to replace the placeholder MAC addresses with the actual MAC addresses of your K3s VMs.**

    ```conf
    # --- Start of content for /etc/dhcp/dhcpd.conf ---

    # Option definitions common to all supported networks...
    option domain-name "komebacklabs.lan"; # Optional: Sets the domain name for clients
    option domain-name-servers 192.168.10.1; # Provide k8-svc as the DNS server

    default-lease-time 600;
    max-lease-time 7200;

    # If this DHCP server is the official DHCP server for the local
    # network, the authoritative directive should be uncommented.
    authoritative;

    # Use this to send dhcp log messages to a different log file (optional)
    # log-facility local7;

    # Definition for the internal K3s network
    subnet 192.168.10.0 netmask 255.255.255.0 {
      range 192.168.10.50 192.168.10.99; # Optional: Dynamic range if needed for other clients
      option subnet-mask 255.255.255.0;
      option broadcast-address 192.168.10.255;
      option routers 192.168.10.1; # Gateway for K3s nodes is k8-svc internal IP

      # Static assignments for K3s nodes
      # --- REPLACE MAC ADDRESSES BELOW ---
      host k3s-cp-1 {
        hardware ethernet 00:11:22:33:44:51; # Replace with actual MAC for CP1
        fixed-address 192.168.10.11;
      }

      host k3s-cp-2 {
        hardware ethernet 00:11:22:33:44:52; # Replace with actual MAC for CP2
        fixed-address 192.168.10.12;
      }

      host k3s-w-1 {
        hardware ethernet 00:11:22:33:44:61; # Replace with actual MAC for Worker1
        fixed-address 192.168.10.21;
      }
      # Add more 'host' entries if you have more workers
    }

    # --- End of content for /etc/dhcp/dhcpd.conf ---
    ```

    *   **Explanation:**
        *   `option domain-name-servers 192.168.10.1;`: Tells clients to use the `k8-svc` VM itself for DNS queries.
        *   `authoritative;`: Declares this DHCP server as the official one for this subnet.
        *   `subnet ... { ... }`: Defines the scope for the `192.168.10.0/24` network.
        *   `range ...;`: Defines a pool of IPs for dynamic assignment (optional, useful for temporary clients).
        *   `option routers 192.168.10.1;`: Sets the default gateway for clients to be the `k8-svc` internal IP.
        *   `host k3s-cp-1 { ... }`: Defines a static reservation. When a client with the specified `hardware ethernet` (MAC address) requests an IP, it will always receive the `fixed-address` listed.

3.  **Specify Listening Interface:** We need to tell the DHCP server to only listen on the internal network interface.
    ```bash
    sudo nano /etc/default/isc-dhcp-server
    ```
    Find the line starting with `INTERFACESv4=""` and modify it to specify your internal interface:
    ```ini
    INTERFACESv4="ens35"
    ```
    Save and close the file.

4.  **Enable and Start the Service:**
    ```bash
    # Start the service
    sudo systemctl start isc-dhcp-server

    # Enable the service to start automatically on boot
    sudo systemctl enable isc-dhcp-server

    # Check the status to ensure it started correctly
    sudo systemctl status isc-dhcp-server
    ```
    If the status shows errors, check the system logs (`journalctl -u isc-dhcp-server`) or configuration syntax (`sudo dhcpd -t`). Common issues include typos in `dhcpd.conf` or incorrect interface names.

---

At this point, the DHCP server is running and ready to assign IPs to your K3s nodes once they boot up and you've updated the `dhcpd.conf` with their correct MAC addresses. The next part will cover configuring DNS forwarding.
---

## Part 3: Configure DNS Forwarding (`systemd-resolved`)

The K3s nodes will be configured (via DHCP) to send their DNS queries to the `k8-svc` VM's internal IP (`192.168.10.1`). We need to configure the `systemd-resolved` service on `k8-svc` to listen on this internal IP address and forward those queries to your main LAN's Pi-hole DNS server (`192.168.1.53`).

**Objective:** Configure `systemd-resolved` to listen on the internal interface (`192.168.10.1`) in addition to its default loopback listener, and forward DNS queries to the upstream Pi-hole server.

**Background:** By default, `systemd-resolved` runs a "stub" listener only on `127.0.0.53` for local processes. Setting `DNSStubListener=no` (as previously attempted) disables *all* listeners, which is not what we want. The correct approach for Ubuntu 22.04 (systemd v249+) is to keep the default stub listener enabled (`DNSStubListener=yes`) and add our internal IP using `DNSStubListenerExtra=`.

**Procedure:**

1.  **Check `systemd-resolved` Status:** First, ensure the service is running.
    ```bash
    sudo systemctl status systemd-resolved
    ```
    It should be active (running). If you previously modified `/etc/systemd/resolved.conf` to set `DNSStubListener=no`, revert that change (either set it back to `yes` or comment out the line to use the default `yes`).

2.  **Create Drop-in Configuration File:** Instead of modifying the main `/etc/systemd/resolved.conf` file (which could be overwritten by package updates), we'll create a drop-in configuration file. This is the recommended way to customize systemd services.
    ```bash
    # Create the configuration directory if it doesn't exist
    sudo mkdir -p /etc/systemd/resolved.conf.d

    # Create and populate the drop-in file
    sudo tee /etc/systemd/resolved.conf.d/k8s-forwarding.conf > /dev/null <<'EOF'
    [Resolve]
    DNS=192.168.1.53
    DNSStubListener=yes
    DNSStubListenerExtra=192.168.10.1
    EOF
    ```
    *   **Explanation:**
        *   `mkdir -p ...`: Creates the directory for drop-in configuration files if it's not already there.
        *   `tee ...`: Creates the file `k8s-forwarding.conf` inside that directory.
        *   `[Resolve]`: Specifies that these settings apply to the main Resolve section.
        *   `DNS=192.168.1.53`: Confirms that `systemd-resolved` should forward queries it can't answer locally to your Pi-hole.
        *   `DNSStubListener=yes`: Explicitly keeps the default stub listener on `127.0.0.53` active (this is the default behavior).
        *   `DNSStubListenerExtra=192.168.10.1`: This is the key directive. It tells `systemd-resolved` to open *additional* listening sockets (UDP and TCP) on port 53 specifically on the `192.168.10.1` IP address.

3.  **Restart `systemd-resolved`:** Apply the new configuration by restarting the service.
    ```bash
    sudo systemctl restart systemd-resolved
    ```

4.  **Verify Listeners:** Check which addresses `systemd-resolved` is now listening on.
    ```bash
    sudo ss -tulnp | grep systemd-resolve
    ```
    You should now see `systemd-resolved` listening on **both** `127.0.0.53` (the default stub) and `192.168.10.1:53` (the extra listener for your K3s network) for UDP and TCP. The output might look similar to this (PID/FD will vary):
    ```
    udp   UNCONN  0      0      192.168.10.1:53     0.0.0.0:*    users:(("systemd-resolve",pid=1234,fd=12))
    udp   UNCONN  0      0      127.0.0.53:53     0.0.0.0:*    users:(("systemd-resolve",pid=1234,fd=10))
    tcp   LISTEN  0      4096   192.168.10.1:53     0.0.0.0:*    users:(("systemd-resolve",pid=1234,fd=13))
    tcp   LISTEN  0      4096   127.0.0.53:53     0.0.0.0:*    users:(("systemd-resolve",pid=1234,fd=11))
    ```

5.  **Verify DNS Forwarding (Optional but Recommended):** Test that DNS resolution works correctly through the new listener.
    ```bash
    # Test from k8-svc itself, querying the internal IP
    dig @192.168.10.1 google.com +short

    # If possible, test from a K3s node (once they are up)
    # ssh k3s-cp-1 'dig google.com +short'
    ```
    Both commands should successfully return an IP address for google.com, confirming that queries sent to `192.168.10.1` are being forwarded correctly to the Pi-hole (`192.168.1.53`) via `systemd-resolved`.

---

With DNS forwarding correctly configured using `DNSStubListenerExtra=`, the `k8-svc` VM is now properly set up to handle DNS requests from the internal K3s network and forward them to your Pi-hole. The next part is installing and configuring HAProxy.
---

## Part 4: Install and Configure HAProxy

HAProxy will serve as the single point of entry from your main LAN (`192.168.1.0/24`) into your K3s cluster. It will listen on the `k8-svc` VM's LAN IP (`192.168.1.89`) and intelligently forward traffic to the appropriate backend K3s nodes based on the requested port (API, HTTP, HTTPS).

**Objective:** Install HAProxy and configure it to load balance the K3s API and route ingress traffic.

**Procedure:**

1.  **Install HAProxy:**
    ```bash
    sudo apt update
    sudo apt install haproxy -y
    ```

2.  **Prepare Configuration File:** The main configuration is `/etc/haproxy/haproxy.cfg`. We will define frontends (listening points) and backends (target servers).

    **Action:** Replace the entire content of `/etc/haproxy/haproxy.cfg` with the following configuration. This configuration assumes the K3s node IPs assigned in the DHCP section (`192.168.10.11`, `192.168.10.12`, `192.168.10.21`).

    ```cfg
    # --- Start of content for /etc/haproxy/haproxy.cfg ---

    global
        log /dev/log    local0
        log /dev/log    local1 notice
        chroot /var/lib/haproxy
        stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
        stats timeout 30s
        user haproxy
        group haproxy
        daemon

        # Default SSL material locations
        # ca-base /etc/ssl/certs
        # crt-base /etc/ssl/private

        # Default ciphers to use on SSL-enabled listening sockets.
        # ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
        # ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
        # ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

    defaults
        log     global
        mode    tcp # Default to TCP mode for Layer 4 load balancing
        option  tcplog
        option  dontlognull
        timeout connect 5000
        timeout client  50000 # 50 seconds
        timeout server  50000 # 50 seconds
        # errorfile 400 /etc/haproxy/errors/400.http
        # errorfile 403 /etc/haproxy/errors/403.http
        # errorfile 408 /etc/haproxy/errors/408.http
        # errorfile 500 /etc/haproxy/errors/500.http
        # errorfile 502 /etc/haproxy/errors/502.http
        # errorfile 503 /etc/haproxy/errors/503.http
        # errorfile 504 /etc/haproxy/errors/504.http

    #---------------------------------------------------------------------
    # Frontend: K3s API Server
    #---------------------------------------------------------------------
    frontend k3s_api_frontend
        bind 192.168.1.89:6443 # Listen on LAN IP, port 6443
        mode tcp
        option tcplog
        default_backend k3s_api_backend

    #---------------------------------------------------------------------
    # Backend: K3s API Server
    #---------------------------------------------------------------------
    backend k3s_api_backend
        mode tcp
        option tcp-check # Use TCP checks for health
        balance source # Use source IP hashing for persistence
        server k3s-cp-1 192.168.10.11:6443 check # Point to CP1 internal IP
        server k3s-cp-2 192.168.10.12:6443 check # Point to CP2 internal IP

    #---------------------------------------------------------------------
    # Frontend: HTTP Ingress (Port 80)
    #---------------------------------------------------------------------
    frontend http_ingress_frontend
        bind 192.168.1.89:80 # Listen on LAN IP, port 80
        mode tcp
        option tcplog
        default_backend http_ingress_backend

    #---------------------------------------------------------------------
    # Backend: HTTP Ingress (Port 80)
    #---------------------------------------------------------------------
    backend http_ingress_backend
        mode tcp
        balance source
        # Initially point to worker nodes where Traefik/Ingress might run.
        # Later, if specific services get dedicated MetalLB IPs, you might
        # create separate backends or adjust these servers.
        server k3s-w-1 192.168.10.21:80 check # Point to Worker1 internal IP, port 80
        # Add other workers here if applicable:
        # server k3s-w-2 192.168.10.22:80 check

    #---------------------------------------------------------------------
    # Frontend: HTTPS Ingress (Port 443)
    #---------------------------------------------------------------------
    frontend https_ingress_frontend
        bind 192.168.1.89:443 # Listen on LAN IP, port 443
        mode tcp
        option tcplog
        default_backend https_ingress_backend

    #---------------------------------------------------------------------
    # Backend: HTTPS Ingress (Port 443)
    #---------------------------------------------------------------------
    backend https_ingress_backend
        mode tcp
        balance source
        # Point to worker nodes where Traefik/Ingress handles TLS termination.
        server k3s-w-1 192.168.10.21:443 check # Point to Worker1 internal IP, port 443
        # Add other workers here if applicable:
        # server k3s-w-2 192.168.10.22:443 check

    #---------------------------------------------------------------------
    # Optional: HAProxy Stats Page
    #---------------------------------------------------------------------
    listen stats
        bind 192.168.1.89:9000 # Listen on LAN IP, port 9000
        mode http
        stats enable
        stats uri /stats # URL to access stats: http://192.168.1.89:9000/stats
        stats refresh 10s
        stats admin if TRUE # Allows enabling/disabling servers via UI (use basic auth below for security)
        # Optional: Add basic authentication
        # stats auth username:password # Replace with your desired credentials

    # --- End of content for /etc/haproxy/haproxy.cfg ---
    ```

    *   **Explanation:**
        *   `mode tcp`: We operate at Layer 4 (TCP) because TLS termination for applications happens inside the K3s cluster. HAProxy just forwards the encrypted traffic.
        *   `bind 192.168.1.89:PORT`: Tells HAProxy to listen only on the LAN-facing IP address for external connections.
        *   `balance source`: Distributes load based on the client's source IP address. This helps ensure a client consistently connects to the same backend server if needed, though less critical for pure TCP forwarding.
        *   `server NAME IP:PORT check`: Defines a backend server. `check` enables health checking; HAProxy will stop sending traffic to servers that don't respond.
        *   `listen stats`: (Optional) Configures a useful web page on port 9000 to monitor HAProxy's status and backend health.

3.  **Check Configuration Syntax:** Before restarting, it's wise to check the configuration file for errors:
    ```bash
    sudo haproxy -c -f /etc/haproxy/haproxy.cfg
    ```
    It should report "Configuration file is valid".

4.  **Enable and Start HAProxy:**
    ```bash
    # Restart HAProxy to apply the new configuration
    sudo systemctl restart haproxy

    # Enable HAProxy to start automatically on boot
    sudo systemctl enable haproxy

    # Check the status
    sudo systemctl status haproxy
    ```
    Ensure the service is active (running). Check logs (`journalctl -u haproxy`) if there are issues.

---

HAProxy is now configured and running. It's ready to accept connections on `192.168.1.89` for ports 6443, 80, and 443, and forward them to the internal K3s nodes once they are online. The next major phase involves preparing the K3s node VMs and installing the K3s cluster itself.
---
