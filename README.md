# 🛡️ Azure VPS Proxy Setup with 3proxy

This guide will help you deploy a proxy server (HTTPS and SOCKS5) on a newly created Ubuntu-based VPS on Azure using [3proxy]. The setup includes authentication and automated installation via Azure CLI in the Azure Cloud Shell.

## 📌 Prerequisites

* An active Azure subscription.
* Access to [Azure Cloud Shell](https://portal.azure.com/) (choose **Bash**).
* Basic knowledge of Linux shell commands.

## 🚀 Step-by-Step Instructions

### 1. Launch Azure Cloud Shell

Go to [https://portal.azure.com](https://portal.azure.com) and open the **Bash** terminal.

### 2. Run the Script Below

Copy and paste the following script into your Azure Cloud Shell. Be sure to modify the configuration variables first.

```bash
#!/bin/bash

# === CONFIG ===
region="canadacentral"                        
admin_user="your_username"
admin_pass="Your_password@123"
image="Canonical:UbuntuServer:18.04-LTS:latest"
vm_size="Standard_B1s"

# === Proxy Auth Config ===
proxy_user="your_proxy_username"
proxy_pass="your_proxy_password"

# === AUTO-GENERATED NAMES ===
number=$RANDOM
vmgroup="mygroup${number}0"
vmname="myname${number}0"
public_ip="myip${number}0"

# === CREATE RESOURCE GROUP ===
az group create --name $vmgroup --location $region

# === CREATE VM ===
az vm create \
    --resource-group $vmgroup \
    --name $vmname \
    --public-ip-address $public_ip \
    --image $image \
    --size $vm_size \
    --admin-username $admin_user \
    --admin-password $admin_pass \
    --authentication-type all \
    --generate-ssh-keys

# === OPEN ALL PORTS ===
az vm open-port --ids $(az vm list -g $vmgroup --query "[].id" -o tsv) --port '*'

# === INSTALL 3PROXY WITH AUTH ===
az vm run-command invoke \
  -g $vmgroup \
  -n $vmname \
  --command-id RunShellScript \
  --scripts "
    apt-get update && apt-get -y upgrade && \
    apt-get install -y build-essential nano wget && \
    wget --no-check-certificate https://github.com/3proxy/3proxy/archive/refs/tags/0.8.13.tar.gz && \
    tar xzf 0.8.13.tar.gz && cd 3proxy-0.8.13 && make -f Makefile.Linux && \
    cd src && mkdir -p /etc/3proxy && mv 3proxy /etc/3proxy/ && \
    cd /etc/3proxy && \
    wget --no-check-certificate https://raw.githubusercontent.com/guidetuanhp/proxy_azure/main/3proxy.cfg && \
    echo '${proxy_user}:CL:${proxy_pass}' > /etc/3proxy/.proxyauth && chmod 600 /etc/3proxy/.proxyauth && \
    cd /etc/init.d && \
    wget --no-check-certificate https://raw.githubusercontent.com/guidetuanhp/proxy_azure/main/3proxyinit && \
    chmod +x 3proxyinit && update-rc.d 3proxyinit defaults && \
    /etc/init.d/3proxyinit start
  "

# === OUTPUT PUBLIC IP ===
public_ip=$(az vm show -d -g $vmgroup -n $vmname --query publicIps -o tsv)

# === PRINT RESULT ===
echo "✅ Proxy VPS has been successfully created!"
echo "🌐 Public IP: $public_ip"
echo "🧑 Username: $admin_user"
echo "🔐 Password: $admin_pass"
echo ""
echo "➡️ HTTPS Proxy: ${public_ip}:2616:${proxy_user}:${proxy_pass}"
echo "➡️ SOCKS5 Proxy: ${public_ip}:2618:${proxy_user}:${proxy_pass}"
```

## 🧾 Output Format

After the script finishes running, you will see the following proxy details printed in the terminal:

```
✅ Proxy VPS has been successfully created!
🌐 Public IP: <YOUR_PUBLIC_IP>
🧑 Username: your_username
🔐 Password: Your_password@123

➡️ HTTPS Proxy: <YOUR_PUBLIC_IP>:2616:your_proxy_username:your_proxy_password
➡️ SOCKS5 Proxy: <YOUR_PUBLIC_IP>:2618:your_proxy_username:your_proxy_password
```

## 📣 Notes

* Ports `2616` (HTTPS) and `2618` (SOCKS5) are pre-configured in `3proxy.cfg`.
* You can edit the config or credentials by SSH-ing into the VM using the printed login credentials.

