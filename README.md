# Site-to-Site VPN between Azure and GCP (static routing)

## Introduction

On this LAB, an S2S VPN will get created between Azure and GCP using CLI commands on both Cloud Providers. On the Azure side, we have a traditional Hub/Spoke, while on GCP we have a single VM in VPC. The main goals of this lab is to:

1) Build a IPSec site-to-site (S2S) VPN between Azure and GCP.
2) Use GCP as an emulated On-premises environment to reach an Azure Hub/Spoke environment.

:point_right: Note: this LAB uses IPSec S2S VPN tunnel using static routing. Another similar lab, using dynamic routing (BGP) with active-active IPSec tunnel, will be available soon.

## Architecture diagram

![Environment](./media/azure-vpn-s2s-gcp.png)

## Solution components

The components that you can deployed are exactly what is shown above on the Architecture Diagram:

1. **Azure Hub VNET** (10.0.10.0/24) and subnets (subnet1, RouteServerSubnet, GatewaySubnet)
2. **Azure Spoke1** (10.0.11.0/24) and subnet1
3. **Azure Spoke2** (10.0.12.0/24) and subnet1
4. Emulated **On-premises** on GCP VPC 192.168.0.0/24.
5. Azure VPN Gateway: Single tunnel using static routing 192.168.0.0/24 (GCP).
6. GCP VPN Gateway: Single tunnel using static routing to 10.0.0.0/8 (Azure).
7. Azure VMs provisioned: **Az-Hub-lxvm** (10.0.10.4), **Az-Spk1-lxvm** (10.0.11.4), **Az-Spk2-lxvm** (10.0.12.4).
8. GCP VM: *vpnlab-vm1** (192.168.0.4).

## Lab steps

### Prerequisites

You can get Azure CLI and GCP CLI in the same OS (Linux or Windows). That will make your life easier provisioning this solution because both az and gcloud commands can consume the variables making easy to build the lab solution. The steps below have been validated on Ubuntu 18.4.

- Azure CLI installation instructions: [How to install the Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
    - Tip: to install Azure CLI on Linux just run the command:

    ```bash
    curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
    ```

- GCP CLI instruction follow: GCP CLI: [Installing Cloud SDK](https://cloud.google.com/sdk/docs/install#deb)
    - Other useful reference: [What is gcloud and How to install on Windows, macOS and Linux?](https://geekflare.com/gcloud-installation-guide/)
- Make sure you have proper subscription and permission before you proceed to the steps below.

Alternatively you can open respective CLI on GCP Portal or Azure Portal and run the steps below side-by-side. However, keep in mind some of the commands may fail because of the variable parsed between both Azure and GCP CLI commands.

:point_right: All the commands listed in this section are also included in CLI file in this repo: [deploy.azcli]()

### Steps

The steps listed below can be executed on Azure CLI or Google CLI, each step describes the respective Cloud provider where the command will be executed.

1) **(Azure)** - Define variables. Make changes based on your requirements

```bash
#Azure Variables
rg=LAB-VPN-GCP #Define your resource group
location=centralus #Set Region
mypip=$(curl -4 ifconfig.io -s) #Captures your local Public IP and adds it to NSG to restrict access to SSH only for your Public IP.
sharedkey=$(openssl rand -base64 24) #VPN Gateways S2S shared key is automatically generated. This works on Linux only.

#Define parameters for Azure Hub and Spokes (Optional):
AzurehubName=Az-Hub #Azure Hub Name
AzurehubaddressSpacePrefix=10.0.10.0/24 #Azure Hub VNET address space
AzurehubNamesubnetName=subnet1 #Azure Hub Subnet name where VM will be provisioned
Azurehubsubnet1Prefix=10.0.10.0/27 #Azure Hub Subnet address prefix
AzurehubgatewaySubnetPrefix=10.0.10.32/27 #Azure Hub Gateway Subnet address prefix
AzurehubrssubnetPrefix=10.0.10.64/27 #Azure Hub Route Server subnet address prefix
AzureFirewallPrefix=10.0.10.128/26 #Azure Firewall Prefix
Azurespoke1Name=Az-Spk1 #Azure Spoke 1 name
Azurespoke1AddressSpacePrefix=10.0.11.0/24 # Azure Spoke 1 VNET address space
Azurespoke1Subnet1Prefix=10.0.11.0/27 # Azure Spoke 1 Subnet1 address prefix
Azurespoke2Name=Az-Spk2 #Azure Spoke 1 name
Azurespoke2AddressSpacePrefix=10.0.12.0/24 # Azure Spoke 1 VNET address space
Azurespoke2Subnet1Prefix=10.0.12.0/27 # Azure Spoke 1 VNET address space

#Parsing parameters above in Json format (do not change)
JsonAzure={\"hubName\":\"$AzurehubName\",\"addressSpacePrefix\":\"$AzurehubaddressSpacePrefix\",\"subnetName\":\"$AzurehubNamesubnetName\",\"subnet1Prefix\":\"$Azurehubsubnet1Prefix\",\"AzureFirewallPrefix\":\"$AzureFirewallPrefix\",\"gatewaySubnetPrefix\":\"$AzurehubgatewaySubnetPrefix\",\"rssubnetPrefix\":\"$AzurehubrssubnetPrefix\",\"spoke1Name\":\"$Azurespoke1Name\",\"spoke1AddressSpacePrefix\":\"$Azurespoke1AddressSpacePrefix\",\"spoke1Subnet1Prefix\":\"$Azurespoke1Subnet1Prefix\",\"spoke2Name\":\"$Azurespoke2Name\",\"spoke2AddressSpacePrefix\":\"$Azurespoke2AddressSpacePrefix\",\"spoke2Subnet1Prefix\":\"$Azurespoke2Subnet1Prefix\"}
JsonOnPrem={\"name\":\"$OnPremName\",\"addressSpacePrefix\":\"$OnPremVnetAddressSpace\",\"subnet1Prefix\":\"$OnPremSubnet1prefix\",\"gatewaySubnetPrefix\":\"$OnPremgatewaySubnetPrefix\",\"asn\":\"$OnPremgatewayASN\"}
```

2) **(Azure)** - Deploy Azure Hub and Spokes. This process takes about 30 minutes to complete.

Note: You will be prompted to add username (VmAdminUsername) and password (VmAdminPassword)

```bash
#Deploy base lab environment = Hub + VPN Gateway + VM and two Spokes with one VM on each.
echo "***  Note you will be prompted by username and password ***"
echo "*** It will take around 30 minutes to finish the deployment ***"
az group create --name $rg --location $location
az deployment group create --name VPNERCoexist-$RANDOM --resource-group $rg \
--template-uri https://raw.githubusercontent.com/dmauser/azure-hub-spoke-base-lab/main/azuredeploy.json \
--parameters deployHubVPNGateway=true gatewaySku=VpnGw1 vpnGatewayGeneration=Generation1 Restrict_SSH_VM_AccessByPublicIP=$mypip sharedKey=$sharedkey deployHubERGateway=false Onprem=$JsonOnPrem Azure=$JsonAzure \
--output none
```

3) **(GCP)** - Define variables

