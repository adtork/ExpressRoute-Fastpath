# ExpressRoute Fastpath

Goal of this quick article is to simulate how enabling express-route fastpath effects the traffic flow and performance on a express-route gateway. Fastpath changes the traffic flow by directing inbound flows across an express route circuit directly to the VM from the MSEE, bypassing the gateway. For express route, egress flows are always bypassed as well by default and go straight to the MSEE. The main benefit of enabling Fastpath is it eliminates the GW as a choke point and potential bottleneck, because ultra sku gateway is limited to 10Gbps. Customers that push a lot of traffic and have applications that open many connections regulary exceed GW flow limits. Enabling Fastpath is the best way to elleviate potential peformance problems and gain optimal peformance. The only requirements for Fastpath is that the GW is Ultra Peformance SKU or ERGW3Az (Avail Zones Gateway in Ultra SKU). Current limitations and basic info found in our public doc:
https://docs.microsoft.com/en-us/azure/expressroute/about-fastpath

We are going to cover the scenarios currently in public preview per above, minus the PLS scneario as I don't have the ability to create a direct port circuit. I will also cover running Ntttcp with and without Fastpath on to measure the peformance differences. This lab is for visual purposes only with active VMs. I already have a circuit created ahead of time with BGP peering up and Fastpath enabled on the ExpressRoute connection object. This can be done directly on the portal on the connection object, or using CLI as seen here: https://docs.microsoft.com/en-us/azure/expressroute/expressroute-howto-linkvnet-arm#configure-expressroute-fastpath

