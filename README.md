# ExpressRoute Fastpath

Goal of this quick lab is to simulate how enabling express-route fastpath effects the traffic flow and performance on a express-route gateway. Fastpath changes the traffic flow by directing inbound flows across an express route circuit directly to the VM from the MSEE, bypassing the gateway. For express route, egress flows are always bypassed as well by default and go straight to the MSEE. The main benefit of enabling Fastpath is it eliminates the GW as a choke point and potential bottleneck. Customers that push a lot of traffic and have applications that open many connections regulary exceed GW flow limits. Enabling Fastpath is the best way to elleviate potential peformance problems and gain optimal peformance. The only requirements for Fastpath is that the GW is Ultra Peformance SKU or ERGW3Az (Avail Zones Gateway in Ultra SKU). Current limitations and basic info found in our public doc:
https://docs.microsoft.com/en-us/azure/expressroute/about-fastpath

We are going to cover the scenarios currently in public preview per above, minus the PLS scneario as I don't have the ability to create a direct port circuit. I will also cover running Ntttcp with and without Fastpath on to measure the peformance differences. This lab is for visual purposes only with active VMs. I already have a circuit created ahead of time with BGP peering up and Fastpath enabled on the ExpressRoute connection object. This can be done directly on the portal on the connection object, or using CLI as seen here: https://docs.microsoft.com/en-us/azure/expressroute/expressroute-howto-linkvnet-arm#configure-expressroute-fastpath

# Basic Topology
![image](https://user-images.githubusercontent.com/55964102/183223951-e0cfc480-b071-4e79-aab0-3123d3832d48.png)

# Scenario 1: VNET Peering
For this scneario, Spoke-Vnet2 is peered to the Hub-Vnet1 with VNET peering. On the peering connetion, I have made sure I am allowing all the traffic and using remote gateways, in this case our ExpressRoute GW with Fastpath enabled. I will run a tcptraceroute from the GCP VM (192.168.0.3) to Spoke-Vnet2 VM (10.12.0.4) and we should not see the GW as a hop. Its important to note with traceroute, the GW will show as "*", since in Azure we don't respond to traceroute or tcptraceroute. We will then turn off Fastpath and check the traceroute differences

With Fastpath:

172.16.2.90 is MSEE IP on the Priv Peering Circuit. MSFT taks the next IP in /30. Megaport IP on MCR would be 172.16.2.89
![image](https://user-images.githubusercontent.com/55964102/183224651-b0944d34-bcbb-4e0d-afe4-d69f42e1878a.png)
![image](https://user-images.githubusercontent.com/55964102/183219297-acdfcd63-7534-47cb-8e04-5906a8a973f0.png)
![image](https://user-images.githubusercontent.com/55964102/183219645-c2fbbe43-4ef9-49e7-9ab7-ccfdeb64ab40.png)


Without Fastpath:

Now, after toggling off and waiting about 5 mins we now see an extra HOP, HOP 3. That is the Express-Route GW now that we have disabled Fastpath. Packets still reach the destination since we have peering and enabled use remote gateway
![image](https://user-images.githubusercontent.com/55964102/183219938-fc8b5c86-0528-447d-9dfc-7456f7735d9b.png)

# Scenario 2: UDR on Gateway Subnet with Next Hop Linux NVA w/ IP Forwarding

For this simple test, I created a UDR on Hub-Vnet1 with destination of 10.12.0.0/25 (Spoke-Vnet2-Default-Subnet VM 10.12.0.4) applied to Express-Route GW Subnet (10.11.0.128/27). I created a Linux test VM with IPforwarding enabled. All traffic hitting the GW subnet will get sent to NVA (10.11.0.164) before being delivered to our VM in Spoke-Vnet2

With Fastpath:

![image](https://user-images.githubusercontent.com/55964102/183224508-a304ed43-81f9-4b29-9786-379049856857.png)
![image](https://user-images.githubusercontent.com/55964102/183224112-c620acf8-c41a-4aaa-bdb8-5e4a5fa2cb67.png)

Without Fastpath:

![image](https://user-images.githubusercontent.com/55964102/183224719-1080faa6-4158-474c-9b46-c691d5b032a5.png)

# Scenario 3: Testing with Ntttcp across the circuit toggling Fastpath
To be continued......