```bash
# Define GCP variables (Mandatory: Define your GCP project as variable)
project=<add here> #Set your project Name (REQUIRED). Get your PROJECT_ID use command: sudo gcloud projects list 
region=us-central1 (OPTIONAL) #Set your region. Get Regions/Zones Use command: gcloud compute zones list
zone=$region-c # Set availability zone: a, b or c.
vpcrange=192.168.0.0/24
envname=vpnlab
vmname=vm1
mypip=$(curl -4 ifconfig.io -s) #Gets your Home Public IP or replace with that information. It will add it to the Firewall Rule.
```

4) **(GCP)** Deploy VPC, VPN Gateway and Linux VM.

```bash
#Create VPC
sudo gcloud compute networks create $envname-vpc --project=$project --subnet-mode=custom --mtu=1460 --bgp-routing-mode=regional
gcloud compute networks subnets create $envname-subnet --project=$project --range=$vpcrange --network=$envname-vpc --region=$region

#Create Firewall Rule
sudo gcloud compute firewall-rules create $envname-allow-traffic-from-azure --network $envname-vpc --allow tcp,udp,icmp --source-ranges 192.168.0.0/16,10.0.0.0/8,172.16.0.0/16,35.235.240.0/20,$mypip/32

#Create Ubuntu VM:
sudo gcloud compute instances create $envname-vm1 --project=$project --zone=$zone --machine-type=f1-micro --network-interface=subnet=$envname-subnet,network-tier=PREMIUM --image=ubuntu-1804-bionic-v20220118 --image-project=ubuntu-os-cloud --boot-disk-size=10GB --boot-disk-type=pd-balanced --boot-disk-device-name=$envname-vm1 

#GCP VPN
sudo gcloud compute target-vpn-gateways create onpremvpn --project=$project --region=$region --network=$envname-vpc 
sudo gcloud compute addresses create onpremvpn-pip --project=$project --region=$region
gcpvpnpip=$(sudo gcloud compute addresses describe onpremvpn-pip --region=$region --project=$project --format='value(address)')
sudo gcloud compute forwarding-rules create onpremvpn-rule-esp --project=$project --region=$region --address=$gcpvpnpip --ip-protocol=ESP --target-vpn-gateway=onpremvpn 
sudo gcloud compute forwarding-rules create onpremvpn-rule-udp500 --project=$project --region=$region --address=$gcpvpnpip --ip-protocol=UDP --ports=500 --target-vpn-gateway=onpremvpn 
sudo gcloud compute forwarding-rules create onpremvpn-rule-udp4500 --project=$project --region=$region --address=$gcpvpnpip --ip-protocol=UDP --ports=4500 --target-vpn-gateway=onpremvpn
```

