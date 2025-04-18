# K3s Deployment Plan with HAProxy and External DNS

This document outlines the plan for deploying a K3s cluster using an internal network, managed by a dual-homed HAProxy VM, and integrated with an existing external Pi-hole DNS server.

## 1. Proposed Architecture Diagram

```mermaid
graph TD
    subgraph Main LAN (192.168.1.0/24)
        UserClient[Client Machine]
        PiHole[Pi-hole DNS (192.168.1.53)]
        Router[LAN Router/GW]
    end

    subgraph k8-svc VM (Ubuntu 22.04)
        direction LR
        subgraph Network Interfaces
            NIC_LAN[NIC 1: ens192 - 192.168.1.89]
            NIC_K8S[NIC 2: ensXXX - 192.168.10.1]
        end
        subgraph Services
            DHCP[DHCP Server (isc-dhcp-server)]
            DNSFwd[DNS Forwarder (systemd-resolved/dnsmasq)]
            HAP[HAProxy]
            RoutingNAT{Routing & NAT}
        end
        NIC_LAN -- Connected --> MainLANNet
        NIC_K8S -- Connected --> K8SNet
        RoutingNAT -- Routes Traffic --> NIC_LAN & NIC_K8S
        DHCP -- Serves IPs --> K8SNet
        DNSFwd -- Listens/Forwards --> NIC_K8S
        HAP -- Listens/Forwards --> NIC_LAN
    end

    subgraph K8s Internal Network (192.168.10.0/24)
        K8SNet{Internal Network}
        subgraph K3s Cluster (Ubuntu 22.04)
            CP1[K3s CP 1 (DHCP IP)]
            CP2[K3s CP 2 (DHCP IP)]
            W1[K3s Worker 1 (DHCP IP)]
            MetalLB[MetalLB IP Pool (192.168.10.x Range)]
            Ingress(Future Ingress Controller)
            GrafanaSvc[Grafana Service (Type=LB - e.g., 192.168.10.100)]

            CP1 & CP2 & W1 -- Network Config --> DHCP
            CP1 & CP2 & W1 -- DNS Queries --> DNSFwd
            GrafanaSvc -- Gets IP From --> MetalLB
            Ingress -- Gets IP From --> MetalLB
        end
        K8SNet -- Connects --> CP1 & CP2 & W1
    end

    %% Connections & Flows
    UserClient -- DNS Query (grafana.komebacklabs.lan) --> PiHole
    PiHole -- A-Record (grafana -> 192.168.1.89) --> UserClient
    UserClient -- HTTPS Request (grafana.komebacklabs.lan) --> HAP
    HAP -- TCP Forward (Port 443) --> GrafanaSvc
    GrafanaSvc -- TLS Termination & Response --> HAP
    HAP -- Encrypted Response --> UserClient

    UserClient -- kubectl --> PiHole
    PiHole -- A-Record (k8s-api.komebacklabs.lan -> 192.168.1.89) --> UserClient
    UserClient -- API Request (Port 6443) --> HAP
    HAP -- TCP Forward (Port 6443) --> CP1 & CP2

    CP1 & CP2 & W1 -- Internet Traffic --> RoutingNAT -- NAT --> Router
    CP1 & CP2 & W1 -- DNS (komebacklabs.lan) --> DNSFwd -- Forward --> PiHole
    CP1 & CP2 & W1 -- DNS (other) --> DNSFwd -- Forward --> PiHole -- Forward --> Internet DNS

    MainLANNet --- Router
```

## 2. Detailed Plan Steps

### A. `k8-svc` VM Preparation (Ubuntu 22.04)

1.  **Network Configuration:**
    *   Configure the two network interfaces:
        *   `ens192` (or similar): Static IP `192.168.1.89`, Netmask `255.255.255.0`, Gateway `192.168.1.1` (your main LAN router), DNS `192.168.1.53` (Pi-hole).
        *   `ensXXX` (second interface): Static IP `192.168.10.1`, Netmask `255.255.255.0`. No gateway needed on this interface.
2.  **Enable IP Forwarding:**
    *   Edit `/etc/sysctl.conf` and uncomment `net.ipv4.ip_forward=1`.
    *   Apply the change: `sudo sysctl -p`.
3.  **Configure NAT/Masquerading:**
    *   Use `iptables` (or `nftables`) to set up masquerading for traffic originating from the `192.168.10.0/24` network going out through the `ens192` interface.
    *   Example (`iptables`): `sudo iptables -t nat -A POSTROUTING -o ens192 -s 192.168.10.0/24 -j MASQUERADE`
    *   Ensure the rule persists across reboots (e.g., using `iptables-persistent`).
