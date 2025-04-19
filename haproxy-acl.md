# HAProxy Hostname Routing with ACLs Guide

This guide explains how to configure HAProxy on your `k8-svc` machine to route incoming HTTP traffic to different backend services based on the requested hostname (e.g., routing `grafana.komebacklabs.lan` to Grafana and `prometheus.komebacklabs.lan` to Prometheus) using Access Control Lists (ACLs).

**Goal:** Access different internal services (like Grafana and Prometheus) using unique hostnames through the standard HTTP port (80) on your HAProxy instance (`192.168.1.89`).

**When to Use This:**

*   You want HAProxy itself to handle hostname routing on port 80/443 instead of relying solely on an Ingress Controller like Nginx.
*   You are exposing services directly via HAProxy and need to differentiate them by hostname.

**Note:** If you are already using an Ingress Controller like Nginx (as detailed in the main `setup_guide.md`), letting the Ingress Controller handle hostname routing is often simpler and the recommended approach for standard web traffic. This guide presents an alternative where HAProxy performs the hostname routing directly.

## Prerequisites

1.  **HAProxy Installed:** HAProxy is installed and running on your `k8-svc` machine (`192.168.1.89`).
2.  **Pi-hole Access:** You have access to your Pi-hole admin interface to add DNS records.
3.  **Service Hostnames:** You have decided on the hostnames you want to use (e.g., `grafana.komebacklabs.lan`, `prometheus.komebacklabs.lan`).
4.  **Backend Service IPs/Ports:** You know the internal IP addresses and ports of the services you want to route to (e.g., the LoadBalancer IPs assigned by MetalLB like `192.168.10.130:80` for Grafana, `192.168.10.131:9090` for Prometheus).

## Step 1: Configure Pi-hole DNS

DNS resolves the hostname to the IP address of your HAProxy server. HAProxy then uses the hostname information within the HTTP request to route further.

1.  **Log in to Pi-hole:** Access your Pi-hole web admin interface.
2.  **Navigate to Local DNS Records:** Find the section for managing local DNS records (usually under "Local DNS").
3.  **Add A-Records:** Create a new **A-Record** for *each* hostname you want to use, pointing them *all* to your HAProxy LAN IP (`192.168.1.89`):
    *   **Domain:** `grafana.komebacklabs.lan` -> **IP Address:** `192.168.1.89`
    *   **Domain:** `prometheus.komebacklabs.lan` -> **IP Address:** `192.168.1.89`
    *   *(Add records for any other hostnames you plan to route)*
4.  **Save:** Add/Save the records.

## Step 2: Configure HAProxy (`/etc/haproxy/haproxy.cfg`)

This involves modifying the frontend that listens on port 80 to operate in `mode http` and adding ACLs and routing rules.

1.  **SSH into `k8-svc`:** Connect to your HAProxy server.
2.  **Edit HAProxy Configuration:** Open the configuration file:
    ```bash
    sudo nano /etc/haproxy/haproxy.cfg
    ```
