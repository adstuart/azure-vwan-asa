# Connecting a Cisco ASA to Azure Virtual WAN via IPsec VPN

It is possible to terminate S2S VPN's natively in Azure using either native VPN Gateways (that run inside a customer-managed VNet), or by using a Virtual WAN Hub that has the S2S function enabled.

An existing article highlights the pattern options for HA S2S VPN connectivity to native Azure VPN Gateways - https://learn.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-highlyavailable. This is good pre-reading before jumping in here.

I will borrow some diagrams from this article below.

## Context, why Virtual WAN (VWAN) is different for S2S VPN termination

Whenever we deploy a VPN termination function in Azure, it always comes with an element of HA/resilience under the covers. There are two nodes behind the scenes, and how, and where they operate depends on the logical config we choose.

### Resilience deployment options

**Active/Standby**

![](images/2022-09-26-08-21-31.png)

With regular VPN Gateways, we have the option of running the underlying nodes in an A/S state, wherein we always connect our remote sites to the primary active node, via a single tunnel from each remote On-Premises public IP endpoint. The main benefit of this deployment type, is that if the Active node fails, both the tunnel endpoint PiP, **as well as any BGP configuration** fails over to the standby node.

> I.e. The On-Prem device/firewall can be configured with a single tunnel, and even if there is a node failure in Azure, service will resume automatically after the standby nodes takes over (~30 seconds). This is an important point in the context of Cisco ASA, as we will see later.

> This failover behaviour within active/standby includes both the tunnel PiP failover, as well as any associated BGP speaker endpoint within Azure (i.e. your BGP sessions will come back online automagically)

> NB. Azure Virtual WAN does not support Active/standby deployment type

**Active/Active**

![](images/2022-09-26-08-27-35.png)

With regular VPN Gateways we also have the option of running the underlying nodes A/A, wherein the remote On-Prem device builds multiple tunnels to active Azure endpoints (to multiple PiPs). This has benefits in terms of failover flexibility and throughput. Azure Virtual WAN VPN function **always runs in active/active mode**.

However, this A/A design comes with one large consideration that is implied by the diagram, your On-Prem device must build multiple tunnels that are always active, and again this has impact on our Cisco ASA design when using BGP as we will talk about later.

Spoiler, pay attention to [this](https://learn.microsoft.com/en-us/azure/virtual-wan/virtual-wan-faq#what-happens-if-the-on-premises-vpn-device-only-has-1-tunnel-to-an-azure-virtual-wan-vpn-gateway) FAQ item within VWAN docs. (The same applies to A/A VPN Gateways outside of VWAN)

![](images/2022-09-26-08-33-20.png)


## Cisco ASA to VWAN

### Static routing - single tunnel

Let us consider the most basic connection. Single tunnel, static routes.

![](images/2022-09-26-08-38-07.png)

![](images/2022-09-26-08-38-14.png)

What would happen if VWAN VPN instance1 failed? Instance0 would take-over the PiP and service would continue after a short outage. (Remember we are not running BGP here)

### Static routing - dual tunnel

What if you want to run multiple tunnels along with static routing?

![](images/2022-09-26-08-41-27.png)

You cannot do ECMP outbound on an ASA with static routes to the same destination IP range, pointing out of multiple interfaces

![](images/2022-09-26-08-42-56.png)

Therefore you are stuck using one tunnel outbound (OnPrem>Azure), but Azure will return traffic over either tunnel (VWAN>OnPrem), which will cause traffic to be asymmetrical in respect to which tunnel/VTI it ingresses in upon, which in 99% of cases will break your ASA function (Watch out for false positives if testing with ICMP inspect disabled.)

### BGP - single tunnel

This is another problematic config to watch out for when using VWAN or A/A VPN Gateways, in combination with Cisco ASA (or any device that does not support sourcing the BGP session from a loopback interface).

![](images/2022-09-26-08-46-35.png)

As per the [note](https://learn.microsoft.com/en-us/azure/virtual-wan/virtual-wan-faq#what-happens-if-the-on-premises-vpn-device-only-has-1-tunnel-to-an-azure-virtual-wan-vpn-gateway) referenced above, your BGP endpoint won't failover, therefore in a failure scenario your IPsec tunnel will failover to the other VWAN instance, but your BGP session will remain down and require manual re-config.

### BGP - multiple tunnel

The main gotcha with this config, as highlighted in [this](https://github.com/jwrightazure/lab/tree/master/asa-vpn-to-active-active-azurevpngw-ikev2-bgp) lab by Jeremy Wright, is that you need two Public IPs (and two outside interfaces) on your Cisco ASA. This is because Cisco ASA does not support loopback, and you can therefore not source your BGP session from a loopback.

![](images/2022-09-29-22-20-53.png)

> You will need to use [as-path-prepend](https://github.com/adstuart/azure-vpn-s2s/tree/main/active-active-aspath) to ensure path symmetry if using the dual BGP tunnel approach, otherwise you will get asymmetry and stateful drops on the ASA. 

This is in contrast to a design based on Cisco CSR, that does support loopbacks, wherein you can achieve the same design using only one outside interface. Again Jeremy delivers. https://github.com/jwrightazure/lab/tree/master/VWAN. (And of course you can also ECMP, because you don't care about TCP state violations on a router that is not performing firewall functions)

It's worth noting, that in this design, you will end up with VWAN trying to initiate **Four** tunnels to On-Prem, with only two actually being used.

![](images/2022-09-29-22-22-00.png)

Failover based on tunnel going down, and BGP re-syncing routing info, has been tested in the region of 30 seconds.

## Summary

Cisco ASAs are firewalls, not routers, therefore ff you want to connect a Cisco ASA to VWAN (or a normal a/a Azure VPN Gateway), and you want a HA design with automatic failover, then you should either;

1) If you are happy with static routing, setup a single ASA outside interface, a single external onprem public IP, and a single tunnel to a single VWAN instance.

or

2) If you need BGP dynamic routing, this require multiple tunnels to VWAN instance 0+1, which will require multiple onprem public IP, and two outside ASA interfaces. And don't forget that even with two active tunnels, you will need to prepend to ensure symmetrical flows

## Thanks

Jeremy Wright (Microsoft) for his great labs and guides related to this subject.

James Anderson (SCC) for his collaboration in testing out these scenarios.


