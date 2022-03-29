# FortiGate HA - Egress Inspection /w custom-route using network-tag in GCP
This document describes how to inspect outgoing traffic to Internet through FortiGate with multi-zone HA cluster setup using network-tag feature set per Virtual Machine (VM) in Google Cloud Platform (GCP). Use-case is to help customers who wants to shift outgoing traffic to FortiGate deployment in a VM-by-VM fashion. With this option, selected VMs' egress traffic can be inspected by FortiGate.

Official document for network-tags can be found at [here](https://cloud.google.com/vpc/docs/add-remove-network-tags)

In this scenario, FortiGate active/passive (A/P) cluster is operating as Internet breakout firewall in GCP project. "Custom route" is a GCP networking component that we can use to manipulate default routing behavior for inter-VPC type of traffic. In normal conditions, each subnet within a VPC uses default route automatically assigned by platform pointing GCP's Internet services. Egress traffic to Internet can be routed through following next-hop options:

-	Instance (_available in GUI_)
-	IP address (_available in GUI_)
-	VPN tunnel (_available in GUI_)
-	Forwarding rule of internal TCP/UDP load balancer (_available in GUI_)
-	Internal TCP/UDP load balancer IP address (_**not-available** in GUI_)

Last option above cannot be configured using GCP GUI as of today (Q2'22). Configuration will be done using gcloud CLI commands which will be described below.

## Pre-requisites
-	Fortigate multi-zone HA A/P deployed in project using [deployment templates](https://github.com/40net-cloud/fortinet-gcp-solutions/tree/master/FortiGate/architectures/200-ha-active-passive-lb-sandwich),
-	Spoke VPCs and VMs deployed in each VPC,
-	VPC peering established between Spoke VPCs and FortiGate Internal VPC ([configuring VPC peering](https://cloud.google.com/vpc/docs/vpc-peering))
-	FortiGate NAT-enabled security rule allowing specific egress services for Spoke-VMs
-	FortiGate static route for Spoke-VPC CIDRs
-	*(Optional)* FortiGate Fabric-connector to import GCP objects ([FortiGate GCP Fabric-connector how to](https://docs.fortinet.com/document/fortigate-public-cloud/6.2.0/gcp-administration-guide/906632/security-fabric-connector-integration-with-gcp))

## Design & Topology
The following diagram illustrates environment for this use-case. As shown in the topology, an Internal Load Balancer (ILB) is placed behind FortiGate HA cluster. ILB's internal fronting IP address will be used as a next-hop-IP-address setting in custom-route configuration pointing out to Internet. 
<img src=https://github.com/ozanoguz/gcp-customroute_by_nwtag/blob/main/images/SS_Custom_Route.jpg>
## Configuration Steps

### Step 1. Access Google Cloud Shell
GCP management console page can be used to access CLI by clicking "Activage Cloud Shell" button on top right.

<img src=https://github.com/ozanoguz/gcp-customroute_by_nwtag/blob/main/images/SS_GCP_activate_cloud_shell.png width="200"/>

After your Cloud Shell machine provisioned, a black screen will be visible at the bottom.

<img src=https://github.com/ozanoguz/gcp-customroute_by_nwtag/blob/main/images/SS_GCP_cloud_shell.png width="400"/>

### Step 2. Create custom-route with network-tag
After accessing cloud shell over CLI, following syntax can be used to create custom route table with network tag specified.  Same _network_tag_value_ will be used by compute resources in Spoke VPC.  

```
gcloud compute routes create custom_route_name \
   --network=name_of_spoke_vpc \
   --destination-range=0.0.0.0/0 \
   --next-hop-ilb=ip_address_of_ilb \
   --priority=custom_route_priority \
   --tags=network_tag_value
```
### Step 3. Edit Virtual Machine Network Tag
Navigate to specific VM which needs egress inspection by FortiGate under _"Compute Engine > VM Instances > select specific VM"_ 

<img src=https://github.com/ozanoguz/gcp-customroute_by_nwtag/blob/main/images/SS_GCP_navigate_vm.png width="300"/>

You can add network tag value on "_Edit_" screen for a virtual machine. Find "_Network Tags_" section on edit screen as below.

<img src=https://github.com/ozanoguz/gcp-customroute_by_nwtag/blob/main/images/SS_GCP_edit_vm_network_tag.png width="400"/> 

Add _network_tag_value_ defined in custom route table. When you click "_Save_" at the bottom, custom route table will become effective and outgoing traffic will be routed to ILB IP address as it is defined in custom route.

### Step 4. FortiGate Security Rule

A security rule allows outgoing traffic for specific source objects. In the example below, _Spoke1_VPC_ object is used.

Here is the nagivation path for creating those:
Objects:       Under "Policy & Objects > Addresses"
Security Rule: Under "Policy & Objects > Firewall Policy"

<img src=https://github.com/ozanoguz/gcp-customroute_by_nwtag/blob/main/images/SS_FortiGate_security_rule.png width="800"/> 

### Step 5. Validation

In this demo environment, Linux based VM is installed within Spoke VPC. Serial console for this VM can be accessed through GCP console GUI via this path "Compute Engine > VM Instances > Spoke-VM > Connect to Serial Console"

<img src=https://github.com/ozanoguz/gcp-customroute_by_nwtag/blob/main/images/SS_GCP_access_vm_via_console.png width="400"/>

To find out egress-NAT'ed public IP address, _curl ip.me_ can be used. Output will show Public IP address which is used by Cloud NAT deployed via FortiGate deployment template. That means, outgoing packets are traversing through FortiGate instances.

<img src=https://github.com/ozanoguz/gcp-customroute_by_nwtag/blob/main/images/SS_GCP_pip_cloud_nat.png width="300"/>


### Step 6. FortiGate Traffic Log

Traffic log for the test above can be accessed through FortiGate management GUI screen via "Log & Export > Forward Traffic"

<img src=https://github.com/ozanoguz/gcp-customroute_by_nwtag/blob/main/images/SS_FortiGate_traffic_log.png width="800"/>

## Additional Resources

[Fortinet GCP Documentation](https://docs.fortinet.com/cloud-solutions/gc)

[Fortinet GCP Github Repo](https://github.com/40net-cloud/fortinet-gcp-solutions)

[FortiGate GCP Admin Guide](https://docs.fortinet.com/document/fortigate-public-cloud/7.0.0/gcp-administration-guide/736375/about-fortigate-vm-for-gcp)
