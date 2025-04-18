# K3s Ubuntu Lab Setup Guide

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

**Objective:** Assign static IP addresses to both interfaces and ensure they are configured correctly for their respective networks.

**Procedure:**

1.  **Identify Interface Names:** Log in to your `k8-svc` VM via SSH and identify the names of your two network interfaces using the command:
    ```bash
    ip addr show
    ```
    Look for interfaces like `eth0`, `eth1`, `ens192`, `ens224`, etc. Note which one is currently connected to your main LAN (it likely has an IP in the `192.168.1.x` range if it got one via DHCP initially) and which one is intended for the internal K3s network. For this guide, we'll assume:
    *   `ens192`: Connects to Main LAN (`192.168.1.0/24`)
    *   `ens224`: Connects to Internal K3s Network (`192.168.10.0/24`) - *Replace `ens224` with your actual interface name.*

2.  **Configure Static IPs (Netplan):** Ubuntu 22.04 uses Netplan for network configuration. Configuration files are typically located in `/etc/netplan/`. Edit the existing YAML file (e.g., `00-installer-config.yaml`) or create a new one (e.g., `50-k8svc-config.yaml`) with the following content. **Ensure you replace interface names (`ens192`, `ens224`) if yours are different.**

    ```yaml
    # /etc/netplan/50-k8svc-config.yaml
    network:
      version: 2
      ethernets:
        ens192: # Interface connected to Main LAN
          dhcp4: no
          addresses:
            - 192.168.1.89/24 # Static IP for k8-svc on Main LAN
          routes:
            - to: default
              via: 192.168.1.1 # Your Main LAN Router/Gateway IP
          nameservers:
            addresses: [192.168.1.53] # Your Pi-hole IP
        ens224: # Interface connected to Internal K3s Network
          dhcp4: no
          addresses:
            - 192.168.10.1/24 # Static IP for k8-svc on Internal K3s Net (Gateway for K3s nodes)
          # No gateway route needed here
    ```

    *   **Explanation:**
        *   `dhcp4: no`: Disables DHCP for these interfaces as we're setting static IPs.
        *   `addresses`: Defines the static IP address and subnet mask (using CIDR notation, `/24` is equivalent to `255.255.255.0`).
        *   `routes`: For the LAN interface (`ens192`), we define the default gateway (`192.168.1.1`) for outbound internet traffic.
        *   `nameservers`: For the LAN interface (`ens192`), we explicitly set the Pi-hole (`192.168.1.53`) as the DNS server for the `k8-svc` VM itself. The internal interface (`ens224`) doesn't need DNS configured here; we'll set up a forwarder later.

3.  **Apply Netplan Configuration:**
    ```bash
    sudo netplan apply
    ```
    This command applies the changes. You might lose SSH connection briefly if you were connected via an IP that changed; reconnect using the new static IP (`192.168.1.89`).

4.  **Verify Configuration:** Check the IP addresses again:
    ```bash
    ip addr show ens192
    ip addr show ens224
    ```
    Verify the routes:
    ```bash
    ip route show
    ```
    You should see the default route via `192.168.1.1` associated with `ens192`.

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

2.  **Add the Masquerade Rule:** Use `iptables` to add the rule. **Remember to replace `ens192` if your main LAN interface name is different.**
    ```bash
    sudo iptables -t nat -A POSTROUTING -o ens192 -s 192.168.10.0/24 -j MASQUERADE
    ```
    *   **Explanation:**
        *   `-t nat`: Specifies the NAT table.
        *   `-A POSTROUTING`: Appends the rule to the POSTROUTING chain (applied just before packets leave the machine).
        *   `-o ens192`: Matches packets going *out* of the specified interface (`ens192`).
        *   `-s 192.168.10.0/24`: Matches packets with a *source* IP address from the internal K3s network.
        *   `-j MASQUERADE`: Jumps to the MASQUERADE target, which automatically replaces the source IP with the IP of the outgoing interface (`ens192`).

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

