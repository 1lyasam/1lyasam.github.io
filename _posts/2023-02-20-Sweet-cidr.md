---
published: true
---
### TL;DR
SweetCIDR is an AWS cloud network mapping tool. which can help you to answer the following question. "Suppose an attacker will be at 10.0.7.8/24 , Which target services he will be able to reach in the cloud ? by which protocols ? and on which ports ?". 
[https://github.com/1lyasam/SweetCIDR](https://github.com/1lyasam/SweetCIDR)

### Intro

The small tool which I built and going to introduce here, helped me to significantly reduce the time wasted on nmap scanning when doing a white-box AWS network pentest\security assessment. The trick is to combine data collected from AWS SDK together with nmap scanning.


### Background Story
When doing a white-box AWS cloud network pentest on a large cloud environment, The first task that you are dealing with is enumeration. You will need to understand and enumerate which ports are open on which IP addresses. Or in other words “With whom can I communicate from my current position and how”.
You can start by Nmap scanning the 24 CIDR of your current IP. This might be a good starting point for you and chances are good that some low hanging fruits will come up with the scan results. But since as a security consultant\engineer your task is to cover as many vulnerabilities as possible on a wide attack surface that might be not sufficient.
Suppose that you are dealing with the following situation in AWS cloud :
1. There are 2 VPCs 10.0.0.0/16 and 10.2.0.0/16.
2. Your initial access machine is on 10.0.0.33 within the first VPC
3. There is a service running on https://10.2.45.9:8444 (Second VPC) with a critical RCE vulnerability that you might want to exploit if you knew about it's existance.
4. The 2 VPCs have VPC peering between them
5. Routing is configured between 10.0.0.0/24 to 10.2.45.0/24

<img src="/images/cidr_example_2.drawio.png"  width="600" height="375" style="border:1px solid #555">
<font size="3"> The Arrow rerpresents attacker's intereset (The communication itself must be bi-directional)</font>

If you will scan 24 or even 16 CIDRs subnets of your own IP. You might never realize that you missed an easy and critical attack path. On the other hand, In order to discover it with Nmap you will probably need to not only scan 10.0.0.0/8 CIDR, but also scan all possible TCP ports (65536) instead of top 1000 (nmap default). This is because 8444 is not listed as a well known port. 
In this situation, the time which is needed for scanning becomes very unrealistic. Especially for a time limited (sometimes expansive) security assessment.

So while I was frustrated from my slow scans, I wanted to find a way to do the scans much more efficiently and to sharpen their accuracy. And since the assessment is defined as White-box, there is no problem using internal information.
My immediate thought was to somehow pull information from AWS SDK. I knew that I could extract all the IP addresses by using publicly available open source tools such as [awsipinventory](https://github.com/okelet/awsipinventory). This tool does its thing by looping through all the network interfaces with the help of [describe_network_interfaces()](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_DescribeNetworkInterfaces.html) AWS SDK function. This function retrieves information about all network interfaces of the AWS account, regardless of the services they are related to, even though it’s listed under EC2 APIs. It will provide information about NICs of EC2, RDS, OpenSearch, ELB, ELBv2 and probably any other service (since it’s not documented I'm not aware of what is not included but that probably very specific services, if any).
The result of such a tool is a good starting point. We will have an accurate list of all IPs in the account but can I make it any better to fit my scenario ? Maybe I can get hints regarding ports and not only IPs ? I realized that yes !

### Security Groups & Network Interfaces
Security groups in AWS are lists of traffic control rules which can be attached to a network interface which is then attached to a specific resource (EC2, RDS, ELB etc..) . When attached, security groups control the inbound and outbound traffic to and from the instance\service. Each NIC should have at least 1 security group attached to it.

<img src="/images/sg_example.png"  width="700" height="437" style="border:1px solid #555">

As can be seen above, each rule describes the allowed source CIDR, port and protocol. Without having a rule which allows the inbound traffic, communication to the service will not be possible.
Having those things in mind, it will possible to do the following logic :
Loop through all Security groups (sg) in the account
1. For each sg, loop through all rules inside the sg
2. For each rule, check if the source field matches a CIDR of our interest
3. For each match, search for all the attached network interfaces(NICs) + capture the port and protocol.
4. For each NIC, take the private and public IP addresses.
5. For each NIC, print the private + public IP address together with port and protocol.

### How does this logic solve the problem ? 

Going back to our example. The attacker is located on “10.0.0.45”. If the vulnerable service on  https://10.2.45.9:8444 accepts communication form “10.0.0.45”. There must be an SG rule which allows such communication.
By applying the above logic, our attacking server will be allowed to communicate with target server if an SG rule will allow any of the following CIDR combinations for port 8444
10.0.0.0 / 8-26 
10.0.0.45 / 32 (explicit IP address)
By running the logic against each of those combinations the IP 10.2.45.9 should be revealed in the results at some point, with a port configuration that might be one of the following :
port 8444 
-1 (all ports)
Some range that contains 8444 (like 7000-9000)

Mission completed, we have a logic that will expose the possible communication with IP + ports\range. This will definitely save a lot of time ! 
If port range is used we will still need an Nmap scan to find port 8444. But it will be much faster and efficient than scanning hundreds of thousands of addresses.

