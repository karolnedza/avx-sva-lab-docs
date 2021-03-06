# Lab 3  

## Security Features

In this lab, we are going to explore some of the security features that Aviatrix provides. Currently we have open and unlimited communication between all VPC’s and VNET’s and Site2Cloud connection. We rely on security groups/NSG’s to secure the workloads. But what if we need more deliberate segmentation? Aviatrix provides the concept of Security Domains. Let’s see how these work and what they can do for us.

## Lab 3.1 - Enable Segmentation on Transit Gateways
### Description
Until now we have set up a relatively flat any to any network.  Now let's see how we can segment our network by using Aviatrix Multi-Cloud Network Segmentation.
### Task

Your Junior collegue deployed two Transit Gateways from UI and now you need to go to Controller UI **_Multi-Cloud Transit -> Segmentation_**. and enable segmentation for  2 transit gateways: Azure and GCP. 

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/segmentation.png" width="800">

Select the gateway and click enable. **Make sure to do this only for GCP and Azure transit gateways**.

We can use terraform to enable segmentation on AWS Transit GW we created earlier. To do this modify Transit GW configuration by adding  

 ```enable_segmentation = true```

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/transit_gw_segmentation.png" width="800">


Save the file and run ```terraform apply``` Enter ```yes```  

Terraform should modify Transit GW attribute enable_segmentation

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/tgw_segmentation_apply.png" width="800">


### Expected Results
Now that segmentation is enabled on the transits, we can continue and build out Security Domains.

## Lab 3.2 - Create Security Domains
### Description
Now we are going to create some security domains, which we can use for segmentation.

### Task
Create the following security domains: _Red_, _Blue_, _Shared_ and _Onprem_.

Open ```/sva-code/main.tf``` and Create the following security domains: _Red_, _Blue_, _Shared_ and _Onprem_.

