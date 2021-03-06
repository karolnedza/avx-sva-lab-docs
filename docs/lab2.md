# Lab 2

## Create VPCs/VNets and deploy Aviatrix gateways?

In this lab, we will begin to build out the multi-cloud network.  We will create:
- VPCs 
- Aviatrix Transit and Spoke Gateways
- create transit peerings to peer the different clouds together 
- connect an on-prem Data Center to the cloud.

![Lab Overview](images/lab-before.png)
_Fig. Lab Overview_

## Lab 2.1 - Deploy the AWS transit VPC
### Description
As the instructor explained in the first lab, we already have a transit VPC/VNET in Azure and GCP. In this lab, we are going to make the preparations to deploy the aviatrix gateways in the AWS environment. In order to do so, we need a VPC. Instead of having to use the CSP’s console to do this, the Aviatrix Terraform Provider can do this for us.
### Validate
Go to **_Useful Tools -> Create a VPC_**. As you can see, there already are multiple VPC’s here. See if you can identify these VPC’s on the lab diagram.  

### Task

Let's deploy our first VPC using terraform. Go to ```/sva-terraform-master/sva-code/``` folder

Use your favourite text editor to Open file  ```providers.tf``` and configure Aviatrix Terraform provider

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/providers.png" width="800">


Save the file and run ```terraform init``` Did it work? 

Terraform configurations must declare which providers they require, so that Terraform can install and use them.

Open file  ```versions.tf``` and configure Aviatrix Terraform provider 

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/versions.png" width="800">

