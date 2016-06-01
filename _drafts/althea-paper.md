---
layout:     post
title:      Althea Draft 1
---

Althea is a set of protocols for a decentralized point-to-point radio network providing connectivity to an area, where participants in the network are able to pay for connectivity, and receive payment for their contributions to the network, without a centralized ISP collecting subscriptions and owning equipment. It combines the commercial viability of a wireless ISP with the decentralized nature of a community mesh network. There are two main components- payments and routing.

##[1] Routing
Nodes route packets using an ad-hoc “mesh” routing protocol. This protocol must take price as well as link quality into account. We define several extensions to Babel, a popular and performant ad-hoc routing protocol.

Here’s an excerpt to give you an idea of how Babel works:

>2.1. Costs, Metrics, and Neighbourship
>
>As many routing algorithms, Babel computes costs of links between any two neighbouring nodes, abstract values attached to the edges between two nodes. We write C(A, B) for the cost of the edge from node A to node B.
>
>Given a route between any two nodes, the metric of the route is the sum of the costs of all the edges along the route.  The goal of the  routing algorithm is to compute, for every source S, the tree of the routes of lowest metric to S.
>
>Costs and metrics need not be integers.  In general, they can be values in any algebra that satisfies two fairly general conditions (Section 3.5.2).
>
>A Babel node periodically broadcasts Hello messages to all of its neighbours; it also periodically sends an IHU ("I Heard You") message to every neighbour from which it has recently heard a Hello.  From the information derived from Hello and IHU messages received from its neighbour B, a node A computes the cost C(A, B) of the link from A to B.

Babel provides a mechanism for extensibility, which is the basis for the modifications defined in this paper.

###[1.2] Babel extension: Price-aware routing

Babel is a good fit for routing based on payments because of its method of operation, known as "distance vector". 