# Basic Topology
![image](https://user-images.githubusercontent.com/55964102/183462330-b9c8b451-3cc1-42b2-a810-096658deca47.png)


# Scenario 1: VNET Peering
For this scneario, Spoke-Vnet2 is peered to the Hub-Vnet1 with VNET peering. On the peering connetion, I have made sure I am allowing all the traffic and using remote gateways, in this case our ExpressRoute GW with Fastpath enabled. I will run a tcptraceroute from the GCP VM (192.168.0.3) to Spoke-Vnet2 VM (10.12.0.4) and we should not see the GW as a hop. Its important to note with traceroute, the GW will show as "*", since in Azure we don't respond to traceroute or tcptraceroute. We will then turn off Fastpath and check the traceroute differences

With Fastpath:

172.16.2.90 is MSEE IP on the Priv Peering Circuit. MSFT taks the next IP in /30. Megaport IP on MCR would be 172.16.2.89
![image](https://user-images.githubusercontent.com/55964102/183224651-b0944d34-bcbb-4e0d-afe4-d69f42e1878a.png)
![image](https://user-images.githubusercontent.com/55964102/183463526-fb92365d-dacb-4186-ae16-0eb6e082475a.png)
![image](https://user-images.githubusercontent.com/55964102/183465038-8a93ccfc-dee1-434f-a5f5-b709257dd6d1.png)


Without Fastpath:

Now, after toggling off and waiting about 5 mins we now see an extra HOP, HOP 3. That is the Express-Route GW now that we have disabled Fastpath. Packets still reach the destination since we have peering and enabled use remote gateway
<br>
![image](https://user-images.githubusercontent.com/55964102/183219938-fc8b5c86-0528-447d-9dfc-7456f7735d9b.png)

# Scenario 2: UDR on Gateway Subnet with Next Hop Linux NVA w/ IP Forwarding

For this simple test, I created a UDR on Hub-Vnet1 with destination of 10.12.0.0/25 (Spoke-Vnet2-Default-Subnet VM 10.12.0.4) applied to Express-Route GW Subnet (10.11.0.128/27). I created a Linux test VM with IPforwarding enabled. All traffic hitting the GW subnet will get sent to NVA (10.11.0.164) before being delivered to our VM in Spoke-Vnet2

With Fastpath:

![image](https://user-images.githubusercontent.com/55964102/183224508-a304ed43-81f9-4b29-9786-379049856857.png)
![image](https://user-images.githubusercontent.com/55964102/183464302-9bcc53a3-9bd2-4d2f-b735-fa4c6b4205f7.png)

Without Fastpath:

![image](https://user-images.githubusercontent.com/55964102/183224719-1080faa6-4158-474c-9b46-c691d5b032a5.png)

# Scenario 3: Testing with Ntttcp across the circuit toggling Fastpath
For this test, I used NTtttcp and did two runs each to remote VM in Spoke-Vnet2 **without** the UDR applied. The Azure VM in question has 4vCPUs and the GCP VM only has one vCPU. First two tests are with Fastpath toggled on, and last two with Fastpath toggled off. I followed the public docs for setup: https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-bandwidth-testing#testing-between-vms-running-windows-and-linux

With Fastpath:

Run 1:
```bash
Reciever: (Windows-Azure)
c:\Tools>ntttcp -r -m 8,*,10.12.0.4
Copyright Version 5.36

Sender: (linux-GCP)
adam_torkar@onprem-vm1:~$ ntttcp -s -m 2,*,10.12.0.4 -N -t 60
NTTTCP for Linux 1.4.0
---------------------------------------------------------
20:04:57 INFO: Starting sender activity (no sync) ...
20:04:57 INFO: 2 threads created
20:04:57 INFO: 2 connections created in 125099 microseconds
20:05:57 INFO: Test run completed.
20:05:57 INFO: Test cycle finished.
20:06:29 INFO: 2 connections tested
20:06:29 INFO: #####  Totals:  #####
20:06:29 INFO: test duration    :60.00 seconds
20:06:29 INFO: total bytes      :2752512
20:06:29 INFO:   throughput     :367.00Kbps
20:06:29 INFO:   retrans segs   :0
20:06:29 INFO: cpu cores        :1
20:06:29 INFO:   cpu speed      :2299.998MHz
20:06:29 INFO:   user           :0.44%
20:06:29 INFO:   system         :1.14%
20:06:29 INFO:   idle           :98.41%
20:06:29 INFO:   iowait         :0.00%
20:06:29 INFO:   softirq        :0.00%
20:06:29 INFO:   cycles/byte    :796.36
20:06:29 INFO: cpu busy (all)   :3.00%
```
Run 2:
```bash
Same paramaters and flow as per above, slighly better throughput on 60s run

adam_torkar@onprem-vm1:~$ ntttcp -s -m 2,*,10.12.0.4 -N -t 60
NTTTCP for Linux 1.4.0
---------------------------------------------------------
20:13:00 INFO: Starting sender activity (no sync) ...
20:13:00 INFO: 2 threads created
20:13:01 INFO: 2 connections created in 126500 microseconds
20:14:01 INFO: Test run completed.
20:14:01 INFO: Test cycle finished.
20:14:33 INFO: 2 connections tested
20:14:33 INFO: #####  Totals:  #####
20:14:33 INFO: test duration    :60.00 seconds
20:14:33 INFO: total bytes      :2883584
20:14:33 INFO:   throughput     :384.48Kbps
20:14:33 INFO:   retrans segs   :0
20:14:33 INFO: cpu cores        :1
20:14:33 INFO:   cpu speed      :2299.998MHz
20:14:33 INFO:   user           :1.66%
20:14:33 INFO:   system         :1.31%
20:14:33 INFO:   idle           :97.01%
20:14:33 INFO:   iowait         :0.02%
20:14:33 INFO:   softirq        :0.00%
20:14:33 INFO:   cycles/byte    :1428.94
20:14:33 INFO: cpu busy (all)   :2.92%
```
Without Fastpath

Run: 1

```bash
Reciever: (Windows-Azure)
c:\Tools>ntttcp -r -m 8,*,10.12.0.4
Copyright Version 5.36

Sender: (linux-GCP)
adam_torkar@onprem-vm1:~$ ntttcp -s -m 2,*,10.12.0.4 -N -t 60
NTTTCP for Linux 1.4.0

adam_torkar@onprem-vm1:~$ ntttcp -s -m 2,*,10.12.0.4 -N -t 60
NTTTCP for Linux 1.4.0
---------------------------------------------------------
20:28:00 INFO: Starting sender activity (no sync) ...
20:28:00 INFO: 2 threads created
20:28:01 INFO: 2 connections created in 128719 microseconds
20:29:01 INFO: Test run completed.
20:29:01 INFO: Test cycle finished.
20:29:34 INFO: 2 connections tested
20:29:34 INFO: #####  Totals:  #####
20:29:34 INFO: test duration    :60.00 seconds
20:29:34 INFO: total bytes      :2752512
20:29:34 INFO:   throughput     :367.00Kbps
20:29:34 INFO:   retrans segs   :0
20:29:34 INFO: cpu cores        :1
20:29:34 INFO:   cpu speed      :2299.998MHz
20:29:34 INFO:   user           :6.03%
20:29:34 INFO:   system         :1.99%
20:29:34 INFO:   idle           :90.35%
20:29:34 INFO:   iowait         :1.62%
20:29:34 INFO:   softirq        :0.00%
20:29:34 INFO:   cycles/byte    :4836.74
20:29:34 INFO: cpu busy (all)   :2.84%
---------------------------------------------------------
```

Run: 2

Same paramaters and flow as per above:

```bash
adam_torkar@onprem-vm1:~$ ntttcp -s -m 2,*,10.12.0.4 -N -t 60
NTTTCP for Linux 1.4.0
---------------------------------------------------------
20:33:08 INFO: Starting sender activity (no sync) ...
20:33:08 INFO: 2 threads created
20:33:08 INFO: 2 connections created in 128105 microseconds
20:34:08 INFO: Test run completed.
20:34:08 INFO: Test cycle finished.
20:34:41 INFO: 2 connections tested
20:34:41 INFO: #####  Totals:  #####
20:34:41 INFO: test duration    :60.00 seconds
20:34:41 INFO: total bytes      :2621440
20:34:41 INFO:   throughput     :349.52Kbps
20:34:41 INFO:   retrans segs   :0
20:34:41 INFO: cpu cores        :1
20:34:41 INFO:   cpu speed      :2299.998MHz
20:34:41 INFO:   user           :1.50%
20:34:41 INFO:   system         :1.35%
20:34:41 INFO:   idle           :97.15%
20:34:41 INFO:   iowait         :0.00%
20:34:41 INFO:   softirq        :0.00%
20:34:41 INFO:   cycles/byte    :1501.52
20:34:41 INFO: cpu busy (all)   :2.88%
```

## Conclusion
We can see a couple different things from this lab. For the tcptraceroute tests, enabling Fastpath, we see one less hop in the route path, which is an easy way to validate that fastpath is enabled for customers in addition to checking in the UI or using another API call. For the perf tests, we don't get much of a difference with Fastpath enabled or not enabled. In this case my test circuit was only 50Mbps, and I also had 120ms latency pinging from GCP to Azure. Biggest factors that affect perf are iRTT (Round Trip Time), TCP Window Scale Factor, and the application itself (Whether or not it can do parallel connections). You are also only going to be as fast as your lowest denominator. So, even though my ExR GW is ultra-perf (10Gbps) the circuit itself is only 50mbps. Just toggling Fastpath does not seem to greatly alter peformance in this scenario based on the other factors.
