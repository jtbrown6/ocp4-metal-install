# K3s Ubuntu Lab Setup Guide - Part 2

## Introduction

Welcome to the K3s Ubuntu Lab setup guide! This document provides step-by-step instructions for deploying a highly available K3s cluster using Ubuntu 22.04 VMs, managed by a dedicated service VM (`k8-svc`) acting as a router, load balancer, and service provider for the internal cluster network.

**Goal:** To create a functional K3s environment accessible from your main LAN (`192.168.1.0/24`), leveraging an existing Pi-hole DNS (`192.168.1.53`) and custom certificates for the `komebacklabs.lan` domain.

**Prerequisites:**

*   An Ubuntu 22.04 VM designated as `k8-svc`.
*   Access to your hypervisor to configure VM network interfaces.
*   SSH access to the `k8-svc` VM.
*   Basic familiarity with Linux command line and networking concepts.

---

## Part 5: K3s Node Preparation and Cluster Installation

With the `k8-svc` VM providing essential network services (DHCP, DNS forwarding, Routing, Load Balancing), we can now prepare the virtual machines for the K3s nodes and install the cluster. We'll set up a High Availability (HA) control plane using K3s's embedded etcd.

**Objective:** Install Ubuntu on the K3s node VMs, ensure they get correct network configuration via DHCP, and install K3s in an HA configuration.

**Prerequisites:**

*   Three Ubuntu 22.04 VMs created (2 for Control Plane, 1 for Worker).
*   Each VM should have only *one* network interface, connected to the same network segment as the `k8-svc` VM's internal interface (`ens35` / `192.168.10.0/24`).
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
        --tls-san 192.168.1.89 \
        --disable traefik
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
        --tls-san 192.168.1.89 \
        --disable traefik
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

## Part 7: Deploy Nginx Ingress & Sample Application

With the cluster infrastructure (K3s, HAProxy, MetalLB) in place, and having disabled the default Traefik ingress during K3s installation, we will now install the Nginx Ingress Controller. Afterwards, we'll deploy a sample application (Grafana) and make it securely accessible from your main LAN using its hostname (`grafana.komebacklabs.lan`) via Nginx Ingress.

**Objective:** Install Nginx Ingress Controller, deploy Grafana, expose Grafana via Nginx Ingress using a MetalLB LoadBalancer IP for the controller, configure Pi-hole DNS, create a TLS secret, and enable TLS termination via Nginx Ingress.

**Prerequisites:**

*   Working K3s cluster with Traefik disabled and MetalLB configured (Part 6 completed).
*   `kubectl` access configured.
*   Helm installed on your client machine.
*   Access to your Pi-hole admin interface.
*   Your custom certificate files for `grafana.komebacklabs.lan` (e.g., `grafana.crt`, `grafana.key`) and the Root CA certificate (`ca.crt`) available on your client machine.

### Step 7.1: Install Nginx Ingress Controller

We will install the official Nginx Ingress Controller using its Helm chart. This controller will create a `LoadBalancer` service that gets an IP from MetalLB, becoming the entry point for HTTP/HTTPS traffic into the cluster (forwarded by HAProxy).

1.  **Add Nginx Ingress Helm Repository:**
    ```bash
    # On your client machine
    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    helm repo update
    ```

2.  **Install the Helm Chart:**
    ```bash
    # On your client machine
    helm install nginx-ingress ingress-nginx/ingress-nginx \
      --namespace ingress-nginx \
      --create-namespace \
      --set controller.service.type=LoadBalancer
    ```
    *   **Explanation:**
        *   `helm install nginx-ingress ...`: Installs the chart with the release name `nginx-ingress`.
        *   `--namespace ingress-nginx --create-namespace`: Installs the controller into its own dedicated namespace.
        *   `--set controller.service.type=LoadBalancer`: Explicitly tells the chart to create a `LoadBalancer` type service for the controller. MetalLB will assign an IP to this service.

3.  **Verify Nginx Ingress Installation:** Wait a few moments for the pods and service to be ready.
    ```bash
    # Check pods in the ingress-nginx namespace
    kubectl get pods -n ingress-nginx -w

    # Check the service and note its EXTERNAL-IP
    kubectl get svc -n ingress-nginx nginx-ingress-ingress-nginx-controller -w
    ```
    Wait until the pods are `Running` and the `nginx-ingress-ingress-nginx-controller` service has an `EXTERNAL-IP` assigned from your MetalLB pool (e.g., `192.168.10.101`). **Make a note of this specific IP address.** Press Ctrl+C to exit the watches.

