# Site to Site VPN between Azure and GC (static routing)

Lab to build Azure Hub Spoke with S2S VPN to GCP using static routing

## Intro

The goal of this lab is to create a S2S VPN between Azure and GCP.
On Azure side we have a traditional Hub/Spoke while on GCP we have a single VM in VPC.
You can also use GCP as emulated on-premises environment connecting to Azure.

## Components

## Lab

### Prerequisites

You can get Azure CLI and GCP CLI in the same shell as well OS. That will make your life easier provisioning this solution.

- Azure CLI installation instructions: [How to install the Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
- GCP CLI instruction follow: [What is gcloud and How to install on Windows, macOS and Linux?](https://geekflare.com/gcloud-installation-guide/)
- Make sure you have proper subscription and permission before you proceed to the steps below.

Alternatively you can open respective CLI on GCP Portal or Azure Portal and run the steps below side-by-side.

### Steps

The steps listed below can be executed on Azure CLI or Google CLI, each step describes the respective Cloud provider where the command will be executed.

1) (Azure) - Define variables. Make changes based on your requirements

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

2) (Azure) - Deploy Azure Hub and Spokes. This process takes about 30 minutes to complete.

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

3) (GCP) - Define variables

```bash
# Define GCP variables
project=(Required) #Set your project Name. Get your PROJECT_ID use command: gcloud projects list 
region=us-central1 #Set your region. Get Regions/Zones Use command: gcloud compute zones list
zone=$region-c # Set availability zone: a, b or c.
vpcrange=192.168.0.0/24
envname=vpnlab
vmname=vm1
mypip=$(curl -4 ifconfig.io -s) #Gets your Home Public IP or replace with that information. It will add it to the Firewall Rule.
```

4) (GCP) Deploy VPC, VPN Gateway and Linux VM.

```bash
#Create VPC
sudo gcloud compute networks create $envname-vpc --project=$project --subnet-mode=custom --mtu=1460 --bgp-routing-mode=regional
gcloud compute networks subnets create $envname-subnet --project=$project --range=$vpcrange --network=$envname-vpc --region=$region

#Create Firewall Rule
sudo gcloud compute firewall-rules create $envname-allow-traffic-from-azure --network $envname-vpc --allow tcp,udp,icmp --source-ranges 192.168.0.0/16,10.0.0.0/8,172.16.0.0/16,35.235.240.0/20,$mypip/32

#Create Ubuntu VM:
sudo gcloud compute instances create $envname-vm1 --project=$project --zone=$zone --machine-type=f1-micro --network-interface=subnet=$envname-subnet,network-tier=PREMIUM --image=ubuntu-1804-bionic-v20210817 --image-project=ubuntu-os-cloud --boot-disk-size=10GB --boot-disk-type=pd-balanced --boot-disk-device-name=$envname-vm1 

#GCP VPN
sudo gcloud compute target-vpn-gateways create onpremvpn --project=$project --region=$region --network=$envname-vpc 
sudo gcloud compute addresses create onpremvpn-pip --project=$project --region=$region
sudo gcpvpnpip=$(gcloud compute addresses describe onpremvpn-pip --region=$region --project=$project --format='value(address)')
sudo gcloud compute forwarding-rules create onpremvpn-rule-esp --project=$project --region=$region --address=$gcpvpnpip --ip-protocol=ESP --target-vpn-gateway=onpremvpn 
sudo gcloud compute forwarding-rules create onpremvpn-rule-udp500 --project=$project --region=$region --address=$gcpvpnpip --ip-protocol=UDP --ports=500 --target-vpn-gateway=onpremvpn 
sudo gcloud compute forwarding-rules create onpremvpn-rule-udp4500 --project=$project --region=$region --address=$gcpvpnpip --ip-protocol=UDP --ports=4500 --target-vpn-gateway=onpremvpn
```

5) Setup VPN between on Azure and GCP:

- Azure

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

- GCP

```bash
#GCP VPN Tunnel to Azure
azgwnamepip=$(az network public-ip show -g $rg -n Az-Hub-vpngw-pip1 --query ipAddress -o tsv)
sudo gcloud compute vpn-tunnels create vpn-to-azure --project=$project --region=$region --peer-address=$azgwnamepip --shared-secret=$sharedkey --ike-version=2 --local-traffic-selector=0.0.0.0/0 --remote-traffic-selector=0.0.0.0/0 --target-vpn-gateway=onpremvpn 
sudo gcloud compute routes create vpn-to-azure-route-1 --project=$project --network=$envname-vpc --priority=1000 --destination-range=10.0.0.0/8 --next-hop-vpn-tunnel=vpn-to-azure --next-hop-vpn-tunnel-region=$region
```

6) Check status of VPN connection

- Azure

```bash
# Check VPN Status on Azure side
az network vpn-connection show -g $rg --n Azure-to-OnpremGCP --query connectionStatus -o tsv
az network vpn-connection list-ike-sas -g $rg --n Azure-to-OnpremGCP
```

- GCP 

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
ping 10.0.10.4 #Azure Hub VM
ping 10.0.11.4 #Azure Spoke 1 VM
ping 10.0.12.4 #Azure Spoke 2 VM
```