[Terraform Required Providers](https://registry.terraform.io/providers/AviatrixSystems/aviatrix/latest/docs)

Save the file and run ```terraform init``` It should work now.

Open file  ```main.tf``` and configure Transit VPC

For the VPC CIDR, replace “x” with your pod number. For example, if your pod number is 11. “x” should be replaced with 11, so the complete CIDR will become 10.11.40.0/23. Make sure Aviatrix Transit VPC is set to True. The aviatrix_vpc resource will automatically create all the required public and private subnets, AWS IGW and routing tables.  

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/transit_vpc.png" width="800">

[Terraform Aviatrix VPC Documentation](https://registry.terraform.io/providers/AviatrixSystems/aviatrix/latest/docs/resources/aviatrix_vpc)

Save the file and run ```terraform apply``` Enter ```yes```

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/tf_apply_vpc.png" width="800">

Terraform deployed VPC but what actually was deployed and how we can use it?  You can check if by running ```terraform console``` (```terraform.tfstate``` must be in the same folder)

Start ```terraform console``` and run:

- ```aviatrix_vpc.aws_vpc``` (use resource name you configured in main.tf)
- ```aviatrix_vpc.aws_vpc.public_subnets```  
- ```aviatrix_vpc.aws_vpc.public_subnets[0].cidr ```

###### All Public Subnets
<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/public_subnets.png" width="800">

###### First Public Subnet
<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/first_public_subnet.png" width="800">

###### First Public Subnet CIDR - we will need it later!
<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/first_public_subnet_cidr.png" width="800">


#### Bonus:

By default, ```aviatrix_vpc```  will use the assigned CIDR to add a private and public subnet to each availability zone in AWS. As of version 6.1, we can modify this behavior using  advanced options. You can add new VPC with customies CIDR and number of subnets.

[Terraform Aviatrix VPC Documentation](https://registry.terraform.io/providers/AviatrixSystems/aviatrix/latest/docs/resources/aviatrix_vpc#advanced-options)

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/vpc_adv.png" width="800">


### Expected Results
Our environment now looks like this:  
![Topology](images/topology2.png)  
_Fig. Topology_  

Check out the CoPilot Topology View.  Do you see the newly created Transit VPC?

## Lab 2.2 - Deploy the Aviatrix transit gateway
### Description
In this exercise we are going to launch the Aviatrix transit gateway in the newly created Transit VPC in AWS. 

### Task
Deploy AVX Transit Gateway in the VPC that we just created. You are going to use the same ```/sva-code/main.tf``` file to configure Aviatrix Transit GW.
Use ```aviatrix_vpc``` resource reference whenever it is possible.  If you don't know what values can be refferenced use terraform documentation or ```terraform console```

[Terraform Aviatrix VPC Refferences](https://registry.terraform.io/providers/AviatrixSystems/aviatrix/latest/docs/resources/aviatrix_vpc#attribute-reference)

Aviatri Transit Gateway requirments:
- Name:  aws-transit
- Subnet: first available public subnet
- Active-Mesh: enabled
- Size: t3.small

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/transit_gw.png" width="800">


Save the file and run ```terraform apply``` Enter ```yes```  Please read then ingore warning "Deprecated Attribute"

This will take a few minutes, have a coffee.  

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/transit_gw_apply.png" width="800">

  
We are not deploying an HA solution in this workshop, but by adding just two linese we can deploy a second gateway and cluster them in an HA pair.  

[Terraform Aviatrix Transit GW Doc](https://registry.terraform.io/providers/AviatrixSystems/aviatrix/latest/docs/resources/aviatrix_transit_gateway)

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/transit_gw_ha.png" width="800">


### Expected Results
Our environment now looks like this:  
![Toplogy](images/topology3.png)  
_Fig. Topology with AWS Transit_  

Check out CoPilot Topology.  Why is the AWS Transit cluster now visible?  
![Toplogy](images/copilot-trans-gw.png)  
_Fig. Copilot Topology_  

## Lab 2.3 - Deploy the spoke gateways
### Description
Now that we have our transit set up, we will deploy the Aviatrix spoke gateways in the *AWS spoke VPC’s* that were already prepared.   

### Task
This is typical brown-field scenario with existing VPC. To create the spoke gateways for the existing VPC’s we need to get VPC-id and Public Subnets CIDRs. 

Go to Useful Tools -> Create a VPC. As you can see, there are our Spoke VPCs  and we see their VPC-ids. Copy both, you will need them.

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/spoke_vpc_id.png" width="800">

To get Public Subnet CIDR, select VPC and click on ```DETAILS```  Copy first public Subnet CIDR for each VPC (aws-spoke1 and aws-spoke2)

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/spoke_subnets.png" width="800">


```Useful Tools -> Create a VPC``` shows only VPC created by Controller. There is one more VPC created from Cloud Console. 

Go to ```Useful Tools -> VPC Tracker``` and look for ```aws-shared``` to get VPC-Id and VPC Address Space

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/shared_vpc.png" width="800">

Cloud Ops team told you that you can use only second or third avaialable subnets /28 in VPC  ```aws-shared``` /24 address space. 
Deploy ```Aviatrix Spoke GW``` in one of them

Hint: 10.[podx].44.16/28 or 10.[podx].44.32/28 - we can silently agree that subnetting is CCNA folks ;)

We have all details to deploy three spokes in VPCs:

|          VPC  |       GW Name |          Size |  Region       |    VPC ID     | Public Subnet CIDR | Active-Mesh |
| ------------- | ------------- | ------------- | ------------- | ------------- | ------------------ | ----------- |
| aws-spoke-1   | aws-spoke-1   | t3.small      | eu-central-1  |   xxxxx       |    x.x.x.x/28      |  true    |
| aws-spoke-2   | aws-spoke-2   | t3.small      | eu-central-1  |   xxxxx       |    x.x.x.x/28      |  true    |
| aws-shared   | aws-shared   | t3.small      | eu-central-1  |   xxxxx       |    x.x.x.x/28      |   true     |


You are going to use the same ```/sva-code/main.tf``` to configure all Spoke Gateways. Do not delete previous configuration!

[Terraform Aviatrix Spoke GW Doc](https://registry.terraform.io/providers/AviatrixSystems/aviatrix/latest/docs/resources/aviatrix_spoke_gateway)

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/spoke_gw.png" width="800">

Save the ```/sva-code/main.tf``` file and run ```terraform apply``` Enter ```yes```

Three Spoke GWs should be deployed in a few minutes

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/spoke_gw_apply.png" width="800">


### Expected Results
Our environment now looks like this:
![Toplogy](images/topology4.png)  
_Fig. Topology with Spokes_  

Check out CoPilot Topology.    
![Toplogy](images/copilot-spoke-gw.png)  
_Fig. Copilot Topology_

## Lab 2.4 - Attach Spoke Gateways to Aviatrix Transit GW
### Description
We now have all of our gateways deployed. Next we need to set up connectivity between the spoke gateways and the transit gateway.

### Task
Attach three Spoke Gateways to Transit GW. You are going to use the same ```/sva-code/main.tf``` to configure spoke attachments. 
Do not delete previous configuration!

[Terraform Aviatrix Spoke-to-Transit gateway attachments](https://registry.terraform.io/providers/AviatrixSystems/aviatrix/latest/docs/resources/aviatrix_spoke_transit_attachment)

Do NOT enter GW names but use terraform resource reference. This will ensure that attachement will be created after Spoke and Transit Gateways are ready.

If you don't know what values can be refferenced use terraform documentation

[Terraform Aviatrix Spoke GW Refferences](https://registry.terraform.io/providers/AviatrixSystems/aviatrix/latest/docs/resources/aviatrix_spoke_gateway#gw_name)

[Terraform Aviatrix Transit GW Refferences](https://registry.terraform.io/providers/AviatrixSystems/aviatrix/latest/docs/resources/aviatrix_transit_gateway#gw_name)

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/spoke_attachments.png" width="800">


### Expected Results
If you now go to the **_Multi-Cloud Transit -> List_** page, you can see the 3 transit gateways and each of the spoke gateways attached to them in the Spoke List. Validate that the spokes you just attached are showing up here.  
  
Our environment now looks like this:
![Toplogy](images/topology5.png)  
_Fig. Topology with Attached Spokes_  

Check out CoPilot Topology.    
![Toplogy](images/copilot-spoke-attach.png)  
_Fig. Copilot Topology with Attached Spokes_

## Lab 2.5 - Test Connectivity Between AWS Spokes
### Description
The AWS Spokes should now be connected to the AWS Transit so now we can check connectivity between the spokes.
### Validate
To test the connectivity, we will SSH using the public FQDN of one of our Linux hosts, and run ping to the private FQDNs of the hosts in the other spokes.  
* Connect into AWS-SRV1
* Run the following commands:
```
ping aws-srv2-priv.pod[x].sva.aviatrixlab.de
ping shared-priv.pod[x].sva.aviatrixlab.de
```
> Were you able to ping aws-srv2-priv and shared-priv?

Go to **_Multi-Cloud Transit -> List_** and look at the gateway routing tables for _aws-spoke1_, _AWS-Spoke2_ and _AWS-Shared_.  

By default, routes are not propagated between spokes attached to the same transit gateway. We have to enable a setting on our transit gateway to allow this to happen.  

You need to enable Connected Transit. To do this modify Transit GW configuration by adding   ```connected_transit = true```

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/connected_transit.png" width="800">


Save the file and run ```terraform apply``` Enter ```yes```  Terraform should modify Transit GW attribute connected_transit

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/connected_transit_apply.png" width="800">


* Connect into AWS-SRV1
* Run the following commands:
```
ping aws-srv2-priv.pod[x].sva.aviatrixlab.de
ping shared-priv.pod[x].sva.aviatrixlab.de
```
* Trace the hops between Spoke1 and Spoke2 in AWS
```
mtr aws-srv2-priv.pod[x].sva.aviatrixlab.de
```
> Were you able to ping aws-srv2-priv and shared-priv?

### Expected Results
After enabling Connected Transit, the spoke to spoke communication tests should now be successful and spoke routes should be propagated.

## Lab 2.6 - Set Up the Multi-Cloud Transit Peering
### Description
Now that we have built the AWS environment, let’s connect it to the Azure and GCP environment.

### Task

Have a look at **_Multi-Cloud Transit -> Transit Peering_**. As you can see, the _Azure_ and _GCP_ gateways are already peered together. You have validated this connectivity to work already in exercise 1.3. 

We can create a new transit peering on this page, by clicking but let's use Terraform. Create two transit peering:
* AWS and the Azure gateway (azure-transit)
* AWS and the GCP gateway (gcp-transit)

You are going to use the same ```/sva-code/main.tf``` to configure transit peering. Do not delete previous configuration!

[Terraform Aviatrix Transit GW Peering](https://registry.terraform.io/providers/AviatrixSystems/aviatrix/latest/docs/resources/aviatrix_transit_gateway_peering)

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/transit_peering.png" width="800">

Save the file and run ```terraform apply``` Enter ```yes``` 


* Try to ping the server in Azure and GCP from one of the AWS instances
```
ping gcp-srv1-priv.pod[x].sva.aviatrixlab.de
ping azure-srv1-priv.pod[x].sva.aviatrixlab.de
```

### Expected Results
After adding the transit peerings and creating a full mesh, all resources should be reachable from each of the clouds.  

Our lab environment now looks like this:
![Toplogy](images/topology6.png)  
_Fig. Topology with Transit Peering_  

Have a look at the Dashboard again. You should see all the connectivity on the map.

Topology should show the newly created, beautiful Multi-Cloud full mesh, which should look like this:  
![Toplogy](images/copilot-transit-peering.png)  
_Fig. Copilot Topology with Transit Peering_

## Lab 2.7 - Connection to the On-Prem Datacenter
### Description
At this point, all cloud resources should be connected and able to communicate with one another.  The only part that is missing is the connection to On-Prem.
### Validate
In order to connect our Multi-Cloud environment to On-Prem, we will use the ```aviatrix_transit_external_device_conn``` terraform resource.  This allows us to create a secure link between Cloud and On-Prem, and to enable dynamic routing (BGP).  

Before adding On-Prem to our Multi-Cloud environment, let’s enable ```Route Approval```  Enabling Route Approval is a best practice as one can control the routes learned from On-Premise before distributing them in the cloud.  

To enable Route Approval, navigate to **_Multi-Cloud Transit -> Approval -> Select gcp-transit_** and switch the knob from **Disabled** to **Enabled**.  

Now let’s add the On-Premise Datacenter connection.  Open ```/sva-code/main.tf``` to configure the connection. Do not delete previous configuration!

[Terraform Aviatrix Transit GW Peering](https://registry.terraform.io/providers/AviatrixSystems/aviatrix/latest/docs/resources/aviatrix_transit_external_device_conn)

Enter the following information:

|   Attribute   |   Value        |
| ------------------ | ----------------- |
|  **vpc_id**    |   gcp-transit  -  aviatrix-lab-pod-[pod#]    |
|  **connection_name**  |  MyOnPrem  |
|  **gw_name**  |  gcp-transit  |
|  **connection_type**  |  bgp  |
|  **bgp_local_as_num**  |  650[pod#] _For Pods 1-9, pad the pod# with an additional 0 (ie. 65004)_  |
|  **bgp_remote_as_num**  |  65000  |
|  **remote_gateway_ip**  |  <ip-address> _Please resolve the FQDN onprem-cne-gw.sva.aviatrixlab.de_  |
|  **pre_shared_key**  |  mapleleafs  |
|  **local_tunnel_cidr**  |  169.254.[pod#].2/30  |
|  **remote_tunnel_cidr**  |  169.254.[pod#].1/30  |


<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/s2c_onprem.png" width="800">

Save the file and run ```terraform apply``` Enter ```yes``` 

After 1-2 minutes, under the ```Site2Cloud ``` menu option, you should see that the connection to On-Prem is green.  In order to test connectivity between cloud and on-prem, a test VM is available with the FQDN onprem-cne-priv.sva.aviatrixlab.de.

* Connect into _GCP-SRV1_
* Run the following commands:
```
ping onprem-cne-priv.sva.aviatrixlab.de
```
> Was the ping successful?

Let’s go back to the **_Multi-Cloud Transit -> Approval -> gcp-transit_** to see which Routes we have learned.  
![Toplogy](images/route-approval.png)  
_Fig. Route Approval_  

Select the appropriate Routes to approve (_10.254.0.0/20_), click _Approve_ and _Update_ and then re-run the connectivity tests. If it does not ping immediately, give it about a minute.

* Connect into _AWS-SRV1_
* Run the following commands:
```
ping onprem-cne-priv.sva.aviatrixlab.de
mtr onprem-cne-priv.sva.aviatrixlab.de
```
* Connect into _GCP-SRV1_
* Run the following commands:
```
ping onprem-cne-priv.sva.aviatrixlab.de
mtr onprem-cne-priv.sva.aviatrixlab.de
```
> Was the ping successful?

### Bonus 
One of your customers wants to establish IPSEC tunnel to your AVX Transit GW. They asked what is your Transit Gateway Public IP.

Open ```/sva-code/outputs.tf``` and configure output value which returns  AVX Transit GW Public IP.

[Terraform Aviatrix Transit GW Refferences](https://registry.terraform.io/providers/AviatrixSystems/aviatrix/latest/docs/resources/aviatrix_transit_gateway#gw_name)

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/transit_gw_eip.png" width="800">

Save the file and run ```terraform apply``` Enter ```yes```  

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/transit_gw_eip_apply.png" width="600">


### Expected Results
After adding the connection to on-prem and approving the learned routes, the connectivity tests should be successful.  Our lab environment now looks like this:  
![Toplogy](images/topology7.png)  
_Fig. Topology with On-Prem Connectivity_