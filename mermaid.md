graph TD

%% Main LAN
subgraph "Main LAN (192.168.1.0/24)"
    UserClient["Client Machine"]
    PiHole["Pi-hole DNS (192.168.1.53)"]
    Router["LAN Router/GW"]
end

%% k8-svc VM
subgraph "k8-svc VM (Ubuntu 22.04)"
    direction LR

    subgraph "Network Interfaces"
        NIC_LAN["NIC 1: ens192 - 192.168.1.89"]
        NIC_K8S["NIC 2: ensXXX - 192.168.10.1"]
    end

    subgraph "Services"
        DHCP["DHCP Server (isc-dhcp-server)"]
        DNSFwd["DNS Forwarder (systemd-resolved/dnsmasq)"]
        HAP["HAProxy"]
        RoutingNAT{{"Routing & NAT"}}
    end

    NIC_LAN -- "Connected" --> MainLANNet["Main LAN Network"]
    NIC_K8S -- "Connected" --> K8SNet["Internal Network"]
    RoutingNAT -- "Routes Traffic" --> NIC_LAN
    RoutingNAT -- "Routes Traffic" --> NIC_K8S
    DHCP -- "Serves IPs" --> K8SNet
    DNSFwd -- "Listens/Forwards" --> NIC_K8S
    HAP -- "Listens/Forwards" --> NIC_LAN
end

%% Kubernetes Internal Network
subgraph "K8s Internal Network (192.168.10.0/24)"
    K8SNet

    subgraph "K3s Cluster (Ubuntu 22.04)"
        CP1["K3s CP 1 (DHCP IP)"]
        CP2["K3s CP 2 (DHCP IP)"]
        W1["K3s Worker 1 (DHCP IP)"]
        MetalLB["MetalLB IP Pool (192.168.10.x Range)"]
        Ingress["Future Ingress Controller"]
        GrafanaSvc["Grafana Service (Type=LB - 192.168.10.100)"]

        CP1 & CP2 & W1 -- "Network Config" --> DHCP
        CP1 & CP2 & W1 -- "DNS Queries" --> DNSFwd
        GrafanaSvc -- "Gets IP From" --> MetalLB
        Ingress -- "Gets IP From" --> MetalLB
    end

    K8SNet -- "Connects" --> CP1
    K8SNet -- "Connects" --> CP2
    K8SNet -- "Connects" --> W1
end

%% Flow Descriptions
UserClient -- "DNS Query (grafana.komebacklabs.lan)" --> PiHole
PiHole -- "A-Record (grafana → 192.168.1.89)" --> UserClient
UserClient -- "HTTPS Request (grafana.komebacklabs.lan)" --> HAP
HAP -- "TCP Forward (Port 443)" --> GrafanaSvc
GrafanaSvc -- "TLS Termination & Response" --> HAP
HAP -- "Encrypted Response" --> UserClient

UserClient -- "kubectl" --> PiHole
PiHole -- "A-Record (k8s-api.komebacklabs.lan → 192.168.1.89)" --> UserClient
UserClient -- "API Request (Port 6443)" --> HAP
HAP -- "TCP Forward (Port 6443)" --> CP1
HAP -- "TCP Forward (Port 6443)" --> CP2

CP1 & CP2 & W1 -- "Internet Traffic" --> RoutingNAT
RoutingNAT -- "NAT" --> Router

CP1 & CP2 & W1 -- "DNS (komebacklabs.lan)" --> DNSFwd
DNSFwd -- "Forward" --> PiHole
CP1 & CP2 & W1 -- "DNS (other)" --> DNSFwd
DNSFwd -- "Forward" --> PiHole
InternetDNS["Internet DNS"]
PiHole -- "Forward" --> InternetDNS

MainLANNet --- Router