3.  **Specify Listening Interface:** We need to tell the DHCP server to only listen on the internal network interface. **Remember to replace `ens224` with your actual internal interface name.**
    ```bash
    sudo nano /etc/default/isc-dhcp-server
    ```
    Find the line starting with `INTERFACESv4=""` and modify it to specify your internal interface:
    ```ini
    INTERFACESv4="ens224"
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

The K3s nodes will be configured (via DHCP) to send their DNS queries to the `k8-svc` VM's internal IP (`192.168.10.1`). We need to configure the service listening on this IP to forward those queries to your main LAN's Pi-hole DNS server (`192.168.1.53`). Ubuntu 22.04 uses `systemd-resolved` for DNS management by default.

**Objective:** Configure `systemd-resolved` to listen on the internal interface and forward DNS queries to the upstream Pi-hole server.

**Procedure:**

1.  **Check `systemd-resolved` Status:** First, ensure the service is running.
    ```bash
    sudo systemctl status systemd-resolved
    ```
    It should be active (running).

2.  **Configure `resolved.conf`:** Edit the main configuration file for `systemd-resolved`.
    ```bash
    sudo nano /etc/systemd/resolved.conf
    ```

3.  **Modify Settings:** Within the `[Resolve]` section of the file, make the following changes:
    *   **`DNS=`:** Set your Pi-hole as the primary upstream DNS server. If the line is commented out (`#DNS=`), remove the `#`. Add or modify it to include your Pi-hole's IP. You can also add a secondary public DNS server as a fallback.
    *   **`FallbackDNS=`:** (Optional) You can specify public DNS servers here (like `1.1.1.1` or `8.8.8.8`) which `systemd-resolved` might use if the primary DNS servers become unresponsive.
    *   **`DNSStubListener=`:** By default, `systemd-resolved` uses a "stub" listener on `127.0.0.53`. For our `k8-svc` VM to listen for queries from K3s nodes on its `192.168.10.1` address, we need to disable the stub listener and let `systemd-resolved` listen directly on port 53 on relevant interfaces. Change this setting to `no`.

    Your modified `[Resolve]` section should look similar to this (ensure other settings are kept unless you intend to change them):

    ```ini
    [Resolve]
    DNS=192.168.1.53 # Primary DNS is Pi-hole
    # FallbackDNS=1.1.1.1 8.8.8.8 # Optional: Add fallback servers
    # Domains=~. # Default setting, usually fine
    # LLMNR=no # Default setting, usually fine
    # MulticastDNS=no # Default setting, usually fine
    # DNSSEC=no # Default setting, usually fine
    # DNSOverTLS=no # Default setting, usually fine
    # Cache=yes # Default setting, usually fine
    DNSStubListener=no # Important: Disable stub listener
    ```

4.  **Save and Close:** Save the file and exit the editor.

5.  **Restart `systemd-resolved`:** Apply the configuration changes by restarting the service.
    ```bash
    sudo systemctl restart systemd-resolved
    ```

6.  **Verify Listener:** Check which addresses `systemd-resolved` is now listening on.
    ```bash
    sudo ss -tulnp | grep systemd-resolve
    ```
    You should now see `systemd-resolved` listening on `192.168.10.1:53` (and likely `127.0.0.1:53`, `192.168.1.89:53`, etc.) for both TCP and UDP (port 53). If you don't see it listening on `192.168.10.1:53`, double-check the `DNSStubListener=no` setting and restart the service again. Sometimes a full system reboot might be needed for network service changes to fully take effect.

    *   **Explanation:** By disabling the stub listener, `systemd-resolved` binds directly to port 53 on the configured network interfaces (`ens192` and `ens224` in our case, based on the Netplan config). When a K3s node sends a DNS query to `192.168.10.1`, `systemd-resolved` receives it, checks its cache, and if necessary, forwards the query to the upstream server defined in `DNS=` (our Pi-hole at `192.168.1.53`).

---

With DNS forwarding configured, the `k8-svc` VM is now ready to handle DNS requests from the internal K3s network. The next step is to install and configure HAProxy to manage access to the K3s API and applications.
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

## Part 5: K3s Node Preparation and Cluster Installation

With the `k8-svc` VM providing essential network services (DHCP, DNS forwarding, Routing, Load Balancing), we can now prepare the virtual machines for the K3s nodes and install the cluster. We'll set up a High Availability (HA) control plane using K3s's embedded etcd.

**Objective:** Install Ubuntu on the K3s node VMs, ensure they get correct network configuration via DHCP, and install K3s in an HA configuration.

**Prerequisites:**

*   Three Ubuntu 22.04 VMs created (2 for Control Plane, 1 for Worker).
*   Each VM should have only *one* network interface, connected to the same network segment as the `k8-svc` VM's internal interface (`ens224` / `192.168.10.0/24`).
*   You have recorded the MAC addresses for each of these VMs and updated the `/etc/dhcp/dhcpd.conf` file on the `k8-svc` VM (as described in Part 2, Step 2) and restarted the DHCP server (`sudo systemctl restart isc-dhcp-server`).

### Step 5.1: Prepare K3s Node VMs