[Terraform Aviatrix Transit GW Peering](https://registry.terraform.io/providers/AviatrixSystems/aviatrix/latest/docs/resources/aviatrix_segmentation_security_domain)

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/segmentation_domains.png" width="800">

Save the file and run ```terraform apply``` Enter ```yes```  


### Expected Results
The 4 security domains should now be created.

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/seg_domains_apply.png" width="800">


## Lab 3.3 - Create Connection Policies
### Description
In order to specify allowed Security Domain to Security Domain communication, we need to set up some Connection Policies.
### Task
Open ```/sva-code/main.tf``` and modify the _Shared_ security domain so it is connected to _Red_ and _Blue_. Also connect security domain _Onprem_ to _Red_.  

[Terraform Aviatrix Transit GW Peering](https://registry.terraform.io/providers/AviatrixSystems/aviatrix/latest/docs/resources/aviatrix_segmentation_security_domain_connection_policy)

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/segmentation_policy.png" width="800">

Save the file and run ```terraform apply``` Enter ```yes```  


### Expected Results
The above connection policies should now be created.

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/seg_policy_apply.png" width="800">


## Lab 3.4 - Add VPC’s/VNET’s/S2C to Security Domains
### Description
Now we will add each of the Spokes and On-Prem connection to the security domains.
### Task
Open ```/sva-code/main.tf```  and create the following associations:

| Transit Gateway Name | Attachment Name | Security Domain Name |
| ------ | ----------- | ---------- |
| aws-transit   | aws-spoke1 | Red |
| aws-transit   | aws-spoke2 | Blue |
| aws-transit   | shared-service | Shared |
| gcp-transit   | gcp-spoke1 | Red |
| azure-transit   | azure-spoke1 | Blue |
| gcp-transit   | MyOnPrem | Onprem |
		
[Terraform Security Domain and Transit Gateway Attachment Associations](https://registry.terraform.io/providers/AviatrixSystems/aviatrix/latest/docs/resources/aviatrix_segmentation_security_domain_association)

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/seg_domain_associations.png" width="820">

Save the file and run ```terraform apply``` Enter ```yes```  


### Expected Results
Once you have done this, have a look at the Security Domain overview in ```CoPilot```. You will find it by clicking **_Security_** in the menu. This should help you clearly understand the Security Domains and the Connection Policies.

## Lab 3.5 - Connectivity Tests
### Description
Now that we have created our segmentation and connection policies, let’s test what connectivity is possible.
### Validate
* Connect into AWS-SRV1
* Run the following commands:
```
ping aws-srv2-priv.pod[x].sva.aviatrixlab.de
ping azure-srv1-priv.pod[x].sva.aviatrixlab.de
ping gcp-srv1-priv.pod[x].sva.aviatrixlab.de
ping shared-priv.pod[x].sva.aviatrixlab.de
ping onprem-cne-priv.sva.aviatrixlab.de
```

> Can you explain why some ping’s were successful and others weren’t?

* Connect into AWS-SRV2
* Run the following commands:
```
ping aws-srv1-priv.pod[x].sva.aviatrixlab.de
ping azure-srv1-priv.pod[x].sva.aviatrixlab.de
ping gcp-srv1-priv.pod[x].sva.aviatrixlab.de
ping shared-priv.pod[x].sva.aviatrixlab.de
ping onprem-cne-priv.sva.aviatrixlab.de
```

> Can you explain why some ping’s were successful and others weren’t?

### Expected Results
Connectivity tests should only be successful when the necessary Connection Policies are in place between two Security Domains.

## Lab 3.6 - FQDN Filtering
### Description
Another security feature the Aviatrix gateways provide, is FQDN filtering. A common challenge in the cloud is protecting cloud instances from accessing untrusted destinations on the internet. We want to make sure they can only gain access to trusted destinations, like update servers, known API endpoints, etc. FQDN based filtering does exactly that, without having to maintain a list of trusted IP addresses in a NSG or SG.

FQDN Egress filtering is supported in multiple clouds, but we are going to configure it in AWS. AWS has the concept of public and private subnets. Aviatrix can apply FQDN egress filtering on both types of networks. Because our test instances are in the public subnet, we need to set up our environment to filter traffic for public instances.

![Egress Diagram](images/egress-diagram.png)  
_Fig. Egress Diagram_  

### Task


First, we are going to deploy a Public Filtering Subnet and a gateway which will do the actual FQDN egress filtering. 

**Please create the egress gateways via the UI, since the spoke VPC are not described in terraform**

Go to the **_Security -> Public Subnet_** page. Click **_Add New_**. Create the gateway according to the settings shown below. In order to select all routing tables, you can use shift or control.  

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/egress-gw1.png" width="800">

We need to create another gateway for our second AWS spoke. Create it with the below settings.

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/egress-gw2.png" width="800">

Let’s create a new tag for FQDN filtering. We can create multiple tags and we can attach multiple tags to gateways.


Open ```/sva-code/main.tf```  create three FQDN tags and attach to the specific FQDN gatways


[Terraform Aviatrix Gateway FQDN tags](https://registry.terraform.io/providers/AviatrixSystems/aviatrix/latest/docs/resources/aviatrix_fqdn)

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/fqdn_tags.png" width="800">

Save the file and run ```terraform apply``` Enter ```yes```  


We have FQDN Gateways and FQDN tags. The only missing part are FQDN rules.

Open ```/sva-code/main.tf```  create three FQDN rules and attach to the specific FQDN tags

[Terraform Aviatrix FQDN rules](https://registry.terraform.io/providers/AviatrixSystems/aviatrix/latest/docs/resources/aviatrix_fqdn_tag_rule)


| Tag | Domain | Protocol & Port | Action |
| ------ | ----------- | ---------- | ---------- |
| all spokes   | www.github.com | ICMP / Empty | Base Policy |
| spoke1   | www.microsoft.com | ICMP / Empty | Base Policy |
| spoke2   | www.ubuntu.com | ICMP / Empty | Base Policy |


Open ```/sva-code/main.tf```  create three FQDN rules and attach to the specific FQDN tags

<img src="https://raw.githubusercontent.com/karolnedza/avx-sva-lab-docs/master/docs/images/fqdn_rules.png" width="800">

Save the file and run ```terraform apply``` Enter ```yes```  


* Connect into AWS-SRV1 (If you are having trouble connecting, disable the tags, try to connect again and then re-enable the tags)
* Run the following commands:
```
ping www.github.com
ping www.microsoft.com
ping www.ubuntu.com
```

> Can you explain why some ping’s were successful and others weren’t?

* Connect into AWS-SRV2
* Run the following commands:
```
ping www.github.com
ping www.microsoft.com
ping www.ubuntu.com
```

> Can you explain why some ping’s were successful and others weren’t?

You can use the same gateways to filter ingress traffic, with [Aviatrix ThreatIQ and ThreatGuard](https://aviatrix.com/resources/solution-briefs/threatiq-threatguard-datasheet-fnl)! Feel free to play around with this if you have some time.

### Expected Results
The Egress gateways should only allow communication to the URLs specified in the tags, and all other Internet bound traffic should be dropped.
