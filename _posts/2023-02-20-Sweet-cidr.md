---
published: true
---
<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-V5SL99F0XX"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-V5SL99F0XX');
</script>

### Intro
SweetCIDR is a cloud network tool designed for Amazon Web Services (AWS). It can help with mapping the attack surface on AWS by answering questions such as "Which target services can an attacker at 10.0.7.8/24 reach in the cloud, by which protocols, and on which ports?" By using data which collected from AWS SDK , the tool can significantly reduce the time required for AWS network penetration testing and security assessments (Assuming a White-Box scenario).

[https://github.com/1lyasam/SweetCIDR](https://github.com/1lyasam/SweetCIDR)

### Background Story
When conducting an internal White-Box AWS cloud network penetration test on a large cloud environment, Usually there is some kind of "assume breach" approach where you already have some control over a chosen Initial access machine.\
Your first task will be enumeration. The goal is to understand and enumerate which IP addresses have open ports, or in other words, "With whom can I communicate from my current position and how."\
You are facing a challenge here because now you need to have a list of IP address ranges. Not only that, but also after having such a list, you will need to preform a very time consuming port scans on those IP ranges (or CIDRs). 
For example, suppose you are in a situation where
1. There are two VPCs (10.0.0.0/16 and 10.2.0.0/16)
2. The scenario simulates initial access from 10.0.0.33 within the first VPC
3. There is a service running on https://10.2.45.9:8444 (Second VPC) with a critical RCE vulnerability that you may want to exploit if you knew about its existence. 
4. The two VPCs have VPC peering between them, and routing is configured between 10.0.0.0/24 to 10.2.45.0/24.

<img src="/images/cidr_example_2.drawio.png"  width="600" height="375" style="border:1px solid #555">
<font size="3"> The Arrow rerpresents attacker's intereset (The communication itself must be bi-directional)</font>

To get a full coverage results you will need to scan not only the 10.0.0.0/8 CIDR (which is crazy amount of addresses) but also preform a port scan on all possible TCP ports (65536) for each IP address. Using the defult Nmap scan will only scan for 1000 well known ports. To refer to our example, 8444 is not listed as one of the well known ports. In this situation, the time required for scanning becomes very unrealistic, especially for a time-limited (sometimes expensive) security assessment.

While frustrated with slow scans, I wanted to find a way to do the scans more efficiently and to sharpen their accuracy. Since the assessment is defined as White-Box, there is no problem using internal information. My immediate thought was to somehow pull information from AWS SDK.

I could use publicly available open-source tools such as [awsipinventory](https://github.com/okelet/awsipinventory), which loops through all network interfaces with the help of [describe_network_interfaces()](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_DescribeNetworkInterfaces.html) AWS SDK function, it is possible to extract all IP addresses. 
The results of such a tool are a good starting point. The tool provides an accurate list of all IPs in the account. But I will still have a knowledge gap regarding the ports. This gap currently can be only closed with a Nmap Port scan.
I realized that I can use Security groups (SG) and Network interfaces (NICs) to close this knowledge gap.

### Security Groups & Network Interfaces
Security groups in AWS are lists of traffic control rules that can be attached to a network interface, which is then attached to a specific resource (EC2, RDS, ELB, etc.). When attached, security groups control the inbound and outbound traffic to and from the instance/service. Each NIC should have at least one security group attached to it.

<img src="/images/sg_example.png"  width="700" height="437" style="border:1px solid #555">

As shown above, each rule specifies the allowed source CIDR, port, and protocol. without an inbound traffic rule, communication with the service is not possible. keeping this in mind, the following logic can be applied:
1. Loop through all Security groups (sg) in the account
1. For each sg, loop through all rules inside the sg
2. For each rule, check if the "source" field value matches a CIDR of our interest
3. For each match, search for all the attached network interfaces(NICs) + capture the port and protocol.
4. For each NIC, take the private and public IP addresses.
5. For each NIC, print the private + public IP address together with port and protocol.

### How does this logic solve the problem ? 

Going back to our example, if the attacker is located on 10.0.0.33 and the vulnerable service on "https://10.2.45.9:8444" accepts communication from 10.0.0.33, there must be an SG rule allowing such communication. So the 10.0.0.33 will be allowed to communicate with the target service if an SG rule allows any of the following CIDR combinations for port 8444:
- 10.0.0.0/8, 10.0.0.0/9, 10.0.0.0/10, 10.0.0.0/11 etc.. until CIDR /26
Also some combinations like 10.0.0.32/27, 10.0.0.32/28 etc.. until /31.
- 10.0.0.33/32 (explicit IP address).
- 0.0.0.0/0 (Any IP within any class)
<a/>

By running the aforementioned logic against each of these combinations, the IP 10.2.45.9 (Our vulnerable service) should be revealed in the results at some point, with a port configuration that might be one of the following:
- port 8444 
- -1 (all ports)
- Some range that contains 8444 (like 7000-9000)

Mission completed, we have a logic that will expose all the possible communication paths for the attacker with target IP + ports\range.
If a port range is used on the SG we will still need another stage of Nmap port scan to find port 8444. But it will be much faster and efficient than scanning hundreds of thousands of addresses. and also, large port ranges are not a best practice anyway, so you can look on it as another finding to raise.\
I implemented this logic into the SweetCIDR tool, You can refer to the [github page](https://github.com/1lyasam/SweetCIDR) of the tool to get some information about how to actually use it.

Here is an example of how you can run the script to map your external components : 
```javascript
SweetCIDR.py -s 0.0.0.0
Scanning now CIDR: 0.0.0.0/0
     processing regions ....
[Some logging prints omitted for visual reasons]
Finished Processing, printing a table of results...

PublicIp        PrivateIp        privateIp  Ports
--------------  -------------  -----------  -------
54.**.**.166  172.31.36.249           81  tcp
54.**.**.166  172.31.36.249           80  tcp
54.**.**.166  172.31.36.249         9005  tcp
34.**.**.43   172.31.38.122           22  tcp
18.**.**.6    172.31.29.82            80  tcp
18.**.**.6    172.31.29.82          3389  tcp
34.**.**.125   172.31.31.3             22  tcp
34.**.**.125   172.31.31.3           8000  tcp
Saved Excel file to 0-0-0-0_11.xlsx
Saved CSV file to 0-0-0-0_11.csv
```
