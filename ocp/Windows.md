# OpenShift 4 Bare Metal Install with Windows Server 2019 DNS/DHCP

This guide provides step-by-step instructions for installing OpenShift 4 (OKD) on bare metal using Windows Server 2019 for DNS and DHCP services, and Ubuntu for HAProxy load balancing.

- [OpenShift 4 Bare Metal Install with Windows Server 2019 DNS/DHCP](#openshift-4-bare-metal-install-with-windows-server-2019-dnsdhcp)
  - [Architecture Overview](#architecture-overview)
  - [Download Software](#download-software)
  - [Prepare the 'Bare Metal' environment](#prepare-the-bare-metal-environment)
  - [Configure Windows Server 2019 DNS](#configure-windows-server-2019-dns)
  - [Configure Windows Server 2019 DHCP](#configure-windows-server-2019-dhcp)
  - [Configure Ubuntu HAProxy](#configure-ubuntu-haproxy)
  - [Generate and host install files](#generate-and-host-install-files)
  - [Deploy OpenShift](#deploy-openshift)
  - [Monitor the Bootstrap Process](#monitor-the-bootstrap-process)
  - [Remove the Bootstrap Node](#remove-the-bootstrap-node)
  - [Wait for installation to complete](#wait-for-installation-to-complete)
  - [Join Worker Nodes](#join-worker-nodes)
  - [Configure storage for the Image Registry](#configure-storage-for-the-image-registry)
  - [Create the first Admin user](#create-the-first-admin-user)
  - [Access the OpenShift Console](#access-the-openshift-console)
  - [Troubleshooting](#troubleshooting)

## Architecture Overview

This setup uses:
- Windows Server 2019 for DNS, DHCP, and routing (192.168.40.1)
- Ubuntu VM for HAProxy load balancing (192.168.40.2)
- Network: 192.168.40.0/24
- Domain: komebacklab.local

## Download Software

1. Download [CentOS 8 x86_64 image](https://www.centos.org/centos-linux/) for the HAProxy server
2. Download [Ubuntu 20.04 LTS Server](https://ubuntu.com/download/server) for the HAProxy server (alternative to CentOS)
3. Login to [RedHat OpenShift Cluster Manager](https://cloud.redhat.com/openshift)
4. Select 'Create Cluster' from the 'Clusters' navigation menu
5. Select 'RedHat OpenShift Container Platform'
6. Select 'Run on Bare Metal'
7. Download the following files:

   - Openshift Installer for Linux
   - Pull secret
   - Command Line Interface for Linux and your workstations OS
   - Red Hat Enterprise Linux CoreOS (RHCOS)
     - rhcos-X.X.X-x86_64-metal.x86_64.raw.gz
     - rhcos-X.X.X-x86_64-installer.x86_64.iso (or rhcos-X.X.X-x86_64-live.x86_64.iso for newer versions)

## Prepare the 'Bare Metal' environment

#### Install OCP Version 4.14
Use this [link](https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.14/4.14.0/) to install software files.
> VMware ESXi used in this guide

1. Copy the CentOS 8 or Ubuntu 20.04 iso to an ESXi datastore
2. Create a new Port Group called 'OCP' under Networking
    - (In case of VirtualBox choose "Internal Network" when creating each VM and give it the same name. ocp for instance)
    - (In case of ProxMox you may use the same network bridge and choose a specific VLAN tag. 50 for instance) 
3. Create 3 Control Plane virtual machines with minimum settings:
   - Name: ocp-cp-# (Example ocp-cp-1)
   - 4vcpu
   - 8GB RAM
   - 50GB HDD
   - NIC connected to the OCP network
   - Load the rhcos-X.X.X-x86_64-installer.x86_64.iso image into the CD/DVD drive
4. Create 2 Worker virtual machines (or more if you want) with minimum settings:
   - Name: ocp-w-# (Example ocp-w-1)
   - 4vcpu
   - 8GB RAM
   - 50GB HDD
   - NIC connected to the OCP network
   - Load the rhcos-X.X.X-x86_64-installer.x86_64.iso image into the CD/DVD drive
5. Create a Bootstrap virtual machine (this vm will be deleted once installation completes) with minimum settings:
   - Name: ocp-boostrap
   - 4vcpu
   - 8GB RAM
   - 50GB HDD
   - NIC connected to the OCP network
   - Load the rhcos-X.X.X-x86_64-installer.x86_64.iso image into the CD/DVD drive
6. Create a HAProxy virtual machine with minimum settings:
   - Name: ocp-haproxy
   - 2vcpu
   - 4GB RAM
   - 50GB HDD
   - NIC connected to the OCP network
   - Load the Ubuntu 20.04 or CentOS 8 iso image into the CD/DVD drive
7. Boot all virtual machines so they each are assigned a MAC address
8. Shut down all virtual machines except for 'ocp-haproxy'
9. Use the VMware ESXi dashboard to record the MAC address of each vm, these will be used later to set static IPs

## Configure Windows Server 2019 DNS

### 1. Installing DNS Role (if not already installed)

1. Open **Server Manager** and click on **Add roles and features**
2. Click **Next** until you reach the **Server Roles** page
3. Check the box for **DNS Server** and click **Add Features** when prompted
4. Click **Next** until you reach the **Confirmation** page
5. Click **Install** and wait for the installation to complete
6. Click **Close** when finished

### 2. Creating the Forward Lookup Zone

1. Open **Server Manager** and click on **Tools** in the top-right menu bar, then select **DNS**
2. In the DNS Manager console, right-click on **Forward Lookup Zones** and select **New Zone...**
3. In the New Zone Wizard:
   - Click **Next** on the welcome screen
   - Select **Primary zone** and click **Next**
   - Select **To all DNS servers running on domain controllers in this domain** and click **Next**
   - For Zone name, enter `komebacklab.local` and click **Next**
   - Select **Allow both nonsecure and secure dynamic updates** and click **Next**
   - Click **Finish** to create the zone

### 3. Creating the Reverse Lookup Zone

1. In the DNS Manager console, right-click on **Reverse Lookup Zones** and select **New Zone...**
2. In the New Zone Wizard:
   - Click **Next** on the welcome screen
   - Select **Primary zone** and click **Next**
   - Select **To all DNS servers running on domain controllers in this domain** and click **Next**
   - Select **IPv4 Reverse Lookup Zone** and click **Next**
   - For Network ID, enter `192.168.40` and click **Next**
   - Select **Allow both nonsecure and secure dynamic updates** and click **Next**
   - Click **Finish** to create the zone

### 4. Creating Required DNS Records

#### A Records for Infrastructure

1. In the DNS Manager console, expand **Forward Lookup Zones** and click on `komebacklab.local`
2. Right-click in the right pane and select **New Host (A or AAAA)...**
3. Create the following A records (repeat steps for each):

   | Name | IP Address |
   |------|------------|
   | ocp-haproxy | 192.168.40.2 |
   | ocp-bootstrap.lab | 192.168.40.200 |
   | ocp-cp-1.lab | 192.168.40.201 |
   | ocp-cp-2.lab | 192.168.40.202 |
   | ocp-cp-3.lab | 192.168.40.203 |
   | ocp-w-1.lab | 192.168.40.211 |
   | ocp-w-2.lab | 192.168.40.212 |
   | api.lab | 192.168.40.2 |
   | api-int.lab | 192.168.40.2 |
   | etcd-0.lab | 192.168.40.201 |
   | etcd-1.lab | 192.168.40.202 |
   | etcd-2.lab | 192.168.40.203 |

#### Creating the Wildcard DNS Record for Applications

1. Right-click on `komebacklab.local` and select **New Alias (CNAME)...**
2. For Alias name, enter `*.apps.lab`
3. For Fully qualified domain name (FQDN), enter `ocp-haproxy.komebacklab.local.` (note the trailing dot)
4. Click **OK**

#### Creating SRV Records for etcd

1. Right-click on `komebacklab.local` and select **Other New Records...**
2. Select **Service Location (SRV)** and click **Create Record...**
3. Create the following SRV records (repeat for each):

   | Service | Protocol | Priority | Weight | Port | Host |
   |---------|----------|----------|--------|------|------|
   | _etcd-server-ssl | _tcp | 0 | 10 | 2380 | etcd-0.lab.komebacklab.local. |
   | _etcd-server-ssl | _tcp | 0 | 10 | 2380 | etcd-1.lab.komebacklab.local. |
   | _etcd-server-ssl | _tcp | 0 | 10 | 2380 | etcd-2.lab.komebacklab.local. |

   Critical note: For each SRV record:
   - Set **Domain** to `lab.komebacklab.local`
   - Ensure the **Host offering this service** ends with a period (.)

### 5. Creating PTR Records (Reverse Lookup)

1. In the DNS Manager console, expand **Reverse Lookup Zones** and click on `40.168.192.in-addr.arpa`
2. Right-click in the right pane and select **New Pointer (PTR)...**
3. Create the following PTR records (repeat steps for each):

   | Host IP | Host FQDN |
   |---------|-----------|
   | 192.168.40.2 | ocp-haproxy.komebacklab.local. |
   | 192.168.40.2 | api.lab.komebacklab.local. |
   | 192.168.40.2 | api-int.lab.komebacklab.local. |
   | 192.168.40.200 | ocp-bootstrap.lab.komebacklab.local. |
   | 192.168.40.201 | ocp-cp-1.lab.komebacklab.local. |
   | 192.168.40.202 | ocp-cp-2.lab.komebacklab.local. |
   | 192.168.40.203 | ocp-cp-3.lab.komebacklab.local. |
   | 192.168.40.211 | ocp-w-1.lab.komebacklab.local. |
   | 192.168.40.212 | ocp-w-2.lab.komebacklab.local. |

   Note: For the Host IP, you'll only enter the last octet (2, 200, 201, etc.)

### 6. Verify DNS Configuration

1. From a command prompt on your Windows Server, run:
   ```
   nslookup api.lab.komebacklab.local
   nslookup ocp-cp-1.lab.komebacklab.local
   ```

2. Test reverse DNS with:
   ```
   nslookup 192.168.40.201
   ```

3. Test SRV records with:
   ```
   nslookup -type=srv _etcd-server-ssl._tcp.lab.komebacklab.local
   ```

## Configure Windows Server 2019 DHCP

### 1. Installing DHCP Role (if not already installed)

1. Open **Server Manager** and click on **Add roles and features**
2. Click **Next** until you reach the **Server Roles** page
3. Check the box for **DHCP Server** and click **Add Features** when prompted
4. Click **Next** until you reach the **Confirmation** page
5. Click **Install** and wait for the installation to complete
6. Click **Close** when finished
7. In Server Manager, click the flag icon in the top menu and select **Complete DHCP configuration**
8. Click **Next** in the DHCP Post-Install configuration wizard
9. Click **Commit** and then **Close**

### 2. Creating a DHCP Scope

1. Open **Server Manager** and click on **Tools** in the top-right menu bar, then select **DHCP**
2. In the DHCP console, expand the server node and right-click on **IPv4**, then select **New Scope...**
3. In the New Scope Wizard:
   - Click **Next** on the welcome screen
   - Enter a name (e.g., "OpenShift Network") and description, then click **Next**
   - For IP Address Range:
     - Start IP address: `192.168.40.80`
     - End IP address: `192.168.40.99`
     - Subnet mask: `255.255.255.0`
   - Click **Next**
   - For Exclusions, leave blank and click **Next**
   - For Lease Duration, set to 1 day (or preferred time) and click **Next**
   - Select **Yes, I want to configure these options now** and click **Next**
   - For Router (Default Gateway), enter `192.168.40.1` and click **Add**, then **Next**
   - For Domain Name and DNS Servers:
     - Parent domain: `komebacklab.local`
     - DNS Server: Enter your Windows Server IP address (192.168.40.1)
     - Click **Next**
   - For WINS Servers, click **Next**
   - Select **Yes, I want to activate this scope now** and click **Next**
   - Click **Finish** to create and activate the scope

### 3. Creating DHCP Reservations for OpenShift Nodes

1. In the DHCP console, expand your server > IPv4 > Your Scope
2. Right-click on **Reservations** and select **New Reservation...**
3. Create the following reservations (repeat steps for each node):

   | Name | IP Address | MAC Address | Description |
   |------|------------|-------------|-------------|
   | ocp-haproxy | 192.168.40.2 | [haproxy MAC] | HAProxy Load Balancer |
   | ocp-bootstrap | 192.168.40.200 | [bootstrap MAC] | Bootstrap Node |
   | ocp-cp-1 | 192.168.40.201 | [cp-1 MAC] | Control Plane 1 |
   | ocp-cp-2 | 192.168.40.202 | [cp-2 MAC] | Control Plane 2 |
   | ocp-cp-3 | 192.168.40.203 | [cp-3 MAC] | Control Plane 3 |
   | ocp-w-1 | 192.168.40.211 | [w-1 MAC] | Worker Node 1 |
   | ocp-w-2 | 192.168.40.212 | [w-2 MAC] | Worker Node 2 |

   Notes:
   - You'll need to replace [MAC] with the actual MAC addresses from your VMs
   - Ensure "DHCP client identifier" is not checked
   - Set "Supported types" to "Both"

### 4. Verify DHCP Configuration

1. From a client machine, test DHCP with:
   ```
   ipconfig /release
   ipconfig /renew
   ```

2. Verify the client received the correct IP address, DNS server, and default gateway

## Configure Ubuntu HAProxy

### 1. Install Ubuntu Server

1. Boot the ocp-haproxy VM with the Ubuntu 20.04 ISO
2. Follow the installation prompts:
   - Choose language and keyboard layout
   - Configure the network interface to use DHCP (it will get the reserved IP from Windows DHCP)
   - Set the hostname to `ocp-haproxy`
   - Create a user account (e.g., `haproxy-admin`)
   - Choose to install OpenSSH server when prompted
   - Complete the installation and reboot

### 2. Update Ubuntu and Install HAProxy

1. SSH to the ocp-haproxy VM:
   ```bash
   ssh haproxy-admin@192.168.40.2
   ```

2. Update the system:
   ```bash
   sudo apt update
   sudo apt upgrade -y
   ```

3. Install HAProxy:
   ```bash
   sudo apt install haproxy -y
   ```

### 3. Configure HAProxy

1. Backup the original configuration:
   ```bash
   sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
   ```

2. Create a new HAProxy configuration:
   ```bash
   sudo nano /etc/haproxy/haproxy.cfg
   ```

3. Replace the content with the following configuration:

```
# Global settings
#---------------------------------------------------------------------
global
    maxconn     20000
    log         /dev/log local0 info
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    log                     global
    mode                    http
    option                  httplog
    option                  dontlognull
    option http-server-close
    option redispatch
    option forwardfor       except 127.0.0.0/8
    retries                 3
    maxconn                 20000
    timeout http-request    10000ms
    timeout http-keep-alive 10000ms
    timeout check           10000ms
    timeout connect         40000ms
    timeout client          300000ms
    timeout server          300000ms
    timeout queue           50000ms

# Enable HAProxy stats
listen stats
    bind :9000
    stats uri /stats
    stats refresh 10000ms

# Kube API Server
frontend k8s_api_frontend
    bind :6443
    default_backend k8s_api_backend
    mode tcp

backend k8s_api_backend
    mode tcp
    balance source
    server      ocp-bootstrap 192.168.40.200:6443 check
    server      ocp-cp-1 192.168.40.201:6443 check
    server      ocp-cp-2 192.168.40.202:6443 check
    server      ocp-cp-3 192.168.40.203:6443 check

# OCP Machine Config Server
frontend ocp_machine_config_server_frontend
    mode tcp
    bind :22623
    default_backend ocp_machine_config_server_backend

backend ocp_machine_config_server_backend
    mode tcp
    balance source
    server      ocp-bootstrap 192.168.40.200:22623 check
    server      ocp-cp-1 192.168.40.201:22623 check
    server      ocp-cp-2 192.168.40.202:22623 check
    server      ocp-cp-3 192.168.40.203:22623 check

# OCP Ingress - layer 4 tcp mode for each. Ingress Controller will handle layer 7.
frontend ocp_http_ingress_frontend
    bind :80
    default_backend ocp_http_ingress_backend
    mode tcp

backend ocp_http_ingress_backend
    balance source
    mode tcp
    server      ocp-w-1 192.168.40.211:80 check
    server      ocp-w-2 192.168.40.212:80 check

frontend ocp_https_ingress_frontend
    bind *:443
    default_backend ocp_https_ingress_backend
    mode tcp

backend ocp_https_ingress_backend
    mode tcp
    balance source
    server      ocp-w-1 192.168.40.211:443 check
    server      ocp-w-2 192.168.40.212:443 check
```

4. Save the file (Ctrl+O, then Enter) and exit (Ctrl+X)

### 4. Configure Firewall

1. Install UFW (Uncomplicated Firewall) if not already installed:
   ```bash
   sudo apt install ufw -y
   ```

2. Configure firewall rules:
   ```bash
   sudo ufw allow 22/tcp        # SSH
   sudo ufw allow 6443/tcp      # Kubernetes API
   sudo ufw allow 22623/tcp     # Machine Config Server
   sudo ufw allow 80/tcp        # HTTP
   sudo ufw allow 443/tcp       # HTTPS
   sudo ufw allow 9000/tcp      # HAProxy Stats
   ```

3. Enable the firewall:
   ```bash
   sudo ufw enable
   ```

4. Verify the firewall status:
   ```bash
   sudo ufw status
   ```

### 5. Enable and Start HAProxy

1. Enable HAProxy to start on boot:
   ```bash
   sudo systemctl enable haproxy
   ```

2. Start HAProxy:
   ```bash
   sudo systemctl start haproxy
   ```

3. Verify HAProxy is running:
   ```bash
   sudo systemctl status haproxy
   ```

### 6. Test HAProxy Configuration

1. Access the HAProxy stats page from a web browser:
   ```
   http://192.168.40.2:9000/stats
   ```

2. You should see the HAProxy statistics page with the configured backends (they will show as red until the OpenShift nodes are up)

## Generate and host install files

### 1. Set up a Web Server on the HAProxy VM

1. Install Apache:
   ```bash
   sudo apt install apache2 -y
   ```

2. Change the default port to 8080:
   ```bash
   sudo sed -i 's/Listen 80/Listen 8080/' /etc/apache2/ports.conf
   sudo sed -i 's/*:80/*:8080/' /etc/apache2/sites-available/000-default.conf
   ```

3. Configure firewall for web server:
   ```bash
   sudo ufw allow 8080/tcp
   ```

4. Restart Apache:
   ```bash
   sudo systemctl restart apache2
   ```

### 2. Prepare for OpenShift Installation

1. Install required packages:
   ```bash
   sudo apt install git wget -y
   ```

2. Generate an SSH key pair:
   ```bash
   ssh-keygen -t rsa -b 4096 -N '' -f ~/.ssh/id_rsa
   ```

3. Create an install directory:
   ```bash
   mkdir ~/ocp-install
   ```

4. Download the OpenShift installer and client tools:
   ```bash
   # Replace with the actual URLs from the Red Hat OpenShift Cluster Manager
   wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-install-linux.tar.gz
   wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux.tar.gz
   ```

5. Extract the tools:
   ```bash
   tar xvf openshift-install-linux.tar.gz
   tar xvf openshift-client-linux.tar.gz
   sudo mv oc kubectl /usr/local/bin/
   ```

6. Create the install-config.yaml file:
   ```bash
   nano ~/ocp-install/install-config.yaml
   ```

7. Add the following content (replace with your pull secret and SSH key):

```yaml
apiVersion: v1
baseDomain: komebacklab.local
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: lab
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
pullSecret: '{"auths":{"fake":{"auth":"aWQ6cGFzcwo="}}}'  # Replace with your pull secret
sshKey: 'ssh-rsa AAAA...'  # Replace with your SSH public key
```

8. Generate Kubernetes manifest files:
   ```bash
   ./openshift-install create manifests --dir ~/ocp-install
   ```

9. Generate the Ignition config files:
   ```bash
   ./openshift-install create ignition-configs --dir ~/ocp-install/
   ```

10. Create a hosting directory for the web server:
    ```bash
    sudo mkdir -p /var/www/html/ocp4
    ```

11. Copy the generated files to the web server:
    ```bash
    sudo cp -R ~/ocp-install/* /var/www/html/ocp4/
    ```

12. Download the RHCOS image:
    ```bash
    # Replace with the actual URL for your version
    wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.14/4.14.0/rhcos-4.14.0-x86_64-metal.x86_64.raw.gz -O /tmp/rhcos
    sudo mv /tmp/rhcos /var/www/html/ocp4/
    ```

13. Set permissions:
    ```bash
    sudo chown -R www-data:www-data /var/www/html/ocp4/
    sudo chmod -R 755 /var/www/html/ocp4/
    ```

14. Verify the files are accessible:
    ```bash
    curl http://localhost:8080/ocp4/
    ```

## Deploy OpenShift

1. Power on the ocp-bootstrap host and ocp-cp-\# hosts and select 'Tab' to enter boot configuration. Enter the following configuration:

   ```bash
   # Bootstrap Node - ocp-bootstrap
   coreos.inst.install_dev=sda coreos.inst.image_url=http://192.168.40.2:8080/ocp4/rhcos coreos.inst.insecure=yes coreos.inst.ignition_url=http://192.168.40.2:8080/ocp4/bootstrap.ign
   
   # Or if you waited for it boot, use the following command then just reboot after it finishes and make sure you remove the attached .iso
   sudo coreos-installer install /dev/sda -u http://192.168.40.2:8080/ocp4/rhcos -I http://192.168.40.2:8080/ocp4/bootstrap.ign --insecure --insecure-ignition
   ```

   ```bash
   # Each of the Control Plane Nodes - ocp-cp-\#
   coreos.inst.install_dev=sda coreos.inst.image_url=http://192.168.40.2:8080/ocp4/rhcos coreos.inst.insecure=yes coreos.inst.ignition_url=http://192.168.40.2:8080/ocp4/master.ign
   
   # Or if you waited for it boot, use the following command then just reboot after it finishes and make sure you remove the attached .iso
   sudo coreos-installer install /dev/sda -u http://192.168.40.2:8080/ocp4/rhcos -I http://192.168.40.2:8080/ocp4/master.ign --insecure --insecure-ignition
   ```

2. Power on the ocp-w-\# hosts and select 'Tab' to enter boot configuration. Enter the following configuration:

   ```bash
   # Each of the Worker Nodes - ocp-w-\#
   coreos.inst.install_dev=sda coreos.inst.image_url=http://192.168.40.2:8080/ocp4/rhcos coreos.inst.insecure=yes coreos.inst.ignition_url=http://192.168.40.2:8080/ocp4/worker.ign
   
   # Or if you waited for it boot, use the following command then just reboot after it finishes and make sure you remove the attached .iso
   sudo coreos-installer install /dev/sda -u http://192.168.40.2:8080/ocp4/rhcos -I http://192.168.40.2:8080/ocp4/worker.ign --insecure --insecure-ignition
   ```

## Monitor the Bootstrap Process

1. You can monitor the bootstrap process from the ocp-haproxy host:

   ```bash
   ./openshift-install --dir ~/ocp-install wait-for bootstrap-complete --log-level=debug
   ```

2. Once bootstrapping is complete the ocp-boostrap node [can be removed](#remove-the-bootstrap-node)

## Remove the Bootstrap Node

1. Remove all references to the `ocp-bootstrap` host from the `/etc/haproxy/haproxy.cfg` file:

   ```bash
   sudo nano /etc/haproxy/haproxy.cfg
   ```

2. Remove the following lines:
   - `server ocp-bootstrap 192.168.40.200:6443 check` from the `k8s_api_backend` section
   - `server ocp-bootstrap 192.168.40.200:22623 check` from the `ocp_machine_config_server_backend` section

3. Save the file and restart HAProxy:
   ```bash
   sudo systemctl restart haproxy
   ```

4. The ocp-bootstrap host can now be safely shutdown and deleted from the VMware ESXi Console, the host is no longer required

## Wait for installation to complete

> IMPORTANT: if you set mastersSchedulable to false the [worker nodes will need to be joined to the cluster](#join-worker-nodes) to complete the installation. This is because the OpenShift Router will need to be scheduled on the worker nodes and it is a dependency for cluster operators such as ingress, console and authentication.

1. Collect the OpenShift Console address and kubeadmin credentials from the output of the install-complete event:

   ```bash
   ./openshift-install --dir ~/ocp-install wait-for install-complete
   ```

2. Continue to join the worker nodes to the cluster in a new tab whilst waiting for the above command to complete

## Join Worker Nodes

1. Setup 'oc' and 'kubectl' clients on the ocp-haproxy machine:

   ```bash
   export KUBECONFIG=~/ocp-install/auth/kubeconfig
   # Test auth by viewing cluster nodes
   oc get nodes
   ```

2. View and approve pending CSRs:

   > Note: Once you approve the first set of CSRs additional 'kubelet-serving' CSRs will be created. These must be approved too.
   > If you do not see pending requests wait until you do.

   ```bash
   # View CSRs
   oc get csr
   # Approve all pending CSRs
   oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve
   # Wait for kubelet-serving CSRs and approve them too with the same command
   oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve
   ```

3. Watch and wait for the Worker Nodes to join the cluster and enter a 'Ready' status

   > This can take 5-10 minutes

   ```bash
   watch -n5 oc get nodes
   ```

## Configure storage for the Image Registry

> A Bare Metal cluster does not by default provide storage so the Image Registry Operator bootstraps itself as 'Removed' so the installer can complete. As the installation has now completed storage can be added for the Registry and the operator updated to a 'Managed' state.

### 1. Set up NFS Server on the HAProxy VM

1. Install NFS Server:
   ```bash
   sudo apt install nfs-kernel-server -y
   ```

2. Create the share directory:
   ```bash
   sudo mkdir -p /shares/registry
   sudo chown -R nobody:nogroup /shares/registry
   sudo chmod -R 777 /shares/registry
   ```

3. Configure the NFS export:
   ```bash
   echo "/shares/registry  192.168.40.0/24(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports
   ```

4. Apply the configuration:
   ```bash
   sudo exportfs -ra
   ```

5. Configure firewall for NFS:
   ```bash
   sudo ufw allow 2049/tcp  # NFS
   sudo ufw allow 111/tcp   # RPC
   sudo ufw allow 111/udp   # RPC
   ```

### 2. Configure the Image Registry to use the NFS storage

1. Create a PV for the registry:
   ```bash
   cat <<EOF > ~/registry-pv.yaml
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: registry-pv
   spec:
     capacity:
       storage: 100Gi
     accessModes:
       - ReadWriteMany
     persistentVolumeReclaimPolicy: Retain
     nfs:
       path: /shares/registry
       server: 192.168.40.2
   EOF
   ```

2. Apply the PV:
   ```bash
   oc create -f ~/registry-pv.yaml
   ```

3. Configure the Image Registry operator to use persistent storage:
   ```bash
   oc edit configs.imageregistry.operator.openshift.io
   ```

4. Update the configuration with:
   ```yaml
   managementState: Managed
   storage:
     pvc:
       claim: # leave the claim blank
   ```

5. Verify the registry pod is running:
   ```bash
   oc get pods -n openshift-image-registry
   ```

## Create the first Admin user

1. Install the htpasswd utility:
   ```bash
   sudo apt install apache2-utils -y
   ```

2. Create an htpasswd file:
   ```bash
   htpasswd -c -B -b ~/htpasswd admin password
   ```

3. Create a secret with the htpasswd file:
   ```bash
   oc create secret generic htpass-secret --from-file=htpasswd=~/htpasswd -n openshift-config
   ```

4. Apply the OAuth configuration:
   ```bash
   cat <<EOF > ~/oauth-htpasswd.yaml
   apiVersion: config.openshift.io/v1
   kind: OAuth
   metadata:
     name: cluster
   spec:
     identityProviders:
     - name: htpasswd_provider
       mappingMethod: claim
       type: HTPasswd
       htpasswd:
         fileData:
           name: htpass-secret
   EOF
   ```

5. Apply the configuration:
   ```bash
   oc apply -f ~/oauth-htpasswd.yaml
   ```

6. Assign the cluster-admin role to the admin user:
   ```bash
   oc adm policy add-cluster-role-to-user cluster-admin admin
   ```

## Access the OpenShift Console

1. Wait for the 'console' Cluster Operator to become available:
   ```bash
   oc get co
   ```

2. Append the following to your local workstations `/etc/hosts` file:

   > From your local workstation

   ```
   # Open the hosts file
   # Windows: notepad C:\Windows\System32\drivers\etc\hosts
   # macOS/Linux: sudo nano /etc/hosts

   # Append the following entries:
   192.168.40.2 api.lab.komebacklab.local console-openshift-console.apps.lab.komebacklab.local oauth-openshift.apps.lab.komebacklab.local downloads-openshift-console.apps.lab.komebacklab.local alertmanager-main-openshift-monitoring.apps.lab.komebacklab.local grafana-openshift-monitoring.apps.lab.komebacklab.local prometheus-k8s-openshift-monitoring.apps.lab.komebacklab.local thanos-querier-openshift-monitoring.apps.lab.komebacklab.local
   ```

3. Navigate to the OpenShift Console URL in your browser:
   ```
   https://console-openshift-console.apps.lab.komebacklab.local
   ```

4. Log in as the 'admin' user with the password you set

   > You will get self-signed certificate warnings that you can ignore
   > If you need to login as kubeadmin and need the password again, you can retrieve it with: `cat ~/ocp-install/auth/kubeadmin-password`

## Managing Multiple Kubernetes Clusters

If you're running both an OpenShift (OKD) cluster and a kubeadm-based Kubernetes cluster, you'll need a strategy to manage them efficiently from the same jumpbox VM. Here are the recommended approaches:

### Option 1: Using Separate kubeconfig Files (Recommended)

This approach uses the KUBECONFIG environment variable to switch between clusters:

1. Keep separate kubeconfig files for each cluster:
   ```bash
   # OKD cluster config
   ~/ocp-install/auth/kubeconfig
   
   # kubeadm cluster config (example location)
   ~/.kube/kubeadm-config
   ```

2. To work with a specific cluster, set the KUBECONFIG environment variable:
   ```bash
   # For OKD cluster
   export KUBECONFIG=~/ocp-install/auth/kubeconfig
   
   # For kubeadm cluster
   export KUBECONFIG=~/.kube/kubeadm-config
   ```

3. Create shell aliases for quick switching:
   ```bash
   # Add these to your ~/.bashrc
   alias okd='export KUBECONFIG=~/ocp-install/auth/kubeconfig'
   alias k8s='export KUBECONFIG=~/.kube/kubeadm-config'
   
   # Then source your bashrc
   source ~/.bashrc
   ```

4. Now you can simply type `okd` or `k8s` to switch contexts, and then use kubectl commands normally.

### Option 2: Merged kubeconfig with Contexts

You can merge multiple kubeconfig files and use contexts to switch between clusters:

1. Merge the kubeconfig files:
   ```bash
   # First, backup your existing config
   cp ~/.kube/config ~/.kube/config.bak
   
   # Merge OKD config (this will create a new context)
   KUBECONFIG=~/.kube/config:~/ocp-install/auth/kubeconfig kubectl config view --flatten > /tmp/merged-config
   mv /tmp/merged-config ~/.kube/config
   ```

2. List available contexts:
   ```bash
   kubectl config get-contexts
   ```

3. Switch between contexts:
   ```bash
   # Switch to OKD cluster
   kubectl config use-context <okd-context-name>
   
   # Switch to kubeadm cluster
   kubectl config use-context <kubeadm-context-name>
   ```

4. Create aliases for quick switching:
   ```bash
   # Add these to your ~/.bashrc
   alias okdctx='kubectl config use-context <okd-context-name>'
   alias k8sctx='kubectl config use-context <kubeadm-context-name>'
   ```

### Option 3: Using Different CLI Tools

You can also use different CLI tools for each cluster:

1. Use `oc` exclusively for OpenShift operations:
   ```bash
   # Set this once
   export KUBECONFIG=~/ocp-install/auth/kubeconfig
   
   # Then use oc for all OpenShift operations
   oc get nodes
   oc get pods
   ```

2. Use `kubectl` exclusively for the kubeadm cluster:
   ```bash
   # Set this once
   export KUBECONFIG=~/.kube/kubeadm-config
   
   # Then use kubectl for all kubeadm operations
   kubectl get nodes
   kubectl get pods
   ```

This approach works because `oc` is a superset of `kubectl` with additional OpenShift-specific commands, while still supporting all standard Kubernetes commands.

## Troubleshooting

### DNS Issues

1. Verify DNS resolution from the HAProxy VM:
   ```bash
   dig api.lab.komebacklab.local
   dig console-openshift-console.apps.lab.komebacklab.local
   ```

2. Check reverse DNS:
   ```bash
   dig -x 192.168.40.201
   ```

3. Verify SRV records:
   ```bash
   dig _etcd-server-ssl._tcp.lab.komebacklab.local SRV
   ```

### DHCP Issues

1. Check DHCP leases on Windows Server:
   - Open DHCP Manager
   - Expand your server > IPv4 > Your Scope > Address Leases
   - Verify all nodes have received the correct IP addresses

2. Check network configuration on a node (if you can access it):
   ```bash
   ip addr
   ip route
   cat /etc/resolv.conf
   ```

### HAProxy Issues

1. Check HAProxy status:
   ```bash
   sudo systemctl status haproxy
   ```

2. View HAProxy logs:
   ```bash
   sudo tail -f /var/log/haproxy.log
   ```

3. Check HAProxy stats page:
   ```
   http://192.168.40.2:9000/stats
   ```

### OpenShift Installation Issues

1. Check bootstrap progress:
   ```bash
   ./openshift-install --dir ~/ocp-install wait-for bootstrap-complete --log-level=debug
   ```

2. Collect logs from all cluster hosts:
   ```bash
   ./openshift-install gather bootstrap --dir ~/ocp-install --bootstrap=192.168.40.200 --master=192.168.40.201 --master=192.168.40.202 --master=192.168.40.203
   ```

3. Check cluster operator status:
   ```bash
   oc get co
   ```

4. Check pod status:
   ```bash
   oc get pods --all-namespaces | grep -v Running
   ```