3.  **Modify/Add Configuration:** Update the relevant sections. Below is an example incorporating the necessary changes into your existing structure.

    ```haproxy
    # --- Existing Global and Defaults Sections (Keep As Is) ---
    global
        # ... your global settings ...
    defaults
        # ... your default settings ...
        # Ensure default mode is tcp if most backends are tcp,
        # but frontends/backends using http features need 'mode http' explicitly.

    # --- Existing K3s API Frontend/Backend (Keep As Is) ---
    frontend k3s_api_frontend
        bind 192.168.1.89:6443
        mode tcp
        option tcplog
        default_backend k3s_api_backend

    backend k3s_api_backend
        mode tcp
        option tcp-check
        balance source
        server lab-k8-cp1 192.168.10.11:6443 check
        server lab-k8-cp2 192.168.10.12:6443 check

    # --- MODIFIED HTTP Frontend (Port 80) for ACL Routing ---
    frontend http_acl_frontend # Renamed for clarity
        bind 192.168.1.89:80
        mode http             # <<< IMPORTANT: Changed to http mode
        option httplog        # Use httplog for http mode

        # --- ACL Definitions ---
        acl host_grafana hdr(host) -i grafana.komebacklabs.lan
        acl host_prometheus hdr(host) -i prometheus.komebacklabs.lan
        # Add more ACLs here for other hostnames

        # --- Routing Rules (use_backend) ---
        # Order matters! First match wins.
        use_backend grafana_http_backend if host_grafana
        use_backend prometheus_http_backend if host_prometheus
        # Add more 'use_backend' rules here for other hostnames

        # --- Default Backend ---
        # If no hostname ACL matches, traffic goes here.
        # This could point to your Nginx Ingress LB if you still use it as a fallback.
        # Or, you might remove the default if all traffic MUST match a hostname.
        default_backend http_ingress_fallback_backend # Example fallback

    # --- NEW HTTP Backends for Grafana & Prometheus ---
    backend grafana_http_backend
        mode http
        balance roundrobin
        # Point to the Grafana LoadBalancer service IP and Port
        server grafana-lb 192.168.10.130:80 check

    backend prometheus_http_backend
        mode http
        balance roundrobin
        # Point to the Prometheus LoadBalancer service IP and Port
        server prometheus-lb 192.168.10.131:9090 check

    # --- Fallback Backend (Example: Points to Nginx Ingress) ---
    # Ensure this backend also uses 'mode http' if the frontend uses it,
    # OR keep it 'mode tcp' if Nginx handles everything including TLS.
    # Using 'mode tcp' here might be simpler if Nginx is the target.
    backend http_ingress_fallback_backend
        mode tcp # Keep as TCP if forwarding raw connection to Nginx LB
        balance source
        # !!! Replace with your actual Nginx Ingress LB IP !!!
        server nginx-ingress-lb 192.168.10.NGINX_LB_IP:80 check

    # --- Existing HTTPS Frontend/Backend (Port 443) ---
    # NOTE: Applying hostname ACLs to HTTPS requires TLS termination
    #       at HAProxy, which adds complexity (certificates, etc.).
    #       It's often easier to keep this in 'mode tcp' and let
    #       the Nginx Ingress Controller handle TLS termination and routing.
    frontend https_ingress_frontend
        bind 192.168.1.89:443
        mode tcp # Keeping as TCP is simpler if Nginx handles TLS
        option tcplog
        default_backend https_ingress_backend

    backend https_ingress_backend
        mode tcp # Keeping as TCP
        balance source
        # !!! Replace with your actual Nginx Ingress LB IP !!!
        server nginx-ingress-lb 192.168.10.NGINX_LB_IP:443 check

    # --- Existing Direct Access Frontends/Backends (Keep As Is) ---
    # (grafana_direct_frontend, prometheus_direct_frontend, etc. on ports 8081, 9091)
    frontend grafana_direct_frontend
        bind 192.168.1.89:8081 # Listen on LAN IP, port 8081
        mode http
        option httplog
        default_backend grafana_direct_backend

    backend grafana_direct_backend
        mode http
        balance roundrobin
        server grafana-lb 192.168.10.130:80 check

    frontend prometheus_direct_frontend
        bind 192.168.1.89:9091 # Listen on LAN IP, port 9091
        mode http
        option httplog
        default_backend prometheus_direct_backend

    backend prometheus_direct_backend
        mode http
        balance roundrobin
        server prometheus-lb 192.168.10.131:9090 check
    # --- Existing Stats Page (Keep As Is) ---
    listen stats
        bind 192.168.1.89:9000
        mode http
        stats enable
        stats uri /stats
        stats refresh 10s
        stats admin if TRUE
        # stats auth username:password
    ```

    **Key Configuration Points:**
    *   **`frontend http_acl_frontend`**:
        *   `bind 192.168.1.89:80`: Listens on the standard HTTP port.
        *   `mode http`: **Essential** for reading HTTP headers like `Host`.
        *   `acl host_grafana hdr(host) -i grafana.komebacklabs.lan`: Defines a condition named `host_grafana` that is true if the `Host` header matches `grafana.komebacklabs.lan` (case-insensitive).
        *   `use_backend grafana_http_backend if host_grafana`: If the `host_grafana` condition is true, use the `grafana_http_backend`.
        *   `default_backend http_ingress_fallback_backend`: If no `use_backend` condition matches, use this backend.
    *   **`backend grafana_http_backend` / `prometheus_http_backend`**:
        *   `mode http`: Matches the frontend mode.
        *   `server grafana-lb 192.168.10.130:80 check`: Defines the actual backend server (your Grafana LoadBalancer service).

4.  **Save and Close:** Save the changes to `/etc/haproxy/haproxy.cfg`.

## Step 3: Validate and Reload HAProxy

1.  **Validate Configuration:** Before reloading, check for syntax errors:
    ```bash
    sudo haproxy -c -f /etc/haproxy/haproxy.cfg
    ```
    If it reports "Configuration file is valid", proceed. Otherwise, fix the errors indicated.

2.  **Reload HAProxy:** Apply the new configuration gracefully:
    ```bash
    sudo systemctl reload haproxy
    ```
    Check the status:
    ```bash
    sudo systemctl status haproxy
    ```

## Step 4: Test Access

Open your web browser and try accessing the hostnames you configured:

*   `http://grafana.komebacklabs.lan` (should route to Grafana)
*   `http://prometheus.komebacklabs.lan` (should route to Prometheus)