1.  **Install Ubuntu Server 22.04:** Install the OS on all three VMs (let's call them `k3s-cp-1`, `k3s-cp-2`, `k3s-w-1`). Standard installation is sufficient. Ensure SSH server is enabled during installation or install it afterwards (`sudo apt update && sudo apt install openssh-server -y`).
2.  **Configure Network (DHCP):** During installation or post-installation using Netplan (`/etc/netplan/`), ensure the single network interface on each VM is configured to use DHCP. This is often the default. Example `/etc/netplan/00-installer-config.yaml`:
    ```yaml
    network:
      ethernets:
        ens160: # Replace with your actual interface name
          dhcp4: true
      version: 2
    ```
    Apply with `sudo netplan apply`.
3.  **Verify Network Configuration:** Boot up the VMs. Once booted, log in via SSH (you might need to check your hypervisor or DHCP server logs on `k8-svc` - `sudo tail -f /var/log/syslog | grep DHCPACK` - to find their assigned IPs initially). Verify each node received the correct IP address, gateway, and DNS server from the `k8-svc` DHCP server:
    ```bash
    # Check IP address
    ip addr show

    # Check default route (should be via 192.168.10.1)
    ip route show

    # Check DNS server (should show 192.168.10.1)
    cat /etc/resolv.conf
    # Or use: systemd-resolve --status | grep 'DNS Servers'
    ```
    Also, test connectivity:
    ```bash
    # Ping gateway (k8-svc internal IP)
    ping -c 3 192.168.10.1

    # Ping Pi-hole (should work via k8-svc routing/DNS)
    ping -c 3 192.168.1.53

    # Ping external address (should work via k8-svc NAT)
    ping -c 3 google.com

    # Resolve external address (should work via k8-svc -> Pi-hole)
    dig google.com @192.168.10.1 # Query using k8-svc DNS
    ```
    Troubleshoot any connectivity issues before proceeding. Ensure `iptables` rules and IP forwarding are correct on `k8-svc`.

### Step 5.2: Install K3s HA Cluster (Embedded etcd)

We will use the K3s installation script. The process involves initializing the first control plane node, then joining the second control plane node and the worker node(s).

**Reference:** Always consult the official [K3s High Availability with Embedded DB documentation](https://docs.k3s.io/installation/ha-embedded) for the most up-to-date commands and options.

1.  **Install First Control Plane Node (`k3s-cp-1`):**
    *   SSH into `k3s-cp-1` (`192.168.10.11`).
    *   Run the K3s installation script with the `cluster-init` flag to initialize the embedded etcd cluster. We also specify the `--tls-san` argument to add the HAProxy IP (`192.168.1.89`) to the K3s API server's certificate Subject Alternative Names (SANs). This is crucial for `kubectl` access via HAProxy without certificate errors.
    ```bash
    curl -sfL https://get.k3s.io | K3S_TOKEN=YOUR_SECURE_CLUSTER_SECRET sh -s - server \
        --cluster-init \
        --tls-san 192.168.1.89
    ```
    *   **Explanation:**
        *   `curl ... | sh -s -`: Downloads and executes the K3s installation script.
        *   `K3S_TOKEN=YOUR_SECURE_CLUSTER_SECRET`: Sets a shared secret used to join other nodes to the cluster. **Replace `YOUR_SECURE_CLUSTER_SECRET` with a strong, unique password or token.** Keep this token secure!
        *   `server`: Specifies that this node should run the K3s server components (control plane, etcd).
        *   `--cluster-init`: Tells this server node to initialize a new embedded etcd database cluster. Only use this on the *first* control plane node.
        *   `--tls-san 192.168.1.89`: Adds the specified IP address (our HAProxy LAN IP) as a valid name in the K3s API server's TLS certificate.

2.  **Verify First Node:** After installation completes (it might take a minute or two), check that the K3s service is running:
    ```bash
    sudo systemctl status k3s
    ```
    Check that the node is ready (it might take a bit longer for all pods to start):
    ```bash
    sudo k3s kubectl get nodes
    ```
    You should see `k3s-cp-1` listed with status `Ready`.

3.  **Install Second Control Plane Node (`k3s-cp-2`):**
    *   SSH into `k3s-cp-2` (`192.168.10.12`).
    *   Run the K3s installation script, specifying the `server` flag, the `--server` URL pointing to the *first* control plane node, the shared token, and the `--tls-san` for HAProxy.
    ```bash
    curl -sfL https://get.k3s.io | K3S_TOKEN=YOUR_SECURE_CLUSTER_SECRET sh -s - server \
        --server https://192.168.10.11:6443 \
        --tls-san 192.168.1.89
    ```
    *   **Explanation:**
        *   `K3S_TOKEN=...`: Use the *exact same* token defined in Step 1.
        *   `server`: Specifies this node should also run K3s server components.
        *   `--server https://192.168.10.11:6443`: Tells this node how to connect to the *existing* cluster (by contacting the first node's API). K3s will handle joining the embedded etcd cluster automatically.
        *   `--tls-san 192.168.1.89`: Ensure the HAProxy IP is included in this node's certificate generation as well.

4.  **Verify Second Node:** Check the service status on `k3s-cp-2`:
    ```bash
    sudo systemctl status k3s
    ```
    On *either* control plane node (`k3s-cp-1` or `k3s-cp-2`), check the node status again. You should now see both `k3s-cp-1` and `k3s-cp-2` listed as `Ready`. It might take a few moments for the second node to fully join and become ready.
    ```bash
    sudo k3s kubectl get nodes
    ```

5.  **Install Worker Node (`k3s-w-1`):**
    *   SSH into `k3s-w-1` (`192.168.10.21`).
    *   Run the K3s installation script using the `agent` role, pointing it to the cluster via the *first control plane node's* URL (or the second, it doesn't matter now that the cluster is HA) and providing the token.
    ```bash
    curl -sfL https://get.k3s.io | K3S_TOKEN=YOUR_SECURE_CLUSTER_SECRET sh -s - agent \
        --server https://192.168.10.11:6443
    ```
    *   **Explanation:**
        *   `agent`: Specifies this node should run only the K3s agent components (kubelet, kube-proxy, flannel CNI).
        *   `--server https://...`: Tells the agent how to connect to the control plane API.
        *   `K3S_TOKEN=...`: Use the same shared token.

6.  **Verify Worker Node:** Check the agent service status on `k3s-w-1`:
    ```bash
    sudo systemctl status k3s-agent
    ```
    On either control plane node, check the node list again. You should now see all three nodes (`k3s-cp-1`, `k3s-cp-2`, `k3s-w-1`) listed as `Ready`.
    ```bash
    sudo k3s kubectl get nodes
    ```

### Step 5.3: Configure `kubectl` Access via HAProxy

To manage the cluster from your client machine (or the `k8-svc` VM) using `kubectl`, you need the cluster's kubeconfig file and must modify it to point to the HAProxy load balancer IP.

1.  **Get Kubeconfig:** On one of the control plane nodes (e.g., `k3s-cp-1`), the kubeconfig file is located at `/etc/rancher/k3s/k3s.yaml`. Display its content:
    ```bash
    sudo cat /etc/rancher/k3s/k3s.yaml
    ```
2.  **Copy Kubeconfig:** Copy the entire output from the command above.
3.  **Save Locally:** Paste the copied content into a file on your client machine where you run `kubectl`. A common location is `~/.kube/config`. If that file already exists, you should carefully merge the new cluster, context, and user information into it, or save it as a separate file (e.g., `~/.kube/k3s-lab-config`).
4.  **Modify Server Address:** Edit the kubeconfig file you just saved. Find the line starting with `server:` under the `cluster` definition. It will currently point to `https://127.0.0.1:6443` or `https://<node-ip>:6443`. Change this IP address to point to your HAProxy LAN IP:
    ```yaml
    # Example snippet from kubeconfig file
    clusters:
    - cluster:
        certificate-authority-data: <REDACTED>
        # Modify the server line below:
        server: https://192.168.1.89:6443 # <-- Change this IP
      name: default
    ```
5.  **Test `kubectl` Access:** If you saved the config as `~/.kube/config`, you can test directly. If you saved it as a different file, use the `--kubeconfig` flag:
    ```bash
    # If saved as ~/.kube/config
    kubectl get nodes

    # If saved as ~/.kube/k3s-lab-config
    kubectl --kubeconfig=~/.kube/k3s-lab-config get nodes
    ```
    You should see the list of your three K3s nodes, confirming that access via HAProxy is working correctly. The `--tls-san` flag used during installation ensures `kubectl` trusts the certificate presented by HAProxy (which originates from the K3s API server).

---

Congratulations! You now have a functioning HA K3s cluster installed and accessible via HAProxy. The next steps involve deploying MetalLB for LoadBalancer services and configuring DNS and certificates for applications.
---

## Part 6: Install and Configure MetalLB

To expose services within your K3s cluster using stable IP addresses on your internal network (`192.168.10.0/24`), we need a network load balancer. MetalLB fulfills this role for bare-metal (or VM-based) clusters by assigning IPs from a predefined pool to services of `Type=LoadBalancer`.

**Objective:** Deploy MetalLB to the cluster and configure it to manage a pool of IP addresses from the `192.168.10.0/24` network.

**Prerequisites:**

*   A working K3s cluster (Part 5 completed).
*   `kubectl` configured to access the cluster via HAProxy (`https://192.168.1.89:6443`).

**Procedure:**

1.  **Apply MetalLB Manifests:** MetalLB installation involves applying a set of Kubernetes manifests directly from their official source. It's recommended to check the [MetalLB Installation Guide](https://metallb.universe.tf/installation/) for the latest stable version and manifest URL. We'll use the native Kubernetes API installation method.

    *   **Note:** K3s bundles its own service load balancer (ServiceLB/Klipper), which might conflict if not disabled. However, MetalLB typically works alongside it or can replace it. For simplicity, we'll proceed with the standard MetalLB installation. If you encounter issues with LoadBalancer services later, you might need to disable the K3s ServiceLB by adding `--disable servicelb` to the K3s server startup commands (requires restarting the `k3s` service on control plane nodes). For now, let's assume the default works.

    Run the following `kubectl apply` command from your client machine (replace `vX.Y.Z` with the desired MetalLB version, e.g., `v0.13.12` - check their releases page):

    ```bash
    # Example using v0.13.12 - CHECK FOR LATEST VERSION
    METALLB_VERSION=v0.13.12
    kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/${METALLB_VERSION}/config/manifests/metallb-native.yaml
    ```
    This command creates the necessary namespaces, custom resource definitions (CRDs), controller deployments, and speaker daemonsets for MetalLB.

2.  **Wait for MetalLB Pods:** Allow a minute or two for the MetalLB pods to download and start. You can monitor their status:
    ```bash
    kubectl get pods -n metallb-system -w
    ```
    Wait until the `controller` pod and the `speaker` pods (one per node) are in the `Running` state. Press Ctrl+C to exit the watch.

3.  **Configure IP Address Pool:** We need to tell MetalLB which IP addresses it's allowed to manage and assign to LoadBalancer services. Create a file named `metallb-ip-pool.yaml` on your client machine with the following content:

    ```yaml
    # metallb-ip-pool.yaml
    apiVersion: metallb.io/v1beta1
    kind: IPAddressPool
    metadata:
      name: internal-pool
      namespace: metallb-system
    spec:
      addresses:
      - 192.168.10.100-192.168.10.150 # Define your desired range within the K3s network
    ```

    *   **Explanation:**
        *   `kind: IPAddressPool`: Defines a pool of IPs.
        *   `metadata.name`: A name for this pool (e.g., `internal-pool`).
        *   `metadata.namespace`: Must be created in the `metallb-system` namespace.
        *   `spec.addresses`: A list of IP ranges MetalLB can use. We've defined `192.168.10.100` to `192.168.10.150`. Adjust this range as needed, ensuring it doesn't overlap with your static DHCP assignments or the gateway IP.

4.  **Apply IP Address Pool Configuration:**
    ```bash
    kubectl apply -f metallb-ip-pool.yaml
    ```

5.  **Configure L2 Advertisement:** Now, we need to tell MetalLB *how* to make the assigned IPs reachable on the network. For simple networks without BGP, Layer 2 mode (using ARP/NDP) is appropriate. Create a file named `metallb-l2-advertisement.yaml` on your client machine:

    ```yaml
    # metallb-l2-advertisement.yaml
    apiVersion: metallb.io/v1beta1
    kind: L2Advertisement
    metadata:
      name: internal-advertisement
      namespace: metallb-system
    spec:
      # Optional: Specify which IPAddressPool this advertisement applies to.
      # If omitted, it applies to all pools.
      ipAddressPools:
      - internal-pool
      # Optional: Specify which nodes can advertise IPs.
      # If omitted, all nodes can advertise. You might restrict this to workers.
      # nodeSelectors:
      # - matchLabels:
      #     kubernetes.io/os: linux # Example: Advertise from all Linux nodes
    ```

    *   **Explanation:**
        *   `kind: L2Advertisement`: Configures Layer 2 announcements.
        *   `metadata.name`: A name for this advertisement configuration.
        *   `metadata.namespace`: Must be in `metallb-system`.
        *   `spec.ipAddressPools`: Links this advertisement method to the specific IP pool we created (`internal-pool`). When a service gets an IP from `internal-pool`, MetalLB speakers (running on your nodes) will respond to ARP requests for that IP, effectively announcing "that IP lives here".

6.  **Apply L2 Advertisement Configuration:**
    ```bash
    kubectl apply -f metallb-l2-advertisement.yaml
    ```

7.  **Verify MetalLB Configuration (Optional):** You can check the status of the resources:
    ```bash
    kubectl get ipaddresspools -n metallb-system
    kubectl get l2advertisements -n metallb-system
    ```

---

MetalLB is now installed and configured. When you create a Kubernetes Service of `Type=LoadBalancer`, MetalLB will automatically assign it an available IP from the `192.168.10.100-192.168.10.150` range, and the MetalLB speakers will make that IP reachable on your `192.168.10.0/24` network segment. The next part will cover deploying a sample application and configuring DNS/Certificates.
---

## Part 7: Deploy Application, Configure DNS & Certificates

With the cluster infrastructure (K3s, HAProxy, MetalLB) in place, let's deploy a sample application (Grafana) and make it securely accessible from your main LAN using its hostname (`grafana.komebacklabs.lan`).

**Objective:** Deploy Grafana, expose it via a MetalLB LoadBalancer IP, configure Pi-hole DNS, create a TLS secret from custom certificates, and enable TLS termination within the cluster.

**Prerequisites:**

*   Working K3s cluster with MetalLB configured (Part 6 completed).
*   `kubectl` access configured.
*   Access to your Pi-hole admin interface.
*   Your custom certificate files for `grafana.komebacklabs.lan` (e.g., `grafana.crt`, `grafana.key`) and the Root CA certificate (`ca.crt`) available on your client machine.

### Step 7.1: Deploy Grafana (Example Application)

We'll deploy Grafana using a simple Deployment and Service manifest. For production, using the official Helm chart is recommended, but this illustrates the core concepts.

1.  **Create Grafana Manifest (`grafana-deployment.yaml`):** Create this file on your client machine.

    ```yaml
    # grafana-deployment.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: grafana
      namespace: default # Or create a dedicated namespace
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: grafana
      template:
        metadata:
          labels:
            app: grafana
        spec:
          containers:
          - name: grafana
            image: grafana/grafana:latest # Use a specific version in production
            ports:
            - containerPort: 3000
              name: http
            # Add volume mounts here if you need persistent storage for Grafana data
            # volumeMounts:
            # - name: grafana-storage
            #   mountPath: /var/lib/grafana
          # volumes:
          # - name: grafana-storage
          #   persistentVolumeClaim:
          #     claimName: grafana-pvc # You would need to create a PVC
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: grafana-svc
      namespace: default # Use the same namespace as the deployment
    spec:
      selector:
        app: grafana
      ports:
        - name: http
          protocol: TCP
          port: 443 # Expose HTTPS port externally
          targetPort: 3000 # Target Grafana's internal HTTP port
      type: LoadBalancer # Request an IP from MetalLB
      # Optional: Specify a specific IP from the pool if desired
      # loadBalancerIP: 192.168.10.100
    ```

    *   **Explanation:**
        *   `Deployment`: Defines how to run Grafana pods using the official image. Port 3000 is the default Grafana port. *Note: Persistence is not configured here for simplicity.*
        *   `Service`: Exposes the Grafana deployment.
            *   `selector`: Links the Service to the Deployment pods (`app: grafana`).
            *   `type: LoadBalancer`: Tells Kubernetes to request an external IP from MetalLB.
            *   `ports`: We map external port `443` (standard HTTPS) to the Grafana container's port `3000`. Even though Grafana runs on HTTP internally (port 3000), we expose the service on 443 because we intend to handle TLS termination at this service level (or via an Ingress later).

2.  **Apply the Manifest:**
    ```bash
    kubectl apply -f grafana-deployment.yaml
    ```

3.  **Verify Service and Get External IP:** Check the service status and wait for MetalLB to assign an IP.
    ```bash
    kubectl get svc grafana-svc -w
    ```
    Wait until the `EXTERNAL-IP` column shows an IP address from your MetalLB pool (e.g., `192.168.10.100`). Note this IP address. Press Ctrl+C to exit.

### Step 7.2: Configure Pi-hole DNS

Now, point the desired hostname (`grafana.komebacklabs.lan`) to the HAProxy IP address (`192.168.1.89`) so that requests from your LAN clients are directed correctly.

1.  **Log in to Pi-hole:** Access your Pi-hole web admin interface.
2.  **Navigate to Local DNS Records:** Find the section for managing local DNS A or CNAME records.
3.  **Add A-Record:** Create a new **A-Record**:
    *   **Domain:** `grafana.komebacklabs.lan`
    *   **IP Address:** `192.168.1.89` (The **HAProxy** LAN IP)
4.  **Save:** Add the record.

    *   **Explanation:** When a client on your LAN queries Pi-hole for `grafana.komebacklabs.lan`, Pi-hole will now respond with `192.168.1.89`. The client will then send its HTTPS request to HAProxy. HAProxy's `https_ingress_frontend` (listening on `192.168.1.89:443`) will forward the TCP connection to the backend (initially the worker node `192.168.10.21:443`, but ultimately reaching the Grafana service's external IP `192.168.10.100` via K3s internal routing/kube-proxy). *Self-correction: HAProxy backend should ideally point directly to the MetalLB IP and port if known and stable, or rely on worker node routing.* For simplicity, pointing to workers is common initially.

### Step 7.3: Create Kubernetes TLS Secret

Store your custom certificate and key in a Kubernetes secret so the cluster can use it for TLS termination.

1.  **Prepare Certificate Files:** Ensure you have the following files on your client machine where you run `kubectl`:
    *   `grafana.crt`: The server certificate for `grafana.komebacklabs.lan`.
    *   `grafana.key`: The private key corresponding to the server certificate.
    *   *(Optional but recommended)* Ensure the `.crt` file contains the full certificate chain (server cert + intermediate CA cert(s)), excluding the Root CA.

2.  **Create the Secret:** Use `kubectl create secret tls`.
    ```bash
    kubectl create secret tls grafana-tls \
      --cert=path/to/your/grafana.crt \
      --key=path/to/your/grafana.key \
      -n default # Use the same namespace as Grafana
    ```
    Replace `path/to/your/` with the actual paths to your certificate files.

3.  **Verify Secret Creation (Optional):**
    ```bash
    kubectl get secret grafana-tls -n default
    ```

### Step 7.4: Configure TLS Termination (Example: Using Ingress)

While you could configure Grafana itself for TLS, the standard Kubernetes way is to use an Ingress controller. K3s includes Traefik by default. Let's create an Ingress resource to manage TLS for Grafana.

1.  **Create Ingress Manifest (`grafana-ingress.yaml`):** Create this file on your client machine.

    ```yaml
    # grafana-ingress.yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: grafana-ingress
      namespace: default # Use the same namespace as Grafana
      # Optional annotations for specific ingress controllers (e.g., Traefik)
      # annotations:
      #   kubernetes.io/ingress.class: "traefik"
    spec:
      tls:
      - hosts:
          - grafana.komebacklabs.lan
        secretName: grafana-tls # Reference the secret created earlier
      rules:
      - host: grafana.komebacklabs.lan
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: grafana-svc # Target the Grafana Service
                port:
                  name: http # Target the service port named 'http' (which maps to 443 externally, 3000 internally)
    ```

    *   **Explanation:**
        *   `kind: Ingress`: Defines rules for routing external HTTP/HTTPS traffic to internal services.
        *   `spec.tls`: Configures TLS termination.
            *   `hosts`: Specifies the domain name(s) this TLS configuration applies to.
            *   `secretName: grafana-tls`: Tells the Ingress controller (Traefik) to use the certificate and key from the `grafana-tls` secret for the specified hosts.
        *   `spec.rules`: Defines routing rules.
            *   `host`: Matches requests for `grafana.komebacklabs.lan`.
            *   `http.paths`: Defines how to route paths under that host.
            *   `backend.service`: Specifies the target service (`grafana-svc`) and port (`http`). Traefik will handle the connection to the service.

2.  **Apply the Ingress Manifest:**
    ```bash
    kubectl apply -f grafana-ingress.yaml
    ```

    *   **How it works now:** When HAProxy forwards the TCP connection for port 443 to the cluster (likely hitting Traefik's LoadBalancer service eventually), Traefik sees the incoming TLS connection for `grafana.komebacklabs.lan`. It uses the `grafana-tls` secret to terminate the TLS connection, then forwards the decrypted HTTP request to the `grafana-svc` on its internal port (3000).

### Step 7.5: Ensure Client Trusts Root CA

For your browser to trust the custom certificate presented by the cluster, the client machine (where you access Grafana from) must trust your custom Root CA certificate (`ca.crt`).

1.  **Import Root CA:** Import your `ca.crt` file into your operating system's trust store and/or your browser's certificate manager. The exact steps vary depending on your OS (Windows, macOS, Linux) and browser. Search for "import root ca certificate [your OS/browser]".

### Step 7.6: Test Access

1.  **Clear DNS Cache (Optional):** On your client machine, you might need to clear the local DNS cache to ensure it picks up the new record from Pi-hole.
    *   Windows: `ipconfig /flushdns`
    *   macOS: `sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder`
    *   Linux (systemd): `sudo systemd-resolve --flush-caches`
2.  **Access Grafana:** Open your web browser and navigate to:
    `https://grafana.komebacklabs.lan`
3.  **Verify:**
    *   You should see the Grafana login page.
    *   The browser should show a secure connection (e.g., a padlock icon).
    *   Check the certificate details in the browser - it should show the details from your custom `grafana.komebacklabs.lan` certificate and indicate it's trusted because the Root CA is installed.

---

This completes the basic setup guide! You have deployed K3s, configured networking services on `k8-svc`, set up MetalLB, and exposed a sample application securely using HAProxy, Pi-hole, and custom certificates with TLS termination inside the cluster via Ingress. You can follow similar principles to deploy other applications. Remember to consult the `k3s-lab-overview.md` for a summary of the architecture and IP addresses.
---

## Part 8: Configure Persistent Storage (NFS)

For stateful applications (like databases, GitLab, Harbor data, etc.) to run effectively in Kubernetes, they need persistent storage that survives pod restarts. We will configure the `k8-svc` VM as an NFS server and deploy a dynamic provisioner in K3s that automatically creates PersistentVolumes (PVs) on this NFS share when applications request storage via PersistentVolumeClaims (PVCs).

**Objective:** Enable dynamic persistent storage provisioning for the K3s cluster using an NFS share hosted on `k8-svc`.

**Prerequisites:**

*   Working K3s cluster accessible via `kubectl` (Part 7 completed).
*   SSH access to `k8-svc` and all K3s nodes.

### Step 8.1: Configure NFS Server on `k8-svc`

1.  **Install NFS Server Package:**
    ```bash
    # On k8-svc VM
    sudo apt update
    sudo apt install nfs-kernel-server -y
    ```

2.  **Create Export Directory:** This is the root directory on `k8-svc` where all the persistent volume data will be stored in subdirectories.
    ```bash
    # On k8-svc VM
    sudo mkdir -p /srv/nfs/kubedata
    ```

3.  **Set Permissions:** The NFS Subdir External Provisioner typically runs as `nobody:nogroup`. We need to grant ownership and permissions accordingly.
    ```bash
    # On k8-svc VM
    sudo chown nobody:nogroup /srv/nfs/kubedata
    sudo chmod 777 /srv/nfs/kubedata
    ```
    *   **Explanation:** `chown nobody:nogroup` assigns ownership to a low-privilege user/group. `chmod 777` grants read, write, and execute permissions to everyone (owner, group, others). While convenient for the provisioner (which creates subdirectories within this export), `777` is very permissive. In a production environment, you'd explore more restrictive permissions combined with specific NFS export options if possible, but `777` is common for this specific provisioner setup.

4.  **Configure NFS Exports:** Edit the `/etc/exports` file to define which directories are shared and with whom.
    ```bash
    # On k8-svc VM
    sudo nano /etc/exports
    ```
    Add the following line to the file:
    ```plaintext
    /srv/nfs/kubedata    192.168.10.0/24(rw,sync,no_subtree_check,no_root_squash)
    ```
    *   **Explanation:**
        *   `/srv/nfs/kubedata`: The directory we created.
        *   `192.168.10.0/24`: Allows access only from clients within the internal K3s network subnet.
        *   `rw`: Allows clients read and write access.
        *   `sync`: Replies to requests only after changes have been committed to stable storage (safer, slightly slower).
        *   `no_subtree_check`: Disables subtree checking, which can cause issues when a subdirectory is exported but the parent isn't, and generally improves reliability.
        *   `no_root_squash`: This is important. It prevents the NFS server from mapping requests from the client's `root` user to the `nobody` user. The provisioner needs this to manage permissions within the subdirectories it creates.

5.  **Apply Export Configuration:** Make the NFS server aware of the changes.
    ```bash
    # On k8-svc VM
    sudo exportfs -a
    ```

6.  **Enable and Start NFS Service:**
    ```bash
    # On k8-svc VM
    sudo systemctl enable --now nfs-kernel-server
    sudo systemctl status nfs-kernel-server
    ```
    Ensure the service is active (running).

7.  **Firewall Considerations (If Applicable):** Although you mentioned your firewall is open, in a firewalled environment, you would need to allow NFS traffic from the `192.168.10.0/24` network to the `k8-svc` VM (`192.168.10.1`). This typically involves allowing traffic on TCP/UDP port 2049 (NFS) and potentially ports for `rpcbind` (111) and `mountd` if they are not handled automatically by the `nfs-kernel-server` service rules.

### Step 8.2: Install NFS Client on K3s Nodes

The K3s nodes need the NFS client utilities to be able to mount the NFS share.

1.  **Install `nfs-common`:** Run this on **all** K3s nodes (`k3s-cp-1`, `k3s-cp-2`, `k3s-w-1`).
    ```bash
    # On EACH K3s node
    sudo apt update
    sudo apt install nfs-common -y
    ```
    *(Note: This might already be installed as part of Ubuntu Server, but running the command ensures it's present.)*

### Step 8.3: Deploy NFS Subdir External Provisioner

This component runs inside Kubernetes, watches for PVCs, and automatically creates corresponding directories on the NFS share. Using Helm is the recommended way to install it.

1.  **Install Helm (If not already installed):** Follow the official Helm installation guide on your client machine where you run `kubectl`. [https://helm.sh/docs/intro/install/](https://helm.sh/docs/intro/install/)

2.  **Add Helm Repository:** Add the repository containing the NFS provisioner chart.
    ```bash
    # On your client machine
    helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
    helm repo update
    ```

3.  **Install the Chart:** Install the provisioner using `helm install`. We need to provide values specific to our setup, particularly the NFS server IP and the exported path.

    ```bash
    # On your client machine
    helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
        --set nfs.server=192.168.10.1 \
        --set nfs.path=/srv/nfs/kubedata \
        --set storageClass.name=nfs-client \
        --set storageClass.defaultClass=true \
        --set storageClass.allowVolumeExpansion=true \
        --set storageClass.reclaimPolicy=Retain \
        --create-namespace \
        --namespace nfs-provisioner
    ```
    *   **Explanation:**
        *   `helm install nfs-subdir-external-provisioner ...`: Installs the chart with the release name `nfs-subdir-external-provisioner`.
        *   `--set nfs.server=192.168.10.1`: Tells the provisioner the IP of our NFS server (`k8-svc`).
        *   `--set nfs.path=/srv/nfs/kubedata`: Tells the provisioner the path exported by the NFS server.
        *   `--set storageClass.name=nfs-client`: Names the StorageClass that will be created. Applications will request storage using this class name in their PVCs.
        *   `--set storageClass.defaultClass=true`: Makes this the default StorageClass. If a PVC doesn't specify a `storageClassName`, it will automatically use this one.
        *   `--set storageClass.allowVolumeExpansion=true`: Allows PVCs using this StorageClass to be resized later if needed.
        *   `--set storageClass.reclaimPolicy=Retain`: When a PVC is deleted, the corresponding PV and the data on the NFS share will be kept (Retained). Change to `Delete` if you want the data automatically removed when the PVC is deleted. `Retain` is safer for important data.
        *   `--create-namespace --namespace nfs-provisioner`: Creates a dedicated namespace `nfs-provisioner` and installs the chart there.

4.  **Verify Provisioner Pod:** Check that the provisioner pod starts successfully in its namespace.
    ```bash
    kubectl get pods -n nfs-provisioner -w
    ```
    Wait for the pod (e.g., `nfs-subdir-external-provisioner-...`) to be `Running`. Press Ctrl+C to exit.

5.  **Verify StorageClass:** Check that the StorageClass was created and marked as default.
    ```bash
    kubectl get sc # sc is short for storageclass
    ```
    You should see `nfs-client` listed, potentially marked as `(default)`.

### Step 8.4: Verify Dynamic Provisioning

Let's test if the setup works by creating a sample PVC.

1.  **Create Sample PVC Manifest (`test-pvc.yaml`):** Create this file on your client machine.
    ```yaml
    # test-pvc.yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: test-nfs-pvc
    spec:
      # storageClassName: nfs-client # Not needed if it's the default SC
      accessModes:
        - ReadWriteOnce # Can be mounted by a single node R/W (common)
                       # Use ReadWriteMany for shared access if needed/supported
      resources:
        requests:
          storage: 1Gi # Request 1 Gibibyte of storage
    ```

2.  **Apply the PVC:**
    ```bash
    kubectl apply -f test-pvc.yaml
    ```

3.  **Check PVC Status:**
    ```bash
    kubectl get pvc test-nfs-pvc
    ```
    Initially, the status might be `Pending`. After a short while (once the provisioner acts), it should change to `Bound`.

4.  **Check PV Status:** A corresponding PersistentVolume should have been created automatically.
    ```bash
    kubectl get pv
    ```
    You should see a PV listed (with a name like `pvc-xxxxx...`) that corresponds to your `test-nfs-pvc`, showing status `Bound` and using the `nfs-client` StorageClass.

5.  **Check NFS Server (Optional):** On the `k8-svc` VM, check the contents of the NFS export directory. You should see a new subdirectory created by the provisioner for this PVC.
    ```bash
    # On k8-svc VM
    ls -l /srv/nfs/kubedata/
    ```
    You'll likely see a directory named something like `default-test-nfs-pvc-pvc-xxxxx...`.

---

Persistent storage using NFS is now configured and operational! Your applications can now request storage using PVCs, and the NFS provisioner will automatically handle creating the necessary volumes on the `k8-svc` NFS share. This significantly improves the capability of your lab for running stateful workloads.