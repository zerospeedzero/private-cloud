# Enterprise private cloud
## Project overview
As a specialist in private cloud solutions with expertise in Kubernetes and OpenShift, I was asked by a banking client to demonstrate the capabilities of our private native cloud platform. The main goals were to make their mobile app platform more agile and to initiate their first pilot project using container technology. We were one of three vendors chosen for this tender. During the selection process, I designed and set up an integrated private cloud platform using enterprise-level Kubernetes solutions and some open-source tools that the client already uses. Ultimately, we were successful in winning this opportunity, and the client has implemented most of our products and suggestions in their private cloud environment. Due to privacy and security concerns, I'll be sharing the technology stacks and the proof-of-technology environment in another IT environment for reference and use a sample application for case showing instead of using customer mobile applications.
## Kubernetes (RedHat OpenShift)
As a prominent bank in Hong Kong, customer security and high-quality post-implementation support are of utmost importance. Our established best practice mandates that all IT solution technology components meet enterprise-level quality standards. Consequently, I recommend the adoption of RedHat OpenShift as the container platform.

[RedHat OpenShift](https://access.redhat.com/documentation/en-us/openshift_container_platform/4.12/html/installing/installing-on-bare-metal#installing-bare-metal) (OCP) expands vanilla Kubernetes into an application platform designed for enterprise use at scale. Starting with the release of [OpenShift 4](https://blog.openshift.com/introducing-red-hat-openshift-4/), the default operating system is [Red Hat CoreOS](https://docs.openshift.com/container-platform/4.1/architecture/architecture-rhcos.html), which provides an immutable infrastructure and automated updates.

## Virtual environment
In the context of setting up a technology environment, I deployed eight VMware servers to serve as the foundation for the Kubernetes infrastructure. These servers are divided into specific roles: one for DNS, NFS (persistent storage), and load balancing, another for the initial setup (bootstrap) of the OpenShift cluster (which can be removed later), and a pfsense virtual appliance to mimic the separation of private and DMZ networks.

For this setup, I utilized an ESXi 6.7 host with substantial resources: 256GB of RAM and 24 CPUs. Additionally, a separate VLAN (Virtual Local Area Network) was configured specifically for the OpenShift Container Platform (OCP). This configuration allows for effective management and control of the Kubernetes master node (OpenShift control plane) as well as the worker nodes (OpenShift compute nodes).

Here's a detailed list of the virtual machines we've set up:

| Machine              | Type                 | OS      | vCPU | RAM | Storage | OCP IP        | WAN IP       |
|----------------------|----------------------|---------|------|-----|---------|---------------|--------------|
| pfsense              | Router/DHCP/Firewall | FreeBSD | 4    | 4   | 8       | 192.168.1.210 | 172.29.1.209 |
| ocp4-service         | DNS/Web/NFS/LB       | RHEL8   | 4    | 4   | 100     | 192.168.1.1   | 172.29.1.225 |
| ocp4-bootstrap       | OCP Bootstrap        | RHCOS   | 4    | 16  | 100     | 192.168.1.200 |              |
| ocp4-control-plane-1 | OCP Master node      | RHCOS   | 4    | 16  | 100     | 192.168.1.201 |              |
| ocp4-control-plane-2 | OCP Master node      | RHCOS   | 4    | 16  | 100     | 192.168.1.202 |              |
| ocp4-control-plane-3 | OCP Master node      | RHCOS   | 4    | 16  | 100     | 192.168.1.203 |              |
| ocp4-compute-1       | OCP Worker node      | RHCOS   | 4    | 16  | 100     | 192.168.1.204 |              |
| ocp4-compute-2       | OCP Worker node      | RHCOS   | 4    | 16  | 100     | 192.168.1.205 |              |
| ocp4-compute-3       | OCP Worker node      | RHCOS   | 4    | 16  | 100     | 192.168.1.206 |              |

### Create a new network in VMWare for OCP:
1. Login to the VMWare Host. Select Networking → Port Groups → Add port group. Setup an OCP network on an unused VLAN, in my instance, VLAN 600.
![9.png](/ocp412/9.png)

2. Name the Group and set the VLAN ID (e.g. 600).
![3.png](/ocp412/3.png)

3. Create the ocp4-services VM:
[Download](https://access.redhat.com/downloads/content/479/ver=/rhel---8/8.7/x86_64/product-software) the RHEL 8.7 ISO DVD. Example: [rhel-8.7-x86_64-dvd.iso](https://access.redhat.com/downloads/content/479/ver=/rhel---8/8.7/x86_64/product-software) and upload it to your ESXi host datastore.

![7.png](/ocp412/7.png)

### Create a new Virtual Machine. Choose Guest OS as Linux and Select Red Hat Enterprise Linux 8 (64-bit)
1. Create a new Virtual server from vCenter. 
![8.png](/ocp412/8.png)

2. Select the Datastore and customize the settings to 4 vCPU, 4GB RAM, 100GB HD. Add a 2nd network adapter to the OCP network. Attached the CentOS ISO.
![10.png](/ocp412/10.png)

3. Review your settings and click Finish.

### Install RHEL8 on the ocp4-services VM:
1. Run through the RedHat 8 installation. Use the “Standard Partition” storage configuration. On the “Installation Destination” page, Leave "Automatic" under Storage Configuration, then Done.
![11.png](/ocp412/11.png)

2. For Software Selection, use Server with GUI and add the Guest Agents.
![12.png](/ocp412/12.png)

3. Enable the NIC connected to the VM Network and set the hostname as ocp4-services, then click Apply and Done.
![13.png](/ocp412/13.png)

4. Click “Begin Installation” to start the install.

5. Set the Root password, and create an admin user.

6. After the installation has completed, login, and update the OS.
```
subscription-manager register --username <username> --password <password> --auto-attach
yum update -y
init 6
```
7. Setup XRDP for Remote Access from Home Network
```
yum -y install tigervnc-server tigervnc
vncpasswd
cp /usr/lib/systemd/system/vncserver@.service /etc/systemd/system/vncserver@:1.service
firewall-cmd --zone=public --permanent --add-port=5901/tcp
firewall-cmd --reload
```
8. Download Google Chrome rpm from Google website and install along with git

```
sudo dnf install -y ~/Downloads/google-chrome-stable_current_x86_64.rpm git
```
### Create the pfsense VM:
1. [Download the pfSense ISO](https://www.pfsense.org/download/) and upload it to your ESXi host’s datastore.
![19.png](/ocp412/19.png)

2. Create a new Virtual Machine. Choose Guest OS as Other and Select FreeBSD 64-bit.
![20.png](/okd45cluster/20.png)

3. Use the default template settings for resources. Select your home network for Network Adapter 1, and add a new network adapter using the OCP network.
![21.png](/okd45cluster/21.png)

### Setup the pfsense VM:
1. Power on your pfSense VM and run through the installation using all the default values. After completion your VM console should look like this
![22.png](/okd45cluster/22.png)

2. Login to pfSense via your web-browser on the ocp4-services VM. The default username is “admin” and the password is “xxxxxxx”.
![23.png](/okd45cluster/23.png)

3. After logging in, click next and use “ocp4-pfsense” for hostname and “ocp.local” for the domain and add 192.168.1.210 as the Primary DNS server.
![24.png](/okd45cluster/24.png)

4. Select your Timezone. Next.
Use Defaults for WAN Configuration. Uncheck “Block RFC1918 Private Networks” since your home network is the “WAN” in this setup. Next.
![25.png](/okd45cluster/25.png)

5. Use the default LAN IP and subnet mask. Set an admin password on the next screen.
![26.png](/okd45cluster/26.png)


### Create bootstrap, master, and worker nodes:
1. Download the [Fedora CoreOS Bare Metal ISO](https://getfedora.org/coreos/download/) and upload it to your ESXi datastore.
The latest stable version at the time of writing is 32.20200715.3.0
![27.png](/okd45cluster/27.png)

2. Create the six ODK nodes (bootstrap, master, worker) on your ESXi host using the values in the spreadsheet at the beginning of this post:
![28.png](/okd45cluster/28.png)
![29.png](/okd45cluster/29.png)

3. You should end up with the following VMs:
![30.png](/okd45cluster/30.png)

### Setup DHCP reservations:
1. Compile a list of the OCP nodes MAC addresses by viewing the hardware configuration of your VMs.
![31.png](/okd45cluster/31.png)

2. Login into pfSense. Go to Services → DHCP Server and change your ending range IP to 192.168.1.99, and set the primary DNS server as 192.168.1.210, then click Save.
![32.png](/okd45cluster/32.png)

3. On the DHCP Server, page click Add at the bottom.
![33.png](/okd45cluster/33.png)


### Configure ocp4-services VM to host various services:
1. The ocp4-services VM is used to provide DNS, NFS exports, web server, and load balancing.
3. Copy the MAC address on the VM Hardware configuration page for the NIC connected to the OCP network and set up a DHCP Reservation for this VM using the IP address 192.168.1.210.
![35.png](/okd45cluster/35.png)
3. Hit “Apply Changes” at the top of the DHCP page when completed.
![36.png](/okd45cluster/36.png)

4. Open a terminal on the ocp4-services VM and clone the ocp4_files repo that contains the DNS, HAProxy, and install-conf.yaml example files:
```
cd
git clone https://github.com/cragr/ocp4_files.git
cd ocp4_files
```
### Install bind (DNS)
1. Open a Linux terminal.
```
sudo dnf -y install bind bind-utils
```
2. Copy the named config files and zones:
```
mkdir -p /custom
cd /custom
git clone http://172.29.1.234:3000/ELM/ocp4_files.git
mv /etc/named.conf  /etc/named.conf.org
sudo cp /custom/ocp4_files/named.conf /etc/named.conf
sudo cp /custom/ocp4_files/named.conf.local /etc/named/
sudo mkdir /etc/named/zones
sudo cp /custom/ocp4_files/db* /etc/named/zones
```

3. Enable and start named:
```
sudo systemctl enable named
sudo systemctl start named
sudo systemctl status named
```
4. Create firewall rules:
```
sudo firewall-cmd --permanent --add-port=53/udp
sudo firewall-cmd --reload
```
5. Change the DNS on the ocp4-service NIC that is attached to the VM Network (not OCP) to 127.0.0.1.
![37.png](/okd45cluster/37.png)
6. Restart the network services on the ocp4-services VM:
```
sudo systemctl restart NetworkManager
```
7. Test DNS on the ocp4-services.
```
dig ocp.internal.elm.com.hk
dig –x 192.168.1.210
```
8. With DNS working correctly, you should see the following results:

![38.png](/okd45cluster/38.png)

![39.png](/okd45cluster/39.png)

### Install HAProxy:
1. Open a terminal.
```
sudo dnf install haproxy -y
```

2. Copy haproxy config from the git ocp4_files directory :
```
mv /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.org
sudo cp /custom/ocp4_files/haproxy.cfg /etc/haproxy/haproxy.cfg
```

3. Start, enable, and verify HA Proxy service:
```
sudo setsebool -P haproxy_connect_any 1
sudo systemctl enable haproxy
sudo systemctl start haproxy
sudo systemctl status haproxy
```

4. Add OCP firewall ports:
```
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=22623/tcp
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```
5. Install Apache/HTTPD
```
sudo dnf install -y httpd
```
6. Change httpd to listen port to 8080:
```
cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd.conf.org
sudo sed -i 's/Listen 80/Listen 8080/' /etc/httpd/conf/httpd.conf
```
7. Enable and Start httpd service/Allow port 8080 on the firewall:
```
sudo setsebool -P httpd_read_user_content 1
sudo systemctl enable httpd
sudo systemctl start httpd
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```
8. Test the webserver:
```
curl localhost:8080
```

### Download the openshift-installer and oc client:
1. SSH to the ocp4-services VM
To download the latest oc client and openshift-install binaries, you need to use an existing version of the oc client.

2. Download the 4.6 version of the oc client and openshift-install from the OCP 
Access the Infrastructure Provider page on the https://cloud.redhat.com/openshift/install site. If you have a Red Hat account, log in with your credentials. If you do not, create an account.
Navigate to the page for your installation type, download the installation program for your operating system, and place the file in the directory where you will store the installation configuration files.


3. scp to services
```
openshift-install-linux.tar.gz
openshift-client-linux.tar.gz
```
4. Extract the ocp version of the oc client and openshift-install:
```
tar -zxvf openshift-client-linux.tar.gz
tar -zxvf openshift-install-linux.tar.gz
```
5. Move the kubectl, oc, and openshift-install to /usr/local/bin and show the version:
```
sudo mv kubectl oc openshift-install /usr/local/bin/
oc version
openshift-install version
```

6. The latest and recent releases are available at https://origin-release.svc.ci.openshift.org


### Setup the openshift-installer:
1. In the install-config.yaml, you can either use a pull-secret from RedHat or the default of “{“auths”:{“fake”:{“auth”: “bar”}}}” as the pull-secret.

2. Generate an SSH key if you do not already have one.
```
(ssh-keygen -t rsa -b 4096 -N '' \
    -f ~/.ssh/id_rsa)
ssh-keygen -t ed25519 -N '' \
    -f ~/.ssh/id_ed25519
eval "$(ssh-agent -s)"
(ssh-add ~/.ssh/id_rsa)
ssh-add ~/.ssh/id_ed25519
vi ~/.ssh/id_ed25519.pub
```
3. Create an install directory and copy the install-config.yaml file:
```
cd
rm -Rf /custom/install_dir
mkdir /custom/install_dir
cp /custom/ocp4_files/install-config.yaml /custom/install_dir
```

4. Edit the install-config.yaml in the install_dir, insert your pull secret and ssh key, and backup the install-config.yaml as it will be deleted in the next step:
```
vim ./install_dir/install-config.yaml
cp ./install_dir/install-config.yaml ./install_dir/install-config.yaml.bak
```
5. Generate the Kubernetes manifests for the cluster, ignore the warning:
```
openshift-install create manifests --dir=install_dir/
```
```
rm -f install_dir/openshift/99_openshift-cluster-api_master-machines-*.yaml install_dir/openshift/99_openshift-cluster-api_worker-machineset-*.yaml
```
6. Modify the cluster-scheduler-02-config.yaml manifest file to prevent Pods from being scheduled on the control plane machines:

```
sed -i 's/mastersSchedulable: true/mastersSchedulable: False/' install_dir/manifests/cluster-scheduler-02-config.yml
```
7. Now you can create the ignition-configs:

```
openshift-install create ignition-configs --dir=install_dir/
```
8. Note: If you reuse the install_dir, make sure it is empty. Hidden files are created after generating the configs, and they should be removed before you use the same folder on a 2nd attempt.



```
vi install_dir/append-bootstrap.ign
{
  "ignition": {
    "config": {
      "append": [
        {
          "source": "http://192.168.1.210:8080/ocp4/bootstrap.ign", 
          "verification": {}
        }
      ]
    },
    "timeouts": {},
    "version": "3.2.0"
  },
  "networkd": {},
  "passwd": {},
  "storage": {},
  "systemd": {}
}
```
```
base64 -w0 install_dir/master.ign > install_dir/master.64
base64 -w0 install_dir/worker.ign > install_dir/worker.64
base64 -w0 install_dir/bootstrap.ign > install_dir/bootstrap.64
base64 -w0 install_dir/append-bootstrap.ign > install_dir/append-bootstrap.64
```

### Host ignition and CentOS files on the webserver:

1. Create ocp4 directory in /var/www/html:
```
rm -Rf /var/www/html/ocp4
sudo mkdir /var/www/html/ocp4
```
2. Copy the install_dir content to /var/www/html/ocp4 and set permissions:

```
sudo cp -R install_dir/* /var/www/html/ocp4/
sudo chown -R apache: /var/www/html/
sudo chmod -R 755 /var/www/html/
```
3. Test the webserver:
[Download the Fedora CoreOS](https://getfedora.org/coreos/download/) bare-metal bios image and sig files and shorten the file names:

```
cd /var/www/html/ocp4/
sudo chown -R apache: /var/www/html/
sudo chmod -R 755 /var/www/html/
```
```
curl localhost:8080/ocp4/metadata.json
jq -r .infraID /custom/install_dir/metadata.json
grep -i infraID /custom/install_dir/.openshift_install_state.json
```
4. After the template deploys, deploy a VM for a machine in the cluster.

5. Right-click the template’s name and click Clone → Clone to Virtual Machine.

6. On the Select a name and folder tab, specify a name for the VM. You might include the machine type in the name, such as compute-1.

7. On the Select a name and folder tab, select the name of the folder that you created for the cluster.

8. On the Select a compute resource tab, select the name of a host in your datacenter.

9. Optional: On the Select storage tab, customize the storage options.

10. On the Select clone options, select Customize this virtual machine’s hardware.

11. On the Customize hardware tab, click VM Options → Advanced.
From the Latency Sensitivity list, select High.

12. Click Edit Configuration, and on the Configuration Parameters window, click Add Configuration Params. Define the following parameter names and values:
```
guestinfo.ignition.config.data: Paste the contents of the base64-encoded compute Ignition config file for this machine type.

guestinfo.ignition.config.data.encoding: Specify base64.

disk.EnableUUID: Specify TRUE.
```
### Starting the bootstrap node:

1. Power on the bootstrap VM. 
```
coreos-installer install \
     --insecure-ignition --ignition-url=http://192.168.1.210:8080/ocp4/bootstrap.ign /dev/sda
```
### Starting the control plane nodes:
1. Power on the control-plane nodes 
```
coreos-installer install \
     --insecure-ignition --ignition-url=http://192.168.1.210:8080/ocp4/master.ign /dev/sda
```
## Starting the compute nodes:
1. Power on the compute nodes 
```
coreos-installer install \
     --insecure-ignition --ignition-url=http://192.168.1.210:8080/ocp4/worker.ign /dev/sda
```

### Monitor the bootstrap installation:

1. You can monitor the bootstrap process from the ocp4-services node:

```
openshift-install --dir=install_dir/ wait-for bootstrap-complete --log-level=info
```
![bootstrap.completed.png](/ocp4.7/bootstrap.completed.png)
2. Once the bootstrap process is complete, which can take upwards of 30 minutes, you can shutdown your bootstrap node. Now is a good time to edit the /etc/haproxy/haproxy.cfg, comment out the bootstrap node, and reload the haproxy service.
```
sudo sed '/ ocp4-bootstrap /s/^/#/' /etc/haproxy/haproxy.cfg
sudo systemctl reload haproxy
```

### Login to the cluster and approve CSRs:
1. Now that the masters are online, you should be able to login with the oc client. Use the following commands to log in and check the status of your cluster:
```
export KUBECONFIG=~/install_dir/auth/kubeconfig
oc whoami
oc get nodes
oc get csr
```
![csr.png](/ocp4.7/csr.png)

2. You should only see the master nodes and several CSR’s waiting for approval. Install the jq package to assist with approving multiple CSR’s at once time.
```
wget -O jq https://github.com/stedolan/jq/releases/download/jq-1.6/jq-linux64
chmod +x jq
sudo mv jq /usr/local/bin/
jq --version
```
![50.png](/okd45cluster/50.png)

3. Approve all the pending certs and check your nodes:
```
oc get csr -ojson | jq -r '.items[] | select(.status == {} ) | .metadata.name' | xargs oc adm certificate approve
```
![compute.up.png](/ocp4.7/compute.up.png)
4. Check the status of the cluster operators.
```
oc get clusteroperators
oc get co | grep -v "True.*False.*False"; oc get no
```
![4.png](/ocp412/4.png)
![5.png](/ocp412/5.png)
add
```
172.29.1.210 console-openshift-console.apps.lab.ocp.internal.elm.com.hk oauth-openshift.apps.lab.ocp.internal.elm.com.hk 
```
to the machine which access the OCP cluster

5. The console has just become available in my picture above. Get your kubeadmin password from the /custom/install_dir/auth folder of the ocp4-services server and login to the web console:
```
cat /custom/install_dir/auth/kubeadmin-password
```
![6.png](/ocp412/6.png)
6. If you don't have network access to 192.168.1.x network, you may configure your Firefox browser to using the proxy as below.
![14.png](/ocp412/14.png)
7. Open your web browser to https://console-openshift-console.apps.lab.ocp.internal.elm.com.hk and login as kubeadmin with the password t2MbY-C7DXp-cj467-Dx2xo
![loginscreen.png](/ocp4.7/loginscreen.png)
8. The cluster status may still say upgrading, and it continues to finish the installation.
![dashboard.png](/ocp4.7/dashboard.png)

### Persistent Storage:

We need to create some persistent storage for our registry before we can complete this project. Let’s configure our ocp4-services VM as an NFS server and use it for persistent storage.

1. Login to your ocp4-services VM and begin to set up an NFS server. The following commands install the necessary packages, enable services, and configure file and folder permissions.
```
sudo dnf install -y nfs-utils
sudo systemctl enable nfs-server rpcbind
sudo systemctl start nfs-server rpcbind
sudo mkdir -p /var/nfsshare/registry
sudo chmod -R 777 /var/nfsshare
sudo chown -R nobody:nobody /var/nfsshare
```

2. Create an NFS Export
Add this line in the new /etc/exports file “/var/nfsshare 192.168.1.0/24(rw,sync,no_root_squash,no_all_squash,no_wdelay)”
```
echo '/var/nfsshare 192.168.1.0/24(rw,sync,no_root_squash,no_all_squash,no_wdelay)' | sudo tee /etc/exports
```
![56.png](/okd45cluster/56.png)
3. Restart the nfs-server service and add firewall rules:
```
sudo setsebool -P nfs_export_all_rw 1
sudo systemctl restart nfs-server
sudo firewall-cmd --permanent --zone=public --add-service mountd
sudo firewall-cmd --permanent --zone=public --add-service rpc-bind
sudo firewall-cmd --permanent --zone=public --add-service nfs
sudo firewall-cmd --reload
```
### Registry configuration:

1. Create a persistent volume on the NFS share. Use the registry_py.yaml in ocp4_files folder from the git repo:
```
oc create -f /custom/ocp4_files/registry_pv.yaml
oc get pv
```
![57.png](/okd45cluster/57.png)

2. Edit the image-registry operator:
```
oc edit configs.imageregistry.operator.openshift.io
```
3. Change the managementState: from Removed to Managed. Under storage: add the pvc: and claim: blank to attach the PV and save your changes automatically:

```
managementState: Managed
storage:
    pvc:
      claim:
```
![58.png](/okd45cluster/58.png)

4. Check your persistent volume, and it should now be claimed:
```
oc get pv
```
![59.png](/okd45cluster/59.png)

5. Check the export size, and it should be zero. In the next section, we will push to the registry, and the file size should not be zero.
```
du -sh /var/nfsshare/registry
```
![60.png](/okd45cluster/60.png)

In the next section, we will create a WordPress project and push it to the registry. After the push, the NFS export should show 200+ MB.

## Deploy EJBCA on RedHat OpenShift
### Overview
In the context of validating technological solutions, it's crucial to demonstrate the efficiency and effectiveness of a particular approach. When implementing cloud-native solutions, a significant challenge arises in ensuring secure TLS connections between the microservices and the backend/front-end services. Typically, large companies maintain their own certificate authority infrastructure, and all IT solutions must utilize secure certificates issued by this authority.

In our Proof of Technology (POT) environment, I have chosen to simulate a customer's internal Certificate Authority (CA) server using EJBCA. Additionally, we've configured the certificate manager to handle the lifecycle management of these certificates. This setup allows us to align with the security requirements commonly found in enterprise settings.

It is assumed an OpenShift environment is up and running.
Reference links are as below.
https://hub.docker.com/r/keyfactor/ejbca-ce
http://ca.tridum.mn/ejbca/doc/Creating_an_Issuing_CA_Signed_by_a_Root_on_Same_Node.html

### Deployment of EJBCA and MySQL
1. Login to OpenShift console with administrator permission.
2. Create OpenShift project
```
oc new-project ejbca
```
3. Deploy MySQL database to store the EJBCA data
```
oc new-app \
    -e MYSQL_USER=ejbca \
    -e MYSQL_PASSWORD='P@ssw0rd' \
    -e MYSQL_ROOT_PASSWORD='P@ssw0rd' \
    -e MYSQL_DATABASE=ejbca \
    mysql:5.6
```
4. Use PVC for data storage
```
oc set volume deploy/mysql --add --name=mysql-volume-1 -t pvc --claim-size=5G --overwrite
```
5. Deploy EJBCA-CA and using MySQL database
```
oc new-app --image=keyfactor/ejbca-ce --name ejbca-ce \
    -e TLS_SETUP_ENABLED="true" \
    -e DATABASE_JDBC_URL='jdbc:mysql://mysql:3306/ejbca?characterEncoding=UTF-8' \
    -e DATABASE_USER=ejbca -e DATABASE_PASSWORD='xxxxxxx'
```
6. Create an Route to the EJBCA web gui
```
oc create route passthrough ejbca-ce --service=ejbca-ce --port=8443
```
### Configuration of EJBCA
1. oc logs -f \<ejbca pod\> and determine the superadmin token.
![1.png](/ejbca/1.png)
> Password changed to XXXXXXXXXXXXXXI4RX
2. Generate client certificate to authenticate EJBCA
![2.png](/ejbca/2.png)
3. Download the client private key and certificate and managementCA certificate and import into computer keychain. Trust all of them.
![3.png](/ejbca/3.png)
![6.png](/ejbca/6.png)
4. Refresh the browser and using client authentication.
![4.png](/ejbca/4.png)
5. EJBCA adminweb can be accessed and ManagementCA status is active
![5.png](/ejbca/5.png)
6. Click "Certification Authorities" and input "Root" and then click "Create..." button.
![7.png](/ejbca/7.png)
7. Input the subject DN as "CN=Root" and validity as 10y.
![8.png](/ejbca/8.png)
8. Create OpenShift certificate authority under Root.
![9.png](/ejbca/9.png)
9. Input the subject DN as "CN=OpenShiftCA", Signed by Root, Certificate Profile as SUBCA and validity as 1y.
![10.png](/ejbca/10.png)
10. Click "Crypoto Tokens" and then choose OpenShiftCA. Check the box next to "Allow export of private keys" and then save.
![11.png](/ejbca/11.png)
11. Ensure all the follow CAs are shown.
![12.png](/ejbca/12.png)
### For Chrome at Windows
1. Browse at Chrome
```
Privacy and security >> Privacy and security
or
chrome://settings/security >> Privacy and security
```
3. Download the client private key and certificate and managementCA certificate and import into computer keychain. Trust all of them.
4. Refresh the browser and using client authentication.
```
chrome://restart
```
## Deploy Cert Manager on RedHat OpenShift
### Overview
It is assumed an OpenShift environment and ejbca are up and running.
Before are the reference links.
https://cert-manager.io/v1.2-docs/installation/openshift/
https://docs.openshift.com/container-platform/4.12/networking/ingress-operator.html
https://github.com/cert-manager/openshift-routes


### Deployment of Cert Manager using Operator
1. Login to OpenShift Web GUI and then go to OperatorHub and search 
![1.png](/cert-manager/1.png)
2. Click Install button
![2.png](/cert-manager/2.png)
3. Choose default value and click "Install"
![3.png](/cert-manager/3.png)
4. Ensure the cert-manager is succeeded status
![4.png](/cert-manager/4.png)
5. Then login to EJBCA and select "OpenShiftCA"
![5.png](/cert-manager/5.png)
6. Export the private keys and certificate of OpenShiftCA
![6.png](/cert-manager/6.png)
7. Use openssl to retrieve the private key and certificates.
![7.png](/cert-manager/7.png)
8. Based on the standard output of above steps. Save the private key as OpenShiftCA.key, certificate as OpenShiftCA.pem and root certificate as RootCA.pem.
![8.png](/cert-manager/8.png)
9. Login to OpenShift console with administrator permission.
10. Create secret file.
```
Create OpenShiftCa-secret.yaml file with base64 content of the key and certificate.
e.g.
apiVersion: v1
kind: Secret
metadata:
  name: openshiftca
  namespace: openshift-cert-manager
data:
  tls.crt: <cat OpenShiftCA.pem |base64 -w0>
  tls.key: <cat OpenShiftCA.key |base64 -w0>
```
11. Create the secret
```
oc create -f ./OpenShiftCA_secret.yaml
```
12. Create clusterissuer with the above secret
![10.png](/cert-manager/10.png)

### Testing the cert manager for certificate management
1. Create an OpenShift project for testing
```
oc new-project testing
```
2. Create an Certificate and the following content
```
cat certificate.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: testcert
  namespace: testing
spec:
  # Secret names are always required.
  secretName: mycert
  duration: 2160h # 90d
  renewBefore: 360h # 15d
  subject:
    organizations:
    - ELM
  # The use of the common name field has been deprecated since 2000 and is
  # discouraged from being used.
  commonName: golang-https
  isCA: false
  privateKey:
    algorithm: RSA
    encoding: PKCS1
    size: 2048
  usages:
    - server auth
    - client auth
  # At least one of a DNS Name, URI, or IP address is required.
  dnsNames:
  - golang-https-testing.apps.lab.ocp.internal.elm.com.hk
  - golang-https
  uris:
  - spiffe://cluster.local/ns/sandbox/sa/example
  ipAddresses:
  - 192.168.1.210
  # Issuer references are always required.
  issuerRef:
    name: openshiftca
    # We can reference ClusterIssuers by changing the kind here.
    # The default value is Issuer (i.e. a locally namespaced Issuer)
    kind: ClusterIssuer
    # This is optional since cert-manager will default to this value however
    # if you are using an external issuer, change this to that issuer group.
    group: cert-manager.io
```
3. Create the certicate object under testing project
```
oc create -f ./certificate.yaml
```
![11.png](/cert-manager/11.png)

4. Create an sample deployment
```
oc create -f https://raw.githubusercontent.com/nerdingitout/oc-route/main/golang-https.yml
```
5. Using the secret generated by the cer-manager with the certificate object
```
oc set volume dc/golang-https --add -t secret -m /go/src/app/certs --name cert --secret-name mycert
```
6. Create an pass-through route to access the application with OCP ingress controller
```
oc create route passthrough golang-https --service golang-https
```
7. Get the route of the application
```
oc get route
```
8. Open an Firefox and input the url and you will show the below screen since the certificate is not trused by browser.
![15.png](/cert-manager/15.png)
9. Download the Root certificate from the GUI.
![24.png](/cert-manager/24.png)
10. The certificate will be download
![26.png](/cert-manager/26.png)
11. On the Firefox, import the CA certificate
![25.png](/cert-manager/25.png)
12. Trust the CA.
![27.png](/cert-manager/27.png)
13. Go to the application again and you will find the server certificate will be trusted due to the Root CA is trusted.

Note : With the certifiate is create, the private key and certificates will be maintained and renewed by the Cert Manager on the OpenShift.

## Ensure OpenShift ingress controller route using external CA
1. Create an secret called "custom-certs-default" using certicate file ocproute.yaml
```e.g.
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ocproute
  namespace: openshift-ingress
spec:
  # Secret names are always required.
  secretName: custom-certs-default
  duration: 2160h # 90d
  renewBefore: 360h # 15d
  subject:
    organizations:
    - ELM
  # The use of the common name field has been deprecated since 2000 and is
  # discouraged from being used.
  commonName: '*.apps.lab.ocp.internal.elm.com.hk'
  isCA: false
  privateKey:
    algorithm: RSA
    encoding: PKCS1
    size: 2048
  usages:
    - server auth
    - client auth
  # At least one of a DNS Name, URI, or IP address is required.
  dnsNames:
  - '*.apps.lab.ocp.internal.elm.com.hk'
  uris:
  - spiffe://cluster.local/ns/sandbox/sa/example
  ipAddresses:
  - 192.168.1.210
  # Issuer references are always required.
  issuerRef:
    name: openshiftca
    # We can reference ClusterIssuers by changing the kind here.
    # The default value is Issuer (i.e. a locally namespaced Issuer)
    kind: ClusterIssuer
    # This is optional since cert-manager will default to this value however
    # if you are using an external iss
```
2. Create the certificate object
```
oc create -f ./ocproute.yaml 
```
3. Patching the OpenShift ingress controller using another secret for https edge and re-encryption connection.
```
oc patch --type=merge --namespace openshift-ingress-operator ingresscontrollers/default --patch '{"spec":{"defaultCertificate":{"name":"custom-certs-default"}}}' 
```
4. Now the OpenShift console and other *.apps.lab.ocp.internal.elm.com.hk are under OpenShiftCA and maintained by Cert Manager.
![32.png](/cert-manager/32.png)


Note : Some of the screen captured are using OKD/okd and it is actually OCP/ocp.