---
title: "Post: TCP vs UDP"
categories:
  - Post
tags:
  - Networks
excerpt: "A comparison of TCP and UDP"
---

TCP vs UDP is of one the most common question to be asked in a SWE interview. In this post I will cover TCP and UDP
in details and compare their differences.

## Lower Layers

(klement: This [Stack Exchange Thread](https://serverfault.com/questions/246508/how-is-the-mtu-is-65535-in-udp-but-ethernet-does-not-allow-frame-size-more-than) was very useful in helping my understanding.)

To understand the differences between the UDP and TCP, we have to first understand the lower layers they depend on.

Data Link Layer (ethernet):
* Overview: helps with the transferring of data between two nodes.
* Function:
  * Transfers the **data frames** between two nodes connected by a physical link
* Ethernet protocol:
  * Has a maximum frame size of **1518 bytes** unless **jumbo frames** are supported
  * Usually the **Maximum Transmission Unit**(MTU) is set to the ethernet maximum frame size
  * The protocol **will not** accept data frame larger than maximum frame size
* Jumbo Frames:
  * Special configuration to allow larger than **1518 bytes**
  * To successfully transfer larger than **1518 bytes**, all nodes in the network need to
  support jumbo frames.
  * If any one of the nodes does not support jumbo frame, the data frame
  will not be able a transmit through.

Network Layer (IP):
* Overview: allow transfering of **packets** from a source node to a destination node through X number of nodes
* Function:
  * Deals with routing the packet through multiple intermediate nodes.
  * Fragmentation: break down large packets into multiple smaller packets that is within the MTU limit
* Fragmentation:
  * If the data needed to be transported exceed the MTU, IP protocol will break down the
  packets into smaller packets (fragements)
  * Each fragement packet will set the `fragement number` and set `more fragement` bit (except last fragement) in the header
  * If anyone of the smaller fragment packet is lost/corrupted, the entire large packet is marked as lost.
  * Downside:
    * More protocol overhead - instead of one IP header, will need IP header for each smaller fragment packet
    * brittle - if a fragment is lost/corrupted the entire packet is lost/corrupted
* Note: IP uses both datagram and packets to refer data being sent

## User Datagram Protocol (UDP)

UDP is a lightweight transport layer protocol that optimises for speed
and trades off reliability of data transfer. UDP optimises for speed
by having a **connectionless protocol** and **push** model.

### How UDP works

Unlike TCP, UDP does not have any specific flow that the receiver and sender must follow.
When there are data to be sent, the sender the wrap the data as **datagram** and send it to
the receiver.

**Header**
* Adds Source port and Destination port to the header to allow the OS to know
which process to direct the datagram to.
* Checksum:
  * Instead of traditional checksum, UDP uses a pseudo header method. Prepends IP layer data 
* Fields:
  * Source Port: 16 bits
  * Destination Port: 16 bits
  * Length: 16 bits
  * Checksum: 16 bits

**Max Datagram Size**: Industry standard uses 512 byte.

### UDP use cases

1. DNS: need to optimize for time when querying the server. Small payload -> fit in a single datagram
2. Voice Over IP: can deal with loss but not low latency


## Transport Control Protocol (TCP)

The motivation behind TCP is intended for highly reliable host-to-host protocol.
The protocol must be able to push data, recover from lost or damaged data, receiver must be able to
control the flow of data being sent, allow for multiple process to communicate on TCP on a single machine,
establish a connection.

TCP uses the idea of **packet** to represent the data of one transaction between hosts.


### TCP Headers
* Source Port
* Destination Port
* Sequence Number: 32 bits
  * `SYN=1`: the initial sequence number.
  * `SYN=0`: The sequence number of the first byte in this segment
* Acknowledgment Number: 32 bit
  * The next sequence number the sender (sender or receiver) of the segment is expecting to receive
* Data Offset: 4 bits
  * The number of 32 bit words in TCP header. Used to tell when the data begins
* Reserved: 6 bits
* Control Bits:
  * Used to state the different controls
  * Notables: ACK, SYN, FIN
* Window: 
  * The number of data octets beginning with the one indicated in the 
  acknowledgement field which the sender is willing to accept
  * How many packets after the ACK it is can receive
* Checksum
  * Similar to UDP, the checksum includes pseudo header
* Urgent Pointer

### TCP Concepts

#### Sequence number

All packets will have a sequence number. This allows the other party to send an ACK on a sequence number.
When the receiver acknowledges `x` this means that all packets till `x-1` has been received.

The host can choose any sequence number to start with.

#### TCP connection

##### Opening connection

```txt
    TCP A                                                TCP B

1.  CLOSED                                               LISTEN

2.  SYN-SENT    --> <SEQ=100><CTL=SYN>               --> SYN-RECEIVED

3.  ESTABLISHED <-- <SEQ=300><ACK=101><CTL=SYN,ACK>  <-- SYN-RECEIVED

4.  ESTABLISHED --> <SEQ=101><ACK=301><CTL=ACK>       --> ESTABLISHED

5.  ESTABLISHED --> <SEQ=101><ACK=301><CTL=ACK><DATA> --> ESTABLISHED
```
* TCP connection is established only after three packets has been sent
* Three Way handshake
  1. `A` will establish a connection to `B` and stating the initial sequence number(`100`) that `A` uses.
  2. When `B` receives the packet from `A`, `B` will acknowledge it by stating.
  the next sequence number it wants to receive (`100 + 1`) and state the 
  sequence number it will start with.
  3. When `A` receives the packet from `B`, `A` will acknowledge it by stating that it want to receive the next sequence number (`300+1`)


**Why three-way handshake**: To prevent old connection from causing confusion.

##### Closing connection


```txt
    TCP A                                                TCP B

1.  ESTABLISHED                                          ESTABLISHED

2.  (Close)
    FIN-WAIT-1  --> <SEQ=100><ACK=300><CTL=FIN,ACK>  --> CLOSE-WAIT

3.  FIN-WAIT-2  <-- <SEQ=300><ACK=101><CTL=ACK>      <-- CLOSE-WAIT

4.                                                       (Close)
    TIME-WAIT   <-- <SEQ=300><ACK=101><CTL=FIN,ACK>  <-- LAST-ACK

5.  TIME-WAIT   --> <SEQ=101><ACK=301><CTL=ACK>      --> CLOSED

6.  (2 MSL)
    CLOSED
```

* When `A` has completed sending all data and would like to close the connection,
it will set `FIN` bit.
* When `B` receive the packet, it will ACK `A`'s close connection packet but
will still send any data left on `B` side
* Once `B` has completed sending the data, it will send a close connection packet to `A`
* Once `A` receives the close packet from `B` it will close and send `B` a packet to ACK it
* After receiving `A`'s ACK of closing, `B` will close the connection.

### Data Transfer

TCP has the following key features:

* **Ordered data transfer**: the destination will re-arrange the segments in order
* **Retransmission of lost packets**: any cumulative stream not ACKed will be re-transmitted
* **Error-free data transfer**: corrupted segments are treated as lost and re-transmitted
* **Flow control**: limits the rate a sender transfer data to guarantee reliable delivery
* **Congestion control**: lost packet due to congestion will lead to reduction in data delivery

#### Reliable Transmission

* Uses sequence number to determine the ordering of the segments
* Receiver uses **ACK** to tell the sender that it is now the sender turn to send data

#### Dup-ACK based retransmission

* If a segment (ie: 100) is lost, the receiver cannot send ACK higher than what it never receive (`ACK > 100`)
. The sender will ack the latest in order segment (`ACK=100` the ack sequence number is one more than latest segment).
* Result in the receiver sending duplicate ACK (might have previously send `ACK=100`)
* If the sender receive **3** duplicate ACKs, the sender will send the same segment again 
  * Choose 3 as some segments might arrive out of order causing unneccessary ACKs 

#### Timeout-based retransmission

* Sender will make a conservative estimate of the RTT.
* If the receiver does not ACK with in the threshold time, the sender will retransmit.
* The timeout of the retransmit will be double to allow for a exponential back-off


#### Error detection

* Uses checksum to determine if a segment has been corrupted

#### Flow control

Control the flow of data to allow devices of different specs to communicate

* Act of reducing the transmission rate when the receiver has maxed out its resources
* In each TCP segment, the receiver will state the **receive window**, the number of bytes it is willing to add
to the buffer
* The sender will be able to send `k` number of segments with bytes less than the receive window without waiting
for an ACK by the receiver
* If the receiver advertise window to be `0` (buffer full):
  * The sender will start a persist timer
  * Once persist timer times out, it sender will start by sending a small segment to unlock the dead lock
  * Possible Deadlock: The sender receive that the window `=0` and do not send anymore segments. However, the
  receiver clear the buffer and send a segment to advertise that the window `>0` gets lost. Both the sender and
  receiver will be waiting on each other.

#### Congestion Control


Congestion control is the act of reducing transmission rate when the network is congested.

**Congestion Windows**

* The maximum number of bytes that can be send through the network at anytime.
* Multiple of Maximum Segment Size (MSS)
* Totally different from the receiver sliding window

**Algorithms**:
* Slow start: Start with a small congestion window and keep doubling until retransmission is needed.
Assume loss of packet is due to congestion in the network
* Congestion avoidance:
  * Once the congestion window slow start algorithm reach a certain size, slow start threshold(ssthresh),
  the algorithm will change to linear growth.

#### Handling Multiple connections

A TCP connection session is uniquely identified by the client **IP** and **PORT** number. The
OS will have a Transmission Control Block (**TCB**) that maps the **CLIENT_IP:CLIENT_PORT:SEVER_PORT** 
to the process that holds owns the connection.

#### Maximum segment size

* **Maximum Transmission Unit** (MTU):
  * The largest data that can be transmitted over the network layer (IP).
  * Each device will know the MTU size of the interface it is connected to.
  * If data > MTU, IP layer will split packets into smaller fragments (**IP fragmentation**)
* **Maximum Segment Size** (MSS):
  * The maximum segment size that can be transmitted over the application layer (TCP)
    * Segment is the data component of a TCP packet (does not include TCP headers)
  * Each device will determine on its own MSS based on its own MTU.
    * Normally, `MSS = MTU - sizeof(IP headers)[20 bytes] - sizeof(TCP headers)[20 bytes]`
  * During connection establishment phase, the devices will announce their own MSS

### TCP vs UDP

#### Sending large payload

**UDP**:
* Not good at sending large payload
* Forces all data to be sent as a single datagram through IP layer
* If the payload > MTU, IP layer will be forced to use fragmentation to split the large data into smaller
fragments that are within the MTU
* This will result in the packet to be **easily lost** through the IP layer
  * Any one of the fragment lost/corrupted the entire packet is lost/corrupted
* UDP has not ACK -> cannot recover lost datagram
* Small datagram of size `512` bytes can be reliably transferred with the guarantee that there wont be fragmentation

**TCP**:
* Good for sending large payload
* Does not send a large packet to IP layer with fragmentation
* Split the packet into multiple smaller packets within the size of MSS (implies that the IP packet within the MTU)
  * Each packet send to IP layer is within MSS => no fragmentation => less prone to lost/corruption
* Uses ACK and sequence number to ensure that all the smaller packets are reliably sent
  * Receiver will receive the complete large payload.

#### Broadcast and Mutli-cast

IP multicast:
* IP provides **Internet Group Management Protocol** (IGMP)
* Functionality:
  * Source can send a single datagram to a IGMP IP address and routers will automatically duplicate it and send to members of the group. 
  * Nodes can send message to IGMP to be added to the group.
* A set of reserved IP range 224.0.0.0 to 239.255.255.255 are reserved for IGMP
* A group of multicast destination can hide behind a single IGMP IP address - senders do not need to send the packet to each
member

UDP vs TCP:
* UDP can muilticast but TCP cannot.
* Impossible for TCP to use multicast cast as it requires establishing a connection (the IP address is just a proxy to multiple nodes)
* IGMP is only demultiplex the message cannot multiplex the ACKs from the members