4.  **(Recommended) Update HAProxy Backend:** For optimal routing, update your HAProxy configuration (`/etc/haproxy/haproxy.cfg` on `k8-svc`) so that the `http_ingress_backend` and `https_ingress_backend` sections point to the **EXTERNAL-IP** and corresponding ports (80/443) of the `nginx-ingress-ingress-nginx-controller` service you just noted, instead of pointing directly to worker nodes.
    *   Example change in `haproxy.cfg` (replace `192.168.10.101` with the actual Nginx LoadBalancer IP):
        ```cfg
        backend http_ingress_backend
            mode tcp
            balance source
            # Point to the Nginx Ingress Controller LoadBalancer Service IP
            server nginx-ingress-lb 192.168.10.101:80 check

        backend https_ingress_backend
            mode tcp
            balance source
            # Point to the Nginx Ingress Controller LoadBalancer Service IP
            server nginx-ingress-lb 192.168.10.101:443 check
        ```
    *   After editing `haproxy.cfg`, reload HAProxy: `sudo systemctl reload haproxy` on `k8-svc`.
    *   *Note: While pointing HAProxy directly to worker nodes might work initially if Nginx pods land there, pointing to the stable LoadBalancer IP is more reliable.*

### Step 7.2: Deploy Grafana (Example Application)

This step remains largely the same, deploying Grafana and its internal `ClusterIP` service.

1.  **Create Grafana Manifest (`grafana-deployment.yaml`):** Create this file on your client machine. *Note: The Service type is now `ClusterIP` as Nginx Ingress will handle external exposure.*

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
        - name: http # Service port name
          protocol: TCP
          port: 3000 # Service port matches container port
          targetPort: 3000 # Target Grafana's internal HTTP port
      type: ClusterIP # Only needs to be reachable within the cluster by Nginx Ingress
    ```

    *   **Explanation:**
        *   `Deployment`: Unchanged.
        *   `Service`: Now uses `type: ClusterIP` because external access is handled by the Nginx Ingress Controller, not this service directly. The service port is set to `3000` to match Grafana's container port.

2.  **Apply the Manifest:**
    ```bash
    kubectl apply -f grafana-deployment.yaml
    ```

3.  **Verify Deployment and Service:**
    ```bash
    kubectl get deployment grafana
    kubectl get service grafana-svc
    ```
    Ensure the deployment is available and the service exists with a `CLUSTER-IP`.

### Step 7.3: Configure Pi-hole DNS

This step remains the same. Point the desired hostname (`grafana.komebacklabs.lan`) to the HAProxy IP address (`192.168.1.89`).

1.  **Log in to Pi-hole:** Access your Pi-hole web admin interface.
2.  **Navigate to Local DNS Records:** Find the section for managing local DNS A or CNAME records.
3.  **Add A-Record:** Create a new **A-Record**:
    *   **Domain:** `grafana.komebacklabs.lan`
    *   **IP Address:** `192.168.1.89` (The **HAProxy** LAN IP)
4.  **Save:** Add the record.

    *   **Explanation:** Client requests for `grafana.komebacklabs.lan` go to HAProxy (`192.168.1.89`). HAProxy forwards ports 80/443 to the Nginx Ingress Controller's LoadBalancer IP (e.g., `192.168.10.101`). Nginx Ingress then routes the request based on the hostname and path to the correct internal service (Grafana).

### Step 7.4: Create Kubernetes TLS Secret

This step is unchanged. Store your custom certificate and key in a Kubernetes secret.

1.  **Prepare Certificate Files:** Ensure you have `grafana.crt` and `grafana.key` on your client machine.
2.  **Create the Secret:**
    ```bash
    kubectl create secret tls grafana-tls \
      --cert=path/to/your/grafana.crt \
      --key=path/to/your/grafana.key \
      -n default # Use the same namespace as Grafana
    ```
3.  **Verify Secret Creation (Optional):**
    ```bash
    kubectl get secret grafana-tls -n default
    ```

### Step 7.5: Configure TLS Termination via Nginx Ingress

Create an Ingress resource that tells the Nginx Ingress Controller how to handle requests for `grafana.komebacklabs.lan`.

1.  **Create Ingress Manifest (`grafana-ingress.yaml`):** Create this file on your client machine. Note the addition of `ingressClassName`.

    ```yaml
    # grafana-ingress.yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: grafana-ingress
      namespace: default # Use the same namespace as Grafana
      # Optional: Add Nginx specific annotations if needed later
      # annotations:
      #   nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
    spec:
      ingressClassName: nginx # Specify Nginx Ingress Controller
      tls:
      - hosts:
          - grafana.komebacklabs.lan
        secretName: grafana-tls # Reference the secret created earlier
      rules:
      - host: grafana.komebacklabs.lan
        http:
          paths:
          - path: /
            pathType: Prefix # Match all paths under the host
            backend:
              service:
                name: grafana-svc # Target the Grafana ClusterIP Service
                port:
                  number: 3000 # Target the service port (which is 3000)
    ```

    *   **Explanation:**
        *   `kind: Ingress`: Defines routing rules.
        *   `spec.ingressClassName: nginx`: **Crucial:** Tells Kubernetes that this Ingress resource should be handled by the installed Nginx Ingress Controller.
        *   `spec.tls`: Configures TLS termination using the `grafana-tls` secret for the specified host.
        *   `spec.rules`: Defines routing for `grafana.komebacklabs.lan`.
        *   `backend.service`: Specifies the target service (`grafana-svc`) and the target service *port* (`3000`). Nginx Ingress will terminate TLS and forward plain HTTP traffic to `grafana-svc:3000`.

2.  **Apply the Ingress Manifest:**
    ```bash
    kubectl apply -f grafana-ingress.yaml
    ```

    *   **How it works now:** HAProxy forwards the TCP connection for port 443 to the Nginx Ingress Controller's LoadBalancer service. Nginx Ingress sees the incoming TLS connection for `grafana.komebacklabs.lan`, uses the `grafana-tls` secret to terminate TLS, and then forwards the decrypted HTTP request to the `grafana-svc` ClusterIP service on port 3000.

### Step 7.6: Ensure Client Trusts Root CA

This step is unchanged. Import your `ca.crt` into your client machine's trust store.

1.  **Import Root CA:** Follow OS/browser specific steps.

### Step 7.7: Test Access

This step is unchanged.

1.  **Clear DNS Cache (Optional):** Flush DNS on your client machine.
2.  **Access Grafana:** Open `https://grafana.komebacklabs.lan` in your browser.
3.  **Verify:** Check for Grafana login page and a valid, trusted HTTPS connection using your custom certificate.

