# Global Design of the Walker Net

The Walker Net consists of three parts.

1. A protocol-oriented programming language, which can implement almost all protocol you can see in every layers. 

2. A switch or router engine, which is programmable, configurable to execute the configuration generated by the compiler.

3. A virtual network environment, to test the program.

Here are the designs of three part.

## Protocol-Oriented Language Walka

This language, called Walka? I think, is designed based on P4 language. 

### What p4 can do?

p4 is targeted as a protocol programming on the RMT structure, whether hardware or software. 

The global structure of p4 is like :

Parser

MatchAction
...
MatchAction

Deparser

I just know about Parser and Match-Action, let me talk about this.

### Parser

The Parser structure just like a graph, from one header, you can go to parse another header at the next step base on some condition. For example.

``` P4
parser MyParser(packet_in packet,
                out headers hdr,
                inout metadata meta,
                inout standard_metadata_t standard_metadata) {

    state start {
        transition parse_ethernet;
    }

    state parse_ethernet {
        packet.extract(hdr.ethernet);
        transition select(hdr.ethernet.etherType) {
            0x0800: parse_ipv4;
            0x0806: parse_arp;
            default: drop;
        }
    }

    state parse_arp {
        packet.extract(hdr.arp);
        transition accept;
    }


    state parse_ipv4 {
        packet.extract(hdr.ipv4);
        transition accept;
    }

}
```

in start state, we parse ethernet (we parse the ethernet as default), and then we decide that whether it is an ip packet, or an arp request or response use field "etherType".

I think parser is DAG at first. However, MPLS, VPN, broke my cognition, header parsing is not DAG, it often has some rings.

### Match-Action

In Match-Action, it must be a DAG, every Match-Action unit, we call this a stage. When packet enters a stage, it will match something use some fields, and then execute some actions and jump to another stage decided on what it match out. For example:

```
// You should look through this program from bottom to top.

    action forward(macAddr_t dstAddr, egressSpec_t port) {
        standard_metadata.egress_spec = port;
        hdr.ethernet.srcAddr = hdr.ethernet.dstAddr;
        hdr.ethernet.dstAddr = dstAddr;
        hdr.ipv4.ttl = hdr.ipv4.ttl - 1;
    }
    table ipv4_lpm {
        key = {
            hdr.ipv4.dstAddr: lpm;
        }
        actions = {
            forward;
            drop;
        }
        default_action = drop();
    }
    
    apply {
        if (hdr.ipv4.isValid()) {
            ipv4_lpm.apply();
        } else if (hdr.arp.isValid()) {
            ...
        }
    }
```

in apply(), if parser has extracted the ipv4 header, we can forward this packet in the way described in IPV4 protocol. If arp has been extracted, we will match arp-related table to finish our work.

We will use its ipv4 destination address as key in lpm way, to match the ipv4_lpm table. We will get the value which consists of which action I will execute, and the action parameters: dstAddr and egress port.

In the forward action, we can see the header is changed. The egress port, dest mac address are decided by match table.

We done all of the match action work, finally we will transport this packet to a port we wrote to this packet before.

This is the basic p4 program. however, It's too limited.

### What p4 cannot do?

1. First, p4 is stateless, it cannot do any stateful network function. So, how to design a language that support stateful function?

2. Second, p4 is designed for data plane, rather than all of the network devices, such as router. It cannot send packets to finish some functions that need to switch the control information (such as ACK, RIP). What can we do to send control packets?

3. Third, p4 is designed for RMT structure, it isn't turing-complete. If I program a network function for a software server, p4 isn't a good option. But I like it too much to stop myself to implement more and more function for p4.

So the new language is designed for solve these questions.

### Now Design

As little dependences as possible, I want to use **json** as the target code, and let it as a parameter in the Walka Execute Engine as configuration, whether the parser and match-action, or even flow table. 

It is an all-json configuration.


## Walka Execute Engine

Walka Execute Engine will use Match Action structure like p4, but the functions of actions will be very strong. In action, I want to make sure that it will use many data structures, maybe some limits, but cover all of the network functions.

To make sure that it is stateful, engine will make sure that the statuses' consistence.

We will use more efficient data structure to build the match table, parse routes, and action executor. Stateless and Stateful tables will be used in different structure. Parser will be organized as a real graph.

To give a almost real environment, The physical layer should be given at least two version, error-free and error-high physical layer, to test the efficiency in different environments.

### Now Deign

Now Walker implements the physical layer, which is error-free, and no packet-drop.
I call it utopia-version.

## Walker Virtual Environment

This part will use mininet as the basic environment. But I will change this tool as the Walker virtual environment, heihei.

## Software Design

For the software itself, it need to make sure that:

### indepedent
All submodules are independent modules, it can be used to another project.

For example, the physical layer is an independent module, which can not only build a switch, but also make a packet-capture program like wireshark.

### as little dependences as possible

For a student's project, high efficient programming is not what Walker want, but all are under control, and almost all made in Walker.

By the way, what a pity that configuring the dependencies is what a programmer has to do when he/she wants to run the codes, whether it is well-off or hopeless! So I want to eradicate the dependencies configuring as well as possible.
