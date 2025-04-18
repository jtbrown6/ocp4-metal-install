# K3s Lab Environment Overview

## Goal

The primary goal is to deploy a highly available K3s Kubernetes cluster (initially 2 control planes, 1 worker) within a dedicated internal network segment. This cluster should be accessible from the main local area network (LAN) via a dedicated service VM acting as a gateway and load balancer. The setup aims to utilize existing network infrastructure (Pi-hole DNS) and custom certificates while following Kubernetes best practices.

## Network Architecture

*   **Main LAN:** `192.168.1.0/24`
    *   Existing Router/Gateway: `192.168.1.1`
    *   Existing DNS Server (Pi-hole): `192.168.1.53`
*   **Internal K3s Network:** `192.168.10.0/24`
    *   This network hosts the K3s nodes exclusively.
    *   Gateway for this network: `192.168.10.1` (Provided by `k8-svc` VM).

## Core Components

### `k8-svc` VM

*   **Operating System:** Ubuntu 22.04
*   **Purpose:** Acts as the central management and access point between the Main LAN and the Internal K3s Network.
*   **Network Interfaces:**
    *   `ens192` (or similar): Connected to Main LAN, Static IP `192.168.1.89`.
    *   `ensXXX` (or similar): Connected to Internal K3s Network, Static IP `192.168.10.1`.
*   **Key Services & Roles:**
    *   **HAProxy:**
        *   Listens on `192.168.1.89`.
        *   Load balances K3s API traffic (port 6443) to internal control plane nodes (`192.168.10.x:6443`).
        *   Forwards HTTP/HTTPS traffic (ports 80/443) to internal K3s services/ingress (via MetalLB IPs like `192.168.10.100`). Acts as the entry point for accessing cluster applications from the Main LAN.
    *   **DHCP Server (`isc-dhcp-server`):**
        *   Serves the `192.168.10.0/24` network.
        *   Assigns static IP addresses to K3s nodes based on MAC addresses.
        *   Provides `192.168.10.1` as the gateway and DNS server to K3s nodes.
    *   **DNS Forwarder (`systemd-resolved`):**
        *   Listens on `192.168.10.1`.
        *   Receives DNS queries from K3s nodes.
        *   Forwards all queries upstream to the Pi-hole (`192.168.1.53`).
    *   **Routing & NAT:**
        *   Enables IP forwarding between interfaces.
        *   Provides Network Address Translation (NAT/Masquerading) for traffic from the `192.168.10.0/24` network destined for the Main LAN or the internet, allowing K3s nodes outbound connectivity.
    *   **NFS Server (Potential Role):** Can be configured to provide NFS shares (e.g., for PersistentVolumes) accessible by K3s nodes within the `192.168.10.0/24` network if needed.

### K3s Cluster Nodes

*   **Operating System:** Ubuntu 22.04
*   **Nodes:** 2 Control Plane, 1 Worker (initially).
*   **Network:** Connected only to the `192.168.10.0/24` network.
*   **IP Configuration:** Receive static IPs, gateway (`192.168.10.1`), and DNS server (`192.168.10.1`) via DHCP from the `k8-svc` VM.

### Kubernetes Cluster Configuration

*   **Installation:** K3s High Availability (HA) with embedded etcd.
*   **API Access:** Accessed via HAProxy: `https://192.168.1.89:6443`.
*   **Load Balancing (Internal):** MetalLB configured in L2 mode to provide LoadBalancer services within the `192.168.10.0/24` network (e.g., IP Pool `192.168.10.100-192.168.10.150`).
*   **Ingress:** An Ingress controller (e.g., Traefik - default in K3s, or Nginx Ingress) will be deployed later. It will receive an external IP from MetalLB and handle application routing and TLS termination.

## DNS and Certificates

*   **Domain:** `komebacklabs.lan`
*   **DNS Resolution (LAN):**
    *   Pi-hole (`192.168.1.53`) handles DNS for the `192.168.1.0/24` network.
    *   A-Records for all user-facing K3s services (e.g., `grafana.komebacklabs.lan`, `k8s-api.komebacklabs.lan`) must point to the HAProxy IP (`192.168.1.89`).
*   **DNS Resolution (K3s Nodes):**
    *   Nodes use `192.168.10.1` (`k8-svc`) as their DNS server.
    *   `k8-svc` forwards queries to Pi-hole (`192.168.1.53`).
    *   Internal cluster DNS (`*.cluster.local`) is handled by K3s CoreDNS.
*   **Certificates:**
    *   Custom Root CA and Subordinate CA certificates are used for the `komebacklabs.lan` domain.
    *   TLS termination for applications will occur *inside* the K3s cluster, managed by the Ingress controller or individual services (like Grafana initially).
    *   Client machines and browsers on the Main LAN must trust the custom Root CA.

## User Access Flow

1.  **Application Access (e.g., Grafana):**
    *   User requests `https://grafana.komebacklabs.lan`.
    *   Client DNS query goes to Pi-hole, resolves to `192.168.1.89`.
    *   HTTPS request hits HAProxy on `192.168.1.89:443`.
    *   HAProxy forwards TCP traffic to the internal MetalLB IP for Grafana (e.g., `192.168.10.100:443`).
    *   Grafana (or Ingress) terminates TLS using the `grafana.komebacklabs.lan` certificate and serves the content.
2.  **`kubectl` Access:**
    *   User configures `kubectl` with `server: https://192.168.1.89:6443`.
    *   Client DNS query for `k8s-api.komebacklabs.lan` (if used) goes to Pi-hole, resolves to `192.168.1.89`.
    *   API request hits HAProxy on `192.168.1.89:6443`.
    *   HAProxy load balances TCP traffic to internal control plane nodes (`192.168.10.x:6443`).