---

This completes the setup using Nginx Ingress Controller! You have deployed K3s (without Traefik), installed Nginx Ingress, configured MetalLB, and exposed Grafana securely using HAProxy, Pi-hole, Nginx Ingress, and custom certificates.

---

## Part 8: Exposing Additional LoadBalancer Services via HAProxy

While Part 7 demonstrated exposing applications via the Nginx Ingress Controller (recommended for standard HTTP/HTTPS traffic using hostnames), you might occasionally need to expose a non-HTTP service or provide direct access to a specific `LoadBalancer` service IP via a dedicated port on your HAProxy instance (`k8-svc`).

This section details how to configure HAProxy on `k8-svc` to forward traffic from a specific port on its LAN IP (`192.168.1.89`) directly to a Kubernetes `LoadBalancer` service IP within the `192.168.10.0/24` network.

**Objective:** Make a Kubernetes `LoadBalancer` service accessible from the main LAN (`192.168.1.0/24`) by mapping a unique port on `k8-svc` (`192.168.1.89`) to the service's internal LoadBalancer IP and port.

**Prerequisites:**

*   A Kubernetes service of `type: LoadBalancer` deployed in your cluster.
*   MetalLB has assigned an `EXTERNAL-IP` to this service from your configured pool (e.g., `192.168.10.X`).
*   SSH access to `k8-svc`.

**Procedure:**

Let's assume you have deployed a new service named `my-app-svc` in the `my-app-ns` namespace, and it received the LoadBalancer IP `192.168.10.140` and exposes port `1234`. We want to access this service via `http://192.168.1.89:8082` (or just `192.168.1.89:8082` if it's a non-HTTP TCP service).

1.  **Identify Service IP and Port:** Confirm the `EXTERNAL-IP` and `PORT(S)` for your service:
    ```bash
    kubectl get svc my-app-svc -n my-app-ns
    # Example Output:
    # NAME         TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)          AGE
    # my-app-svc   LoadBalancer   10.43.X.Y      192.168.10.140   1234:30123/TCP   5m
    ```
    Note the `EXTERNAL-IP` (`192.168.10.140`) and the service port (`1234`).

2.  **Choose HAProxy Port:** Select a unique, unused port on `k8-svc`'s LAN interface (`192.168.1.89`) that you will use to access the service externally. For this example, we'll use `8082`.