4.  **Install & Configure DHCP Server (`isc-dhcp-server`):**
    *   Install the package: `sudo apt update && sudo apt install isc-dhcp-server -y`.
    *   Configure `/etc/dhcp/dhcpd.conf`:
        *   Define the `192.168.10.0/24` subnet.
        *   Set `option routers 192.168.10.1;`
        *   Set `option domain-name-servers 192.168.10.1;`
        *   Define a dynamic range (optional).
        *   Add `host` entries for K3s nodes using their MAC addresses and desired static IPs (e.g., `192.168.10.11`, `192.168.10.12` for CPs, `192.168.10.21` for worker).
    *   Specify the interface for DHCP in `/etc/default/isc-dhcp-server` (e.g., `INTERFACESv4="ensXXX"`).
    *   Enable and start the service: `sudo systemctl enable --now isc-dhcp-server`.
5.  **Configure DNS Forwarder (`systemd-resolved`):**
    *   Ensure `systemd-resolved` is running and listening on `192.168.10.1`. (May require adjusting `/etc/systemd/resolved.conf`).
    *   Configure forwarding rules to use Pi-hole (`192.168.1.53`) as the upstream DNS server.
6.  **Install & Configure HAProxy:**
    *   Install: `sudo apt install haproxy -y`.
    *   Configure `/etc/haproxy/haproxy.cfg`:
        *   `global` and `defaults` sections.
        *   `frontend k8s_api`: `bind 192.168.1.89:6443`, `mode tcp`, `default_backend k8s_api_backend`.
        *   `backend k8s_api_backend`: `mode tcp`, `balance source`, `server cp1 192.168.10.11:6443 check`, `server cp2 192.168.10.12:6443 check`.
        *   `frontend https_ingress`: `bind 192.168.1.89:443`, `mode tcp`, `default_backend https_ingress_backend`.
        *   `backend https_ingress_backend`: `mode tcp`, `balance source`, point to MetalLB/Worker IPs/Ports where services/ingress run.
        *   `frontend http_ingress`: `bind 192.168.1.89:80`, `mode tcp`, `default_backend http_ingress_backend`.
        *   `backend http_ingress_backend`: `mode tcp`, `balance source`, point to MetalLB/Worker IPs/Ports where services/ingress run.
    *   Enable and start service: `sudo systemctl enable --now haproxy`.

### B. K3s Node Preparation (Ubuntu 22.04)

1.  **OS Installation:** Install Ubuntu 22.04 on 3 VMs (2 CP, 1 Worker).
2.  **Network Configuration:** Configure VMs to use DHCP on their single network interface (connected to the `192.168.10.0/24` network). Verify they receive correct IP/Gateway/DNS from `k8-svc`.

### C. K3s Cluster Installation & Configuration

1.  **Install K3s:** Follow K3s HA documentation (embedded etcd). Install on CPs first, then join the worker.
2.  **Configure `kubectl` Access:** Copy kubeconfig from a CP, modify the `server` address to `https://192.168.1.89:6443`, and test access.
3.  **Install MetalLB:** Apply manifests, configure `IPAddressPool` (e.g., `192.168.10.100-192.168.10.150`) and `L2Advertisement`.
4.  **Deploy Test Service (e.g., Grafana):** Deploy application, create `Service` of `Type=LoadBalancer`. Verify IP assignment from MetalLB.
5.  **Certificate Management:** Create Kubernetes TLS secret from your custom certificate files (`grafana.komebacklabs.lan`). Configure the service/ingress to use this secret for TLS termination.

### D. Pi-hole & Client Configuration

1.  **Pi-hole DNS Records:** Create A-records pointing service hostnames (`k8s-api.komebacklabs.lan`, `grafana.komebacklabs.lan`, etc.) to the HAProxy IP (`192.168.1.89`).
2.  **Client Machine:** Ensure it uses Pi-hole for DNS and trusts your custom Root CA certificate.

### E. Verification Steps

*   Verify network connectivity between all components (pings).
*   Verify DNS resolution (internal cluster, external internet, `komebacklabs.lan`) from K3s nodes.
*   Verify DNS resolution on the client machine.
*   Verify `kubectl` access via HAProxy.
*   Verify access to deployed services (e.g., `https://grafana.komebacklabs.lan`) via HAProxy with valid TLS.