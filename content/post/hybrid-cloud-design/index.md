+++
author = "Kaon Thana"
title = "A Case Study in Hybrid Cloud Network Design"
date = "2024-05-17"
description = "The challenges of interconnecting the big 3 cloud providers to form a cohesive solution"
categories = [
    "cloud",
    "observability",
    "netdevops",
    "bgp"
]

aliases = ["hybrid-cloud-design"]
image = "images/front2.png"
+++

## About

A case study in network design for the **hybrid network engineer**. A walkthrough of the year-long journey this network engineer took on to interconnect public cloud workloads from all three major CSPs ([Azure](https://azure.microsoft.com/en-us), [AWS](https://aws.amazon.com/), [GCP](https://cloud.google.com/?hl=en)) + on-prem to provide a robust and highly available solution for the application teams. I will discuss the architectural strategy, lessons learned, pitfalls and wins of the overall solution. 

## Problem Statement

In many organizations, there may exist scenarios where some **applications** are built in one cloud provider (such as GCP) and another application or supporting system run by a different team is built in a different environment (such as AWS). This may be due to specific **cloud offerings** of one provider over another, developer preferences/experience, historical reasons, cost optimization etc. The **why** doesn't really matter - but when these workloads want to communicate - now this becomes a **connectivity puzzle**. 

The answer for many teams is to route this traffic over the **public internet**. Like so:

![](images/f.png)

This solution works and for most orgs thats usually **good enough**.  There are some quick fixes we could implement instead of the above solution, such as to create **site-to-site VPN tunnels** or more recently CSPs are now offering cloud-to-cloud interconnects. However, when you are dealing with dozens of **cloud accounts** (tenants), large amounts of **traffic** and on-prem **data center** traffic these fixes may not scale. Let's step back and take a look at the whole picture.

Some key **info** to take note of:
- Most of the cloud-to-cloud traffic is happening in the **US-East** region. We can focus our efforts here.
- The traffic pattern rates will not exceed **10Gbps** (for now :D)

Can we **architect** a solution that:

1. Decreases cloud egress **costs**
2. Improves **security posture** by reducing the amount of public endpoints that don't need to be exposed
3. Integrates with the existing on premises **data center network**
4. Improves network **latency**

While still mainting the performance, reliability and **agility** of hosting workloads in the cloud...

## Strategy - Weighing the Options

### Option 1 - Hairpin
In my case, I have a production data center already built out and running in **New York**. I could leverage the existing Headquarters data center to provision new circuits to each Cloud Provider and integrate them into the existing **WAN**. Utilize **BGP** routing to bounce traffic back and forth as needed. 

Like So:

![](images/g.png)

#### Pros
- Relatively **short lead time** to complete this option. 
- The **network** already exists (minus the new circuits)

#### Cons
- **Hair-pinning** traffic from US-East1 (Virginia) to NY back to Virginia
- May require hardware refresh depending on **speeds/feeds** available of current switches in my data center. Additional lead-time to buy new gear and set it up.
- Introduces on-prem **SLA factors** (network uptime, upgrades, power work, maintenance windows etc) to **cloud-to-cloud** apps. 

### Option 2 - Greenfield
Rent out space at two co-location providers near Ashburn. Purchase new layer3 switches/routers and order new circuits for the connections.
i.e. **Build it all yourself**

![](images/c.png)

#### Pros
- As a network engineer who loves new toys this would have been great, a **new greenfield deployment**
- Full **control** of the environment
#### Cons
- New Colo Vendor research, approval and onboarding **process** would take some time
- Upfront **capital** costs
- Vendor lead-time on new switches was still high at the time of this design | 1 year+ (post-covid **supply chain issues**)

### Option 3 - Cross Cloud
Utilize the new **cross cloud** interconnect services provided by GCP

![](images/d.png)

#### Pros
- Cloud **Native** Solution
- Simple to setup and **quickly** implement
#### Cons
- Good for GCP to a CSP but doesn't help with other **traffic patterns** for different cloud providers or on-prem workloads
- Cost savings is not **maximized**

### Option 4 - Partner Interconnect
Onboard a **3rd party interconnect** vendor to essentially “rent a router” in Virginia at a monthly cost. Multiple companies exist who provide this **service** such as Megaport, Equinix, PacketFabric and [others](https://cloud.google.com/network-connectivity/docs/interconnect/concepts/service-providers).

![](images/h.png)

#### Pros
- **Flexibility** in choosing regions or zones to deploy partner interconnect routers as needed
- Ephemeral solution that could be **scaled up/down** quickly
- Easy to compare monthly costs vs **monthly savings**
- Follows the overall **cloud-first** mindset of my technology organization
#### Cons
- **New vendor** research, onboarding and learning
- Losing some **control** of the network to another party - i.e nerd knobs, visibility
- Introducing a new vendor into your critical path of traffic - high availability **system design** is crucial


## Design Decisions

We decided on **Option 4**. Using a 3rd party interconnect provider would give us the most flexibility and allow us the option to **dynamically** spin up and down resources/circuits as needed.
After a few weeks of going back and forth with a couple of **providers**, we chose one of them based on community feedback, price and availability of resources.

Additionally, the architecture should:
- Follow the concept of **least privileged access** - meaning don't open up the routing for **ALL** cloud teams to be able to talk to **ALL** other cloud accounts in perpetuity. Narrow the scope down for specific **account-to-account** workloads.
- Implement **99.99% availability** for each cloud provider interconnect (following each CSP published best practice guides).
- Use **devops principles** to spin up resources as code via Terraform, Ansible, Python APIs etc with well defined pipelines. Make the work visible for the entire tech organization and limit institutional knowledge.

### GCP Architecture

- Create a **hub and spoke** type model
- Centralized **networking hub** project to host the GCP Routers + interconnects to partner routers
- Spoke projects will **VPC peer** to hub project for access as needed
- Following the [GCP Best Practices guide](https://cloud.google.com/network-connectivity/docs/interconnect/tutorials/partner-creating-9999-availability), we can design an architecture as shown below:

![](images/j.png)

### AWS Architecture

- Create a centralized **networking account** to host an AWS Transit Gateway which attaches to other spoke accounts.
- Following the [AWS High Resiliency Guide](https://docs.aws.amazon.com/directconnect/latest/UserGuide/high_resiliency.html), it should look something like this:

![](images/k.png)

### Azure Architecture

- Follow Azure Hub-Spoke network topology for peering [VNETs together](https://learn.microsoft.com/en-us/azure/architecture/networking/architecture/hub-spoke?tabs=cli)
- Design [high availability express route](https://learn.microsoft.com/en-us/azure/expressroute/designing-for-high-availability-with-expressroute)

![](images/l.png)

### Partner Router Architecture

- Following high availability patterns, create **two virtual routers** in two different availability zones in Virginia.
- Create **virtual cross connects** for each desired CSP path
- Provision new on-prem **circuits** (physical paths) to partner locations

![](images/m.png)

## Implementation

### Challenges

As with most projects, you can **plan** and design all day long. But once you start **building**, something unexpected always comes up. We ran into some **challenges** along the way but were able to find solutions and push through. Here's some key ones...

#### GCP Network Peering
A [key requirement](https://cloud.google.com/vpc/docs/vpc-peering) for GCP Network Peering is that IP network space cannot overlap. When I originally audited the candidate GCP accounts to peer, I only looked at the **primary ranges** - I took note that there were no collisions and moved ahead. 

However, when it came time to actually peer the networks we immediately found a problem. There were some **secondary ranges** which shared similar space in the `100.X.X.X` space. These secondary ranges are used by the project's **kubernetes** clusters. 

Some possible solutions to fix this:
1. Re-IP these secondary ranges - wasn't a fan of this option as it would cause more work and intrusion on the **application owners** side.
2. **Implement** Google's New Product offering called [Network Connectivity Center](https://cloud.google.com/network-connectivity-center?hl=en) (it was in Beta at the time)
    - The key feature that could help here was the ability to **prefix filter** routes from peering
    - Unfortunately, even with NCC enabled we quickly ran into [another blocker](https://cloud.google.com/network-connectivity/docs/network-connectivity-center/concepts/vpc-spokes-overview) `IPv4 static and dynamic routes exchange across VPC spokes are not supported.`
    - This meant we could not **exchange** dynamically learned routes from the partner interconnect to the spoke accounts. This was a no-go. 

With NCC Peering off the table, we went back to VPC Peering. But this time the decision was to select a few **high value** GCP Projects that made up the majority of the traffic load. Re-IP those secondary kubernetes **ranges** (if needed) and move on.

#### AWS - Account-to-Account Routing Options

The original design assumed to use the existing **Transit Gateway** connections to route the newly introduced traffic to each account. However, after performing a cost reduction **analysis** we realized that the Transit Gateway **transfer costs** were still very high, making the effort of the entire project less appealing. Another option would be to create new connections with **VGW attachments**. 

As an example exercise, if we assume **1000TB** of monthly data transfers:

- 1000 TB monthly transfer cost via Transit Gateway = **~$40k per month**
- 1000 TB monthly transfer cost via Dedicated Virtual Gateways = **~$20k per month**

The numbers are very different based on how you route **within AWS**. So the tradeoff we made was to select a few **heavy hitter** accounts to peer directly with the hub network account. 

To do this, I needed to create dedicated VGWs in each **heavy hitter** account and attach it to a new DXGW instead of using the existing Transit Gateway **architecture**. For the **rest of the accounts** connectivity would still be established via the TGW. 

Additionally, this meant we had to create 2 additional **virtual cross connects** to our partner routers and also pay close attention to the hard limit quotas of [AWS Direct Connect](https://docs.aws.amazon.com/directconnect/latest/UserGuide/limits.html). Although the cost of additional cross connects doubles our intended budget, it still made sense

The new AWS **Architecture** would be as follows:

![](images/n.png)

#### Azure Quirks

Prior to this project, I had almost no **experience** building anything in Azure. The interface felt foreign to me and the **naming conventions and iconography** also took some time to get used to...but in general, there were no major hangups in **Azure** except for one issue:

- Prior to this work, an Azure **site-to-site VPN** had already been setup in one VNET to an on-prem resource.
- Due to this, I was unable to peer this azure VNET to the new **hub vnet**
- To work around this problem, we had to relocate the **site-to-site VPN** to the hub vnet and pay close attention to the routing. For this specific case I used **less specific routes** to [prefer](https://learn.microsoft.com/en-us/azure/expressroute/designing-for-disaster-recovery-with-expressroute-privatepeering) the express route connection over the existing site-to-site VPN


#### Differences of Networking Concepts across CSPs

In general understanding the **different concepts** of VPCs vs VNETs - Projects vs Accounts vs Subscriptions - Global route tables vs regional route tables etc... could be a **whole book** (that I would not be qualified to write). One example that comes to mind which took me by surprise:

- In GCP - US **East1** is in North Carolina but in AWS US East1 is in **Virginia**. Something to keep in mind when thinking about traffic latency and regional disaster recovery scenarios.

### High Availability

As I mentioned previously in the **design** section, implementing a highly available solution is crucial. We don't want to introduce additional **failure events** that do not typically occurr in cloud deployments (i.e. limit the blame on the network :D )

To Summarize the best practice documentation from each provider environment:

- In [GCP](https://cloud.google.com/network-connectivity/docs/interconnect/tutorials/partner-creating-9999-availability), high availability requires **4** partner interconnects across **2** Google Cloud Routers in **2** different regions

- In [AWS](https://docs.aws.amazon.com/directconnect/latest/UserGuide/high_resiliency.html), high resiliency can be achieved with **2** single connections in **2** different direct connect locations

- [Azure ExpressRoutes](https://learn.microsoft.com/en-us/azure/expressroute/designing-for-high-availability-with-expressroute) requires **2** express route circuits in zone redundant virtual gateways

- 3rd Party Interconnect Routers require **2** virtual routers and the virtual cross connects should be in distinct **A/B Availability Zones**

### Routing Decisions

This is where **network engineering** chops matter. The standard routing protocol for all Cloud Service Providers is [Border Gateway Protocol - BGP](https://datatracker.ietf.org/doc/html/rfc4271)

What are some BGP decisions we have to make to design a **reliable and fast network**?

1. Use [Bidirectional Forwarding Detection - BFD](https://datatracker.ietf.org/doc/html/rfc5880) for fast BGP **Failover**
    - Keep in mind that different CSPs may have different **BFD supported values**, for example:
      - In GCP we **must** use the values:
        ```
        Transmit Interval - 1000 ms
        Receive Interval  - 1000 ms
        Multiplier        - 5
        ```
      - However, AWS supports faster values:
        ```
        Transmit Interval - 300 ms
        Receive Interval  - 300 ms
        Multiplier        - 3
        ```
    - When I performed failover testing of these circuits, it took almost **5 seconds** for the GCP traffic to recover as opposed to the AWS failure took under **1 second** to recover. 

2. Understand Route Priorities in Different Cloud **Environments**
   - For example, in AWS its best practice to **influence** traffic using [longest prefix match](https://aws.amazon.com/blogs/networking-and-content-delivery/influencing-traffic-over-hybrid-networks-using-longest-prefix-match/)
   - We can also influence routing **policies** with [BGP Communities](https://docs.aws.amazon.com/directconnect/latest/UserGuide/routing-and-bgp.html)
   - Additionally, using AS PATH Prepending or MED values is another **option**
   - Also keep in mind, Route Priority of [Propogated Routes](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html#route-table-priority-propagated-routes)

3. Should we use Load Sharing **ECMP** across multiple links?

  - Possible cons/negatives of ECMP:
    - **Non-deterministic** route paths - hard to "know" which way your traffic is flowing at all times.
    - **Gray outages** where one path is not working optimally and causes interment issues making it hard to troubleshoot.
    - If a **stateful firewall** is in line to these connections return traffic will get dropped.
    - Load balancing accross different geographical locations will cause variations oif latency and network performance
  - Benefits:
    - Use up more of your alloted bandwidth per link
    - More resiliant to failovers - i.e if a link fails all of your traffic does not go down waiting for BFD to kick in
    - 


### Visibility and Operations

## Overall Wins

### Lowering Monthly Cloud Traffic Costs

### Improved Security Posture and Private Connectivty

### Better Network Latency

### Team Collaboration accross different business Units

## End