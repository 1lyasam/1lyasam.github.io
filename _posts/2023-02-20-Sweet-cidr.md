---
published: true
---
### TL;DR
SweetCIDR is an AWS cloud network mapping tool. which can help you to answer the following question. "Suppose an attacker will be at 10.0.7.8/24 , Which target services he will be able to reach in the cloud ? by which protocols ? and on which ports ?". 
[https://github.com/1lyasam/SweetCIDR](https://github.com/1lyasam/SweetCIDR)

### Intro

The small tool that I'm going to introduce here, helped me to significantly reduce the time wasted on nmap scanning when doing a white-box AWS network pentest\security assessment. The trick is to combine data collected from AWS SDK together with nmap scanning.


### Background Story
When doing a white-box AWS cloud network pentest on a large cloud environment, The first task that you are dealing with is enumeration. You will need to understand and enumerate which ports are open on which IP addresses. Or in other words “With whom can I communicate from my current position”.
You can start by Nmap scanning the 24 CIDR of your current IP. This might be a good starting point for you and chances are good that some low hanging fruits will come up with the scan results. But since as a security consultant\engineer your task is to cover as many vulnerabilities as possible on a wide attack surface that might be not sufficient.
Suppose that you are dealing with the following situation in AWS cloud :
1. There are 2 VPCs 10.0.0.0/16 and 10.2.0.0/16.
2. Your initial access machine is on 10.0.0.33 within the first VPC
3. There is a service running on https://10.2.45.9:8444 (Second VPC) with a critical RCE vulnerability that you might want to exploit.
4. The 2 VPCs have VPC peering between them
5. Routing is configured between 10.0.0.0/24 to 10.2.45.0/24

<img src="/images/cidr_example_2.drawio.png"  width="600" height="375" style=style="border:1px solid #555">
<font size="3"> The Arrow rerpresents attacker's intereset (The communication itself must be bi-directional)</font>