3.  **SSH into `k8-svc`:**
    ```bash
    ssh user@192.168.1.89 # Or however you connect to k8-svc
    ```

4.  **Edit HAProxy Configuration:** Open the HAProxy configuration file for editing:
    ```bash
    sudo nano /etc/haproxy/haproxy.cfg
    ```

5.  **Add Frontend Block:** Add a new `frontend` section to listen on your chosen external port (`8082`).
    *   Set `mode` to `http` if it's a web service, or `tcp` for generic TCP traffic.
    *   Point `default_backend` to a corresponding new backend name (e.g., `my_app_backend`).

    ```cfg
    #---------------------------------------------------------------------
    # Frontend: Direct My App Access (Port 8082)
    #---------------------------------------------------------------------
    frontend my_app_frontend
        bind 192.168.1.89:8082 # Listen on LAN IP, chosen external port
        mode http             # Use 'tcp' for non-HTTP services
        option httplog        # Use 'tcplog' if mode is tcp
        default_backend my_app_backend
    ```

6.  **Add Backend Block:** Add a new `backend` section with the name referenced in the frontend.
    *   Set `mode` to match the frontend (`http` or `tcp`).
    *   Add a `server` line pointing to the service's `EXTERNAL-IP` (`192.168.10.140`) and service port (`1234`).
    *   Include `check` to enable health checks.

    ```cfg
    #---------------------------------------------------------------------
    # Backend: Direct My App Access
    #---------------------------------------------------------------------
    backend my_app_backend
        mode http             # Use 'tcp' if mode is tcp
        balance roundrobin    # Or 'source' if session persistence is needed
        # Point directly to the My App LoadBalancer service IP and Port
        server my-app-lb 192.168.10.140:1234 check
    ```

7.  **Save and Close:** Save the changes to `/etc/haproxy/haproxy.cfg`.

8.  **Reload HAProxy:** Apply the new configuration without dropping existing connections:
    ```bash
    sudo systemctl reload haproxy
    ```
    Check the status to ensure it reloaded successfully:
    ```bash
    sudo systemctl status haproxy
    ```

9.  **Test Access:** You should now be able to access your service from your main LAN via the HAProxy IP and the port you configured:
    *   If `mode http`: `http://192.168.1.89:8082`
    *   If `mode tcp`: Connect using a suitable client to `192.168.1.89:8082`

**Considerations:**

*   **Port Conflicts:** Ensure the port chosen on `192.168.1.89` (e.g., `8082`) is not already in use by HAProxy or another service on `k8-svc`.
*   **Mode:** Use `mode tcp` for non-HTTP TCP services. Use `mode http` for HTTP services, which allows for more advanced HTTP-specific options later if needed (like header manipulation).
*   **Ingress Controller:** Remember that for standard web applications, using the Nginx Ingress Controller (Part 7) is generally preferred. It allows you to use hostnames (like `my-app.komebacklabs.lan`) routed through standard ports (80/443) and handles TLS termination centrally. Exposing services directly via HAProxy ports is best reserved for specific use cases or non-HTTP traffic.

---

## Part 9: Configure Persistent Storage (NFS)

For stateful applications (like databases, GitLab, Harbor data, etc.) to run effectively in Kubernetes, they need persistent storage that survives pod restarts. We will configure the `k8-svc` VM as an NFS server and deploy a dynamic provisioner in K3s that automatically creates PersistentVolumes (PVs) on this NFS share when applications request storage via PersistentVolumeClaims (PVCs).

**Objective:** Enable dynamic persistent storage provisioning for the K3s cluster using an NFS share hosted on `k8-svc`.

**Prerequisites:**

*   Working K3s cluster accessible via `kubectl` (Part 7 completed).
*   SSH access to `k8-svc` and all K3s nodes.

### Step 9.1: Configure NFS Server on `k8-svc`

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

### Step 9.2: Install NFS Client on K3s Nodes

The K3s nodes need the NFS client utilities to be able to mount the NFS share.

1.  **Install `nfs-common`:** Run this on **all** K3s nodes (`k3s-cp-1`, `k3s-cp-2`, `k3s-w-1`).
    ```bash
    # On EACH K3s node
    sudo apt update
    sudo apt install nfs-common -y
    ```
    *(Note: This might already be installed as part of Ubuntu Server, but running the command ensures it's present.)*

### Step 9.3: Deploy NFS Subdir External Provisioner

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

### Step 9.4: Verify Dynamic Provisioning

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