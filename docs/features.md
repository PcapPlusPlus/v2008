---
layout: page
title: Feature Overview
permalink: /docs/features
nav_order: 3
---

# Feature Overview
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Packet capture

Packet capture (A.K.A packet sniffing or network tapping) is the process of intercepting and logging traffic that passes over a digital network or part of a network (taken from [Wikipedia](https://en.wikipedia.org/wiki/Packet_analyzer)). It is one of the most important and popular features of PcapPlusPlus and it is what PcapPlusPlus is basically all about.

There are multiple packet capture engines out there, the most popular are [libpcap](https://www.tcpdump.org/), [WinPcap](https://www.winpcap.org/) (which is libpcap for Windows), [Npcap](https://nmap.org/npcap/) (WinPcap's successor), [Intel DPDK](https://www.dpdk.org/), [ntop's PF_RING](https://www.ntop.org/products/packet-capture/pf_ring/) and [raw sockets](https://en.wikipedia.org/wiki/Network_socket#Raw_socket). Each engine has different strengths, purposes and features, works on different platforms and obviously has different APIs and setup process. Most of them are written in C (to achieve the best performance) and don't expose a C++ interface.

PcapPlusPlus aims to create a consolidated and easy-to-use C++ API for all of these engines which simplifies their complexity and provides a common infrastructure for capturing, processing, analyzing and forging of network packets.

Here is a list of of the packet capture engines currently supported:

- [libpcap](https://www.tcpdump.org/) live capture (on Linux, MacOS, FreeBSD)
- [WinPcap](https://www.winpcap.org/)/[Npcap](https://nmap.org/npcap/) live capture (on Windows)
- [Intel DPDK](https://www.dpdk.org/) (on Linux)
- [ntop's Vanilla PF_RING](https://www.ntop.org/products/packet-capture/pf_ring/) (on Linux)
- [WinPcap Remote capture](https://www.winpcap.org/docs/docs_412/html/group__remote.html) (on Windows)

The main features provided for each one are:

- Browse all interfaces available on the machine
- Capture network traffic on a specific interface
- Send packets to the network
- Filter packets

### What's next?
{: .no_toc }

You can find out more information in the [API documentation]({{ site.baseurl }}/docs/api), [tutorial pages]({{ site.baseurl }}/docs/tutorials) and browse through the code of the [example apps]{{ site.baseurl }}/docs/examples).

## Packet parsing and crafting

PcapPlusPlus provides advanced capabilities for packet analysis which include:

- Packet parsing, which is a detailed analysis of the protocols and layers of a packet
- Packet crafting which is generation and editing of network packets

A large variety of network protocols are supported, see the full list [here](#supported-network-protocols).

Packets can be analyzed from any source including those captured from the network, packets stored in PCAP/PCAPNG files, and more.
The design of PcapPlusPlus allows similar analysis capabilities for each packet, regardless of where it came from. For example, you can write the same code for parsing packets that are captured using DPDK, libpcap, WinPcap/Npcap, raw sockets or read from a PCAP/PCAPNG file. Same goes for packet crafting.

Consider this simple code snippet that shows how to read a packet from a PCAP file, parse it, identify if it's an IPv4 packet and print the source and dest IP addresses:

```cpp
#include "IPv4Layer.h"
#include "Packet.h"
#include "PcapFileDevice.h"

int main(int argc, char* argv[])
{
    // open a pcap file for reading
    pcpp::PcapFileReaderDevice reader("1_packet.pcap");
    if (!reader.open())
    {
        printf("Error opening the pcap file\n");
        return 1;
    }

    // read the first packet from the file (in this case the
    // file contains only one packet)
    pcpp::RawPacket rawPacket;
    if (!reader.getNextPacket(rawPacket))
    {
        printf("Couldn't read the first packet in the file\n");
        return 1;
    }

    // parse the raw packet into a parsed packet
    pcpp::Packet parsedPacket(&rawPacket);

    // check if it's an IPv4 packet
    if (parsedPacket.isPacketOfType(pcpp::IPv4))
    {
        // extract source and dest IPs
        pcpp::IPv4Address srcIP = parsedPacket.getLayerOfType()->getSrcIpAddress();
        pcpp::IPv4Address destIP = parsedPacket.getLayerOfType()->getDstIpAddress();

        // print source and dest IPs
        printf("Source IP is '%s'; Dest IP is '%s'\n", srcIP.toString().c_str(), destIP.toString().c_str());
    }

    // close the file
    reader.close();

    return 0;
}
```

## Read and write packets from/to files

Network packets can be stored in files, usually for offline analysis. [PCAP](https://wiki.wireshark.org/Development/LibpcapFileFormat) and [PCAPNG](https://github.com/pcapng/pcapng) are the two most popular file formats and both are supported in PcapPlusPlus.

The main features include:

- Read packets from PCAP/PCAPNG files
- Create new PCAP/PCAPNG files and write packets to them
- Use the [packet filtering mechanism](#packet-filtering) to read or write only packets that match the filter
- Append packets to existing PCAP/PCAPNG files
- Write compressed PCAPNG files using [Zstd](https://facebook.github.io/zstd/) compression in real time (OPTIONAL and requires building with Zstd)

Consider this simple code snippet that shows how to open a PCAP file for reading and another PCAPNG file for writing, and then read all packets from the PCAP file and write them to the PCAPNG file:

```cpp
// create a pcap file reader
pcpp::PcapFileReaderDevice pcapReader("input.pcap");
pcapReader.open();

// create a pcapng file writer
pcpp::PcapNgFileWriterDevice pcapNgWriter("output.pcapng");
pcapNgWriter.open();

// raw packet object
pcpp::RawPacket rawPacket;

// read packets from pcap reader and write pcapng writer
while (pcapReader->getNextPacket(rawPacket)) {
  pcapNgWriter.writePacket(rawPacket);
}

pcapNgReader.close(); //These will close when going out of scope
pcapNgWriter.close(); //or you may close them manually
```

For more information please refer to the [Read/Write Pcap Files tutorial]({{ site.baseurl }}/docs/tutorials/read-write-pcap), look at the [API documentation]({{ site.baseurl }}/docs/api) or browse through the code of the [example apps]({{ site.baseurl }}/docs/examples).

### Reading/Writing PCAPNG files with compression
{: .no_toc }

Zstd streaming compression is only supported when working with pcapng files. To enable this feature you must build PcapPlusPlus with Zstd support. For more guidance on building PcapPlusPlus see the [build instructions per platform]({{ site.baseurl }}/docs/install).

Once you have a working build modifying your code to start enabling compression is fast and easy!

When writing PCAPNG files, to enable streaming compression all you need to do is add a second integer argument when constructing your writer. The integer should be between 0-10 and it specifies the compression level. Values outside this range will be clamped. A value of zero, which is also the default, indicates to skip compression. A value of 10 would indicate use maximum compression. For most scenarios a value of 5 or less should be adequate. 

For reading compressed PCAPNG files the only requirement is that the file name extension must terminate in `.zstd`. If a compressed file is supplied to the reader without the `.zstd` extension the file will fail to load. Currently, APPENDING to a compressed file is NOT supported!

If you write code enabling compression, by adding a compression level to your writer constructor, but use a build of PcapPlusPlus without compression support, everything will work just fine and the compression will be skipped/ignored and normal PCAPNG files will be generated/read.

There is a tradeoff between compression speed and compression efficiency. A compression value of 10 will yield the most compression but be slower, while a value of 1 will yield the least compression but be fastest. Depending upon your capture rates and data size you can tune this number to fit your needs. 

Since Zstd is designed to support fast and efficient streaming compression most users should not see any noticeable performance impact when enabling compression. Exact savings from compression will always vary based upon the input data, however; in one test case an uncompressed PCAPNG file of 140MB was duplicated with a compression level of 5 to yield a compressed PCAPNG file of only 40MB giving about 4x space savings! Note that the compression is performed while the file is written so you will not notice any delay when closing the file or a long processing time like you work normally experience when compressing an existing file.

Some example compression code:

```cpp
// create a pcapng file reader
pcpp::PcapNgFileReaderDevice reader("input.pcap.zstd");  //Notice the Zstd extension
reader.open();                                           //This is required for proper loading

// create a pcapng file writer
pcpp::PcapNgFileWriterDevice writer("output.pcapng.zstd", 5);  //The second integer argument 5
pcapNgWriter.open();                                           //is the compression level to use

// read packets from pcapng reader and write pcapng writer
while (reader->getNextPacket(rawPacket)) {
  writer.writePacket(rawPacket);
}
```

## DPDK support

The Data Plane Development Kit (DPDK) is a set of data plane libraries and network interface controller drivers for fast packet processing, currently managed as an open-source project under the Linux Foundation. The DPDK provides a programming framework for x86, ARM, and PowerPC processors and enables faster development of high speed data packet networking applications (taken from [Wikipedia](https://en.wikipedia.org/wiki/Data_Plane_Development_Kit)).

DPDK provides packet processing in line rate using kernel bypass for a large range of network interface cards. Notice that not every NIC supports DPDK as the NIC needs to support the kernel bypass feature. You can read more about DPDK in [DPDK's web-site](https://www.dpdk.org/) and get a list of supported NICs [here](http://core.dpdk.org/supported/).

As DPDK API is written in C, PcapPlusPlus wraps its main functionality in easy-to-use C++ wrappers which have minimum impact on performance and packet processing rate. In addition it brings DPDK to the PcapPlusPlus framework and APIs so you can use DPDK together with other PcapPlusPlus features such as packet analysis, etc.

You can find more information about setting up DPDK and the API in [DPDK support page]({{ site.baseurl }}/docs/dpdk) and also in [DPDK tutorial]({{ site.baseurl }}/docs/tutorials/dpdk).

## PF_RING support

PF_RING™ is a new type of network socket that dramatically improves the packet capture speed. It's providing high-speed packet capture, filtering and analysis (taken from [ntop's website](https://www.ntop.org/products/packet-capture/pf_ring/)).

PcapPlusPlus provides a clean and simple C++ wrapper API for [Vanilla PF_RING](https://www.ntop.org/products/packet-capture/pf_ring/). Currently only Vanilla PF_RING is supported which provides significant performance improvement in comparison to libpcap or Linux kernel, but [PF_RING Zero Copy](https://www.ntop.org/products/packet-capture/pf_ring/pf_ring-zc-zero-copy/) (which allows kernel bypass and zero-copy of packets from NIC to user-space) is not yet supported.

In order to compile PcapPlusPlus with PF_RING you need to:

- Download PF_RING from [ntop's web-site](https://www.ntop.org/get-started/download/#PF_RING)
- Note that PcapPlusPlus was tested with PF_RING versions 6.2.0 - 6.4.1. Later versions aren't guaranteed to work. Please report if you're seeing issues with later versions
- Once PF_RING is compiled successfully, you need to run PcapPlusPlus's `configure-linux.sh` script and type "y" in "Compile PcapPlusPlus with PF_RING?"
- Then you can compile PcapPlusPlus as usual
- Before you activate any PcapPlusPlus application that uses PF_RING, don't forget to enable PF_RING kernel module. If you forget to do that, PcapPlusPlus will output an - appropriate error on startup which will remind you:

```shell
sudo insmod <PF_RING_LOCATION>/kernel/pf_ring.ko
```

## Packet reassembly

Network protocols often need to transport large chunks of data which are complete in themselves, e.g. when transferring a file. The underlying protocol might not be able to handle that chunk size (e.g. limitation of the network packet size), or is stream-based like TCP, which doesn’t know data chunks at all.

In that case the network protocol has to handle the chunk boundaries itself and (if required) spread the data over multiple packets. It obviously also needs a mechanism to determine the chunk boundaries on the receiving side.

This mechanism is called reassembly, although a specific protocol specification might use a different term for this (e.g. desegmentation, defragmentation, etc). 

(this description is taken from [Wireshark documentation](https://www.wireshark.org/docs/wsug_html_chunked/ChAdvReassemblySection.html)).

PcapPlusPlus currently supports two types of packets reassembly:

- IPv4 and IPv6 defragmentation which is a Layer 3 (Network layer) packet reassembly. You can read more about IP fragmentation [here](https://en.wikipedia.org/wiki/IP_fragmentation). To get more information about how it works and the API to use it please refer to the [API documentation]({{ site.baseurl }}/docs/api) and browse through the code of the [IPDefragUtil]({{ site.baseurl }}/docs/examples#ipdefragutil) and [IPFragUtil]({{ site.baseurl }}/docs/examples#ipfragutil) example apps
- TCP reassembly which is a Layer 4 (Transport layer) packet reassembly. To get more information on how it works and the API to use it please refer to the [API documentation]({{ site.baseurl }}/docs/api) and browse through the code of the [TcpReassembly]({{ site.baseurl }}/docs/examples#tcpreassembly) example app

## Packet filtering

Most packet capture engines contain packet filtering capabilities. In order to set the filters there should be a known syntax user can use. The most popular syntax is [Berkeley Packet Filter (BPF)](http://en.wikipedia.org/wiki/Berkeley_Packet_Filter). Detailed explanation of the syntax can be found [here](http://www.tcpdump.org/manpages/pcap-filter.7.html).

The challenge with BPF is that it is too complicated and poorly documented. When compiling BPF filters you often get syntax errors that are hard to understand and debug. Our experience with BPF was not good, so we decided to include in PcapPlusPlus a filter mechanism which is more structured, easier to understand and less error-prone by creating classes that represent filters. Each possible filter phrase is represented by a class. The filter, in the end, is that class.

Consider the following code snippet for creating the filter `src ip 1.1.1.1 and dst port 80` and setting it up on a packet capture device:

```cpp
// create a filter instance to capture only traffic on port 80
pcpp::PortFilter portFilter(80, pcpp::DST);

// create a filter instance to capture only TCP traffic
pcpp::IPFilter ipFilter("1.1.1.1", pcpp::SRC);

// create an AND filter to combine both filters - capture only TCP traffic on port 80
pcpp::AndFilter andFilter;
andFilter.addFilter(&portFilter);
andFilter.addFilter(&ipFilter);

// set the filter on the device
dev->setFilter(andFilter);
```

You can read more in the [PcapFilter.h API documentation]({{ site.baseurl }}/api-docs/_pcap_filter_8h.html) and in the [capture packets tutorial]({{ site.baseurl }}/docs/tutorials/capture-packets#filtering-packets).

## Supported network protocols

PcapPlusPlus currently supports parsing, editing and generation of packets of the following protocols:

1. Ethernet II
2. IEEE 802.3 Ethernet
3. SLL (Linux cooked capture)
4. Null/Loopback
5. Raw IP (IPv4 & IPv6)
6. IPv4
7. IPv6
9. ARP
9. VLAN
10. VXLAN
11. MPLS
12. PPPoE
13. GRE
14. TCP
15. UDP
16. GTP (v1)
17. ICMP
18. IGMP (IGMPv1, IGMPv2 and IGMPv3 are supported)
19. SIP
20. SDP
21. Radius
22. DNS
23. DHCP
24. BGP (v4)
25. HTTP headers (request & response)
26. SSL/TLS - parsing only (no editing capabilities)
27. Packet trailer (a.k.a footer or padding)
28. Generic payload

## License

PcapPlusPlus is released under the [Unlicense license](https://unlicense.org/).
