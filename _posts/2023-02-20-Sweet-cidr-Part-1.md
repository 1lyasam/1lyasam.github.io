---
published: true
---
### TL;DR
SweetCIDR is an AWS cloud network mapping tool. which can help you to answer the question. "Suppose an attacker will be "here", which targets he will be able to reach in the cloud ? by which protocols ? and on which ports ?". 
[https://github.com/1lyasam/SweetCIDR](https://github.com/1lyasam/SweetCIDR)

### Intro
This post is part 1 out of 2 parts series which will discuss the use cases and the reasons for creating the SweetCIDR tool. I created this tool during my work at Controlup to assist with mapping the cloud attack surface efficiently.
This post will discuss the "Internet facing resources" use case which is probably the most critical. Part 2 will deal with internal communication mapping.


### Public Facing

As people who are in charge of cloud security, we always struggle to understand the full picture of our cloud network. The DevOps teams are working hard and fast to develop the cloud infrastructure under very tight schedules and the network picture changes every day. Sometimes this quick pace of work will introduce security problems. Sometimes, very painful ones.  
The most critical question that you would probably ask yourself is, what Is exposed from my cloud infrastructure to the wild internet?  
A brief moment of inattention of a cloud engineer can cause a very sensitive Web UI or management port to be exposed to everyone in the world. If this is combined with another mistake like forgetting to change the system's default password or a CVE, This can quickly escalate to a total security disaster.
Attackers are scanning the whole internet constantly and using tools like Shodan.io and Censys To map their attack surface..
Your exposed System\Web UI\Admin panel might be attacked a few hours or even moments after deployment.
To detect such a situation as quickly as possible, security engineers must constantly map the internet-facing attack surface of the cloud infrastructure.
If you deal with AWS. you can use a script such as [aws_public_ips](https://github.com/arkadiyt/aws_public_ips) to list all your internet-facing IP addresses. Or use the AWS CLI and do describe_{service} for each service and each region which is active in your cloud setup ( “describe_instances”, “describe_load_balancers” etc.)

The problem with those methods is that the output will not exactly point you to what you looked for. Because
1. very frequently, the existence of a public address is not a problem by itself. If the security group of the resource is restricted to a specific IP address that belongs to the organization it might be OK and not interesting to us.
2. We don’t really know which ports are allowed. That fact makes life harder and will require another check like running NMAP or checking the security group configuration for this IP Address.

A more efficient way to approach the public-facing question in AWS would be through security groups. If you will search for “0.0.0.0/0” CIDR in all the inbound SG(security groups) rules in all the regions, you will get a final list for the following values [SG | IPv4 CIDR | Port\s | Protocol].
But that is only half of the way. You will still need to understand where those SGs applied. As a security engineer, you want to see which machine\service uses this inbound rule and check what actual application is listening for this port, for example, an inbound rule for port 443 might be legitimate if the machine to which the rule is attached is a load balancer service for your production SAAS Web UI. on the other hand, if it’s an EC2 with Jenkins Web UI it’s not legitimate. 

### NICs for the rescue

Under the hood, every service which exposes network functionality, whether it’s an EC2, LB or RDS, etc., has an AWS resource which is called “Network interface”. When you apply an SG on an EC2 machine, what happens behind the scenes is that the SG is attached to the Network interface of that machine.

### Sweet CIDR flow

Taking all this information into account, I’ve built the SweetCIDR tool. The tool takes an IPv4 CIDR as an input and retrieves information (InstanceId, IP, Port, Protocol, and more.) about all AWS objects that as part of their SG configuration they allow the CIDR in their inbound rules.
This is achieved by combining information from SGs regarding the rules logic together with network interface information.
This way, you will be able to see the actual components which are exposed to the internet with their public IP, Ports, Protocol, and instance ID.

Here is an example of how you can run the script to map your external components : 
```javascript
C:\Users\*******\Desktop\Projects\SweetCIDR>SweetCIDR.py -s 0.0.0.0
Scanning now CIDR: 0.0.0.0/0
     processing regions ....
[Some logging prints omitted for visual reasons]
Finished Processing, printing a table of results...

PublicIp PrivateIp privateIp Ports
-------------- ------------- ----------- -------
54.**.**.166 172.31.36.249 81 tcp
54.**.**.166 172.31.36.249 80 tcp
54.**.**.166 172.31.36.249 9005 tcp
34.**.**.43 172.31.38.122 22 tcp
18.**.**.6 172.31.29.82 80 tcp
18.**.**.6 172.31.29.82 3389 tcp
34.**.**.125 172.31.31.3 22 tcp
34.**.**.125 172.31.31.3 8000 tcp
Saved Excel file to 0-0-0-0_11.xlsx
Saved CSV file to 0-0-0-0_11.csv
```