5) Setup VPN between on Azure and GCP:

- **Azure**

```bash
#Azure Local Network Gateway
az network local-gateway create --gateway-ip-address $gcpvpnpip \
--name lng-onprem-gcp \
--resource-group $rg \
--local-address-prefixes 192.168.0.0/24

# Azure VPN tunnel to GCP
gwname=Az-Hub-vpngw
az network vpn-connection create --name Azure-to-OnpremGCP \
--resource-group $rg \
--vnet-gateway1 $gwname \
--location $(az group show -n $rg --query location -o tsv) \
--shared-key $sharedkey \
--local-gateway2 lng-onprem-gcp
```

- **GCP**

```bash
#GCP VPN Tunnel to Azure
azgwnamepip=$(az network public-ip show -g $rg -n Az-Hub-vpngw-pip1 --query ipAddress -o tsv)
sudo gcloud compute vpn-tunnels create vpn-to-azure --project=$project --region=$region --peer-address=$azgwnamepip --shared-secret=$sharedkey --ike-version=2 --local-traffic-selector=0.0.0.0/0 --remote-traffic-selector=0.0.0.0/0 --target-vpn-gateway=onpremvpn 
sudo gcloud compute routes create vpn-to-azure-route-1 --project=$project --network=$envname-vpc --priority=1000 --destination-range=10.0.0.0/8 --next-hop-vpn-tunnel=vpn-to-azure --next-hop-vpn-tunnel-region=$region
```

6) Check status of VPN connection

- **Azure**

```bash
# a) Check Connection Status
az network vpn-connection show -g $rg --n Azure-to-OnpremGCP --query connectionStatus -o tsv

# b) Check vpn connection IKE/SAs details
az network vpn-connection list-ike-sas -g $rg --n Azure-to-OnpremGCP
```

- **GCP**

```bash
#GCP VPN Status on GCP Side
#More info: https://cloud.google.com/network-connectivity/docs/vpn/how-to/checking-vpn-status
sudo gcloud compute vpn-tunnels describe vpn-to-azure \
   --region=$region \
   --project=$project \
   --format='flattened(status,detailedStatus)'
```

7) Test VPN connectivity - SSH GCP VM and issue pings against Azure VMs between them.

```bash
# Get Azure Hub and Spoke VMs IPs 
echo $AzurehubName-lxvm && az network nic show --resource-group $rg -n $AzurehubName-lxvm-nic --query "ipConfigurations[].privateIpAddress" -o tsv &&
echo $Azurespoke1Name-lxvm && az network nic show --resource-group $rg -n $Azurespoke1Name-lxvm-nic --query "ipConfigurations[].privateIpAddress" -o tsv &&
echo $Azurespoke2Name-lxvm && az network nic show --resource-group $rg -n $Azurespoke2Name-lxvm-nic --query "ipConfigurations[].privateIpAddress" -o tsv

# Log on GCP VM and try to reach all tree VMs listed above:
sudo gcloud compute ssh $envname-vm1 --zone=$zone

# Inside GCP VM ping Azure VM
echo Az-Hub-Lxvm && ping 10.0.10.4 -O -c 5
echo Az-Spk1-lxvm && ping 10.0.11.4 -O -c 5
echo Az-Spk2-lxvm && ping 10.0.12.4 -O -c 5
```

Optional - You can logon on any Azure VM and ping GCP VM IP (192.168.0.2).

## Clean up

- **(GCP)** remove lab environment.

    (Note: it requires the variables defined during the setup to be effective before running the commands below)

```bash
# GCP
sudo gcloud compute vpn-tunnels delete vpn-to-azure --region $region --quiet
sudo gcloud compute routes delete vpn-to-azure-route-1 --quiet
sudo gcloud compute forwarding-rules delete onpremvpn-rule-esp --region $region --quiet
sudo gcloud compute forwarding-rules delete onpremvpn-rule-udp500 --region $region --quiet
sudo gcloud compute forwarding-rules delete onpremvpn-rule-udp4500 --region $region --quiet
sudo gcloud compute target-vpn-gateways delete onpremvpn --region $region --quiet
sudo gcloud compute addresses delete onpremvpn-pip --region $region --quiet
sudo gcloud compute instances delete $envname-vm1 --project=$project --zone=$zone --quiet
sudo gcloud compute firewall-rules delete $envname-allow-traffic-from-azure --quiet
sudo gcloud compute networks subnets delete $envname-subnet --project=$project --region=$region --quiet
sudo gcloud compute networks delete $envname-vpc --project=$project --quiet
```

- **(Azure)** - Remove resource group via Portal or CLI:

```bash
az group delete -g $rg --no-wait --yes
```