![](http://localhost:4000/images/pir1.png)

Distance vector routing works by assigning a quality metric to the links between nodes, where higher is worse. Nodes then gossip information about which nodes they can reach at which quality. From this information, each node is able to build up a routing table containing the all destinations in the network, along with their composite quality metric, and the neighbor to forward packets for a destination.

![](http://localhost:4000/images/pir2.png)

This extension allows a Babel router to attach information about monetary price to the routes that it maintains. The router also propagates this information to its neighbors, who use it to determine their own prices. The price is taken into account for metric computation and route selection. It is also used by a payment protocol external to Babel (defined below in “Payments”) to pay neighbors to forward data.

####[1.2.1] Changes to data structures

A router implementing price-aware routing has one additional field in each route table entry:

- A price field, denoting how much the router charges to forward packets along this route. 

####[1.2.3] Receiving updates

When a node receives an Update TLV, it creates or updates a routing table entry according to Babel, section 3.5.4.  A node that performs price-aware routing extends that procedure by setting the routing table entry price field to `p'`, where: 

    p'=p+l

Let `p` be the price attached to the received Update TLV. Let `l` be the price per kilobyte charged by the Babel router to forward packets along the update’s route. Determination of `l` is implementation-dependent, but for a simple implementation, a single `l` can be used for all routes.

####[1.2.4] Route selection

Route selection is discussed in Babel, section 3.6. The exact procedure is left as an implementation detail but a simple example is:

>routes with a small metric should be preferred over routes with a   large metric;

Similarly, in price-aware routing, routes with a low price should be preferred over routes with a high price. These two criteria both need to factor into the selection. For example, a combined metric m` could be defined as:

    m'=m+(p*n)

where `m` is the metric, `p` is the price, and `n` is a constant multiplier.

Aside: It was hard to choose whether to make this a route selection procedure, extending section 3.6, or a metric computation, extending section 3.5.2. We chose to make it a route selection procedure, as metrics computed by section 3.5.2 are propagated to a node’s neighbors. Since the price is already propagated by this extension, it seems like a bad idea to propagate it again as a factor in the route metric. There is a possibility that this decision will need to be revisited.

###[2] Payments
Each node on the network establishes payment channels with each of its neighbors. A payment channel is a method for two parties to exchange payments trustlessly by signing transactions that alter the balance of an escrow account held by a bank or blockchain (we may use the Ethereum blockchain for Althea).

The important thing about a payment channel is that after the channel has been opened, and funds have been placed in escrow, individual payments can be made directly between the two parties without submitting anything to the bank or blockchain. This means that the entire payment can be completed in one packet. Most payment systems need to send another transmission to a bank or blockchain, and wait for it to be confirmed. This would be too much overhead for use with Althea, which is why payment channels are used instead.

When Alice wishes to send a packet to a destination (Charlie) on the network, she consults her routing table to find the best neighbor to forward it to. This routing table was built up by Babel, taking link quality and price (as computed in [section 1.2]({{ site.url }}/blog/althea-paper/#payments) above) into account, so the neighbor will be the one judged to have the best and cheapest route to the destination. Alice then appends a state update for her payment channel with Bob to the packet which pays him the rate that he is advertising for that destination. When Bob receives the packet and the payment, he forwards the packet on to his best neighbor, paying them the fee they charge to get a packet to that destination. Since Bob has set his fee to slightly higher that what his neighbor is charging to get to that destination, he will make a profit. This process continues until the packet reaches its destination.

![](http://localhost:4000/images/payment-flow.png)

In this way, Alice can send packets to any packet in the network, while transmitters along the way are compensated.

Incentivizing uplinks
This is great if, say, neighbors want to send messages in a local area. But how can this system be used to provide access to the broader internet? It’s important to note that in this system, nodes pay only to send packets. Payments are unidirectional. How can this be used to pay for bidirectional data?

Let’s say that Alice is an end consumer. She has a router that she has loaded up with tokens. Doris is somewhere else on the network, and has a connection to the internet through a commercial ISP or transit provider. We’ll say that Doris is an “uplink” to the rest of the internet.

Bob and Charlie, on the other hand, are intermediary nodes and can carry packets from Alice to Doris and vice-versa. Alice makes a request to Doris for a movie from the internet. Alice pays Bob for the packets of her request to be delivered to Doris, and Bob pays Charlie. Now that Doris has the request, she can get the data from the internet and send it to Alice. Remember, Doris will have to pay for the data to be sent to Alice. Doris probably also wants to be paid for getting the data off the internet, on top of what it costs to send it to Alice.

Alice needs to be able to pay Doris without having to trust Doris. Alice can set up a payment channel with Doris, like she has with her immediate neighbors. As Doris sends Alice data, Alice sends acknowledgements along with channel updates paying Doris. The first time Alice connects to Doris, she sends a blockchain transaction opening a new channel with Doris. Doris submits this to the blockchain. Now Alice can start buying data from Doris.

How does Alice know that Doris has a connection to the internet that she would like to sell? Babel already propagates information about link quality to every node, so it can also propagate information about additional services offered by each node. See __ for the Babel extension needed to do this.

Quality Monitoring:
Alice must be able to verify that intermediary nodes in the local Althea network, and uplink providers are delivering the quality of service that she is paying for.

Intermediary nodes:
Using Babel, Alice builds up a routing table which contains every neighbor’s expected link quality to every destination. One of Alice’s neighbors might be advertising the quality to a given destination falsely.

Uplink providers:
An uplink provider might also try to advertise their capacity incorrectly, supplying less capacity than advertised. This would take the form of dropped packets. These are easily detected. The difficulty is identifying whether the node at fault is an intermediary node, or the uplink.

Nodes consuming internet send payments to the uplink provider along with acknowledgements for received packets. A consumer node simply does not pay if packets are not received. What if a consumer node pays for less data than they have received? The uplink provider does not know whether the packets were lost by the intermediary nodes, or whether the consumer is cheating.