If you configured a default backend, accessing `http://192.168.1.89` directly (or any other hostname pointing to it but not matching an ACL) should go to that default backend.

## HTTPS Considerations & Using Certificates

Applying hostname-based ACL routing directly in HAProxy for HTTPS (port 443) requires HAProxy to decrypt the incoming TLS traffic so it can read the HTTP headers (like the `Host` header). This process is called **TLS Termination**. To enable this, you need to provide HAProxy with the appropriate SSL/TLS certificate and private key.

Using a certificate issued by your internal Certificate Authority (CA) for `*.komebacklabs.lan` (or specific hostnames like `grafana.komebacklabs.lan`) is exactly what's needed for this.

### How Adding a Certificate Changes Configuration

1.  **Certificate File:** You need your certificate (including any intermediate certificates) and your private key combined into a single `.pem` file that HAProxy can read. Let's assume you create this file at `/etc/haproxy/certs/komebacklabs.lan.pem`. Ensure the `haproxy` user has read permissions for this file. The order within the `.pem` file typically should be: private key, then the server certificate, then any intermediate CA certificates.

2.  **Modify HTTPS Frontend:** You need to update the `frontend` section that listens on port 443:
    *   Change `mode` from `tcp` to `http`.
    *   Modify the `bind` line to include `ssl crt <path-to-your-pem-file>`.
    *   Add the same ACLs and `use_backend` rules as used in the HTTP frontend.

    ```haproxy
    # --- MODIFIED HTTPS Frontend (Port 443) for ACL Routing with TLS Termination ---
    frontend https_acl_frontend # Renamed for clarity
        bind 192.168.1.89:443 ssl crt /etc/haproxy/certs/komebacklabs.lan.pem # <<< Added ssl crt
        mode http             # <<< IMPORTANT: Changed to http mode
        option httplog

        # Add http-request redirect scheme https if needed to force HTTPS
        # http-request redirect scheme https unless { ssl_fc }

        # --- ACL Definitions (Same as HTTP frontend) ---
        acl host_grafana hdr(host) -i grafana.komebacklabs.lan
        acl host_prometheus hdr(host) -i prometheus.komebacklabs.lan
        # Add more ACLs here

        # --- Routing Rules (use_backend - Same as HTTP frontend) ---
        use_backend grafana_http_backend if host_grafana    # Reuse HTTP backends if they expect plain HTTP
        use_backend prometheus_http_backend if host_prometheus # Reuse HTTP backends
        # Add more 'use_backend' rules here

        # --- Default Backend ---
        # Point to a default backend, potentially the Nginx Ingress (expecting plain HTTP now)
        default_backend https_ingress_fallback_backend # Example fallback
    ```

3.  **Backend Considerations:**
    *   **Plain HTTP Backends:** When HAProxy terminates TLS, the connection from HAProxy to the backend server is typically plain HTTP. Your backend definitions (like `grafana_http_backend`, `prometheus_http_backend`) should point to the backend service's *HTTP* port (e.g., `192.168.10.130:80`). This is often the desired setup.
    *   **Re-encryption (HTTPS Backends):** If your backend services *require* HTTPS connections, you need to configure HAProxy to re-encrypt the traffic. This involves adding `ssl verify none` (for internal CAs, adjust verification as needed) or `ssl verify required ca-file <your-ca-bundle>` to the `server` lines in the backend definition. The backend would then point to the HTTPS port of the service (e.g., `server grafana-lb 192.168.10.130:443 check ssl verify none`). This adds overhead.
    *   **Nginx Ingress Fallback:** If your `default_backend` points to the Nginx Ingress Controller's LoadBalancer IP, you need to decide if Nginx should receive HTTP or HTTPS. If HAProxy terminates TLS, you should point the HAProxy backend to the Nginx *HTTP* port (usually port 80 on the Nginx LB IP) and ensure the backend mode is compatible (`http` or `tcp`).

### Summary of Changes with Certificate

*   **Enables HTTPS Hostname Routing:** You can now use ACLs based on `hdr(host)` in your port 443 frontend.
*   **Requires Certificate Management:** You need to place and manage the `.pem` file on the HAProxy server.
*   **Shifts TLS Termination:** TLS encryption/decryption happens at HAProxy instead of further downstream (like at Nginx Ingress or the application itself).
*   **Backend Connections:** Connections from HAProxy to your backend services will be plain HTTP by default unless you configure backend SSL/TLS re-encryption.

For setups using an Ingress Controller like Nginx, letting the Ingress Controller handle TLS termination (using Kubernetes Secrets for certificates) and hostname routing remains a very common and often simpler pattern, keeping HAProxy in `mode tcp` for port 443. However, terminating TLS at HAProxy provides flexibility if you need HAProxy to make decisions based on decrypted traffic.