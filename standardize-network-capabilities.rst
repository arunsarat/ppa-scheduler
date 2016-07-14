..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================
Standardize NIC and Networking capabilities
===========================================

https://blueprints.launchpad.net/nova/+spec/standardize-nic-capabilities

We propose to standardize some of the common networking capabilities supported
by NIC and vSwitch and represent these capabilities as specified in
nova capability standardization spec [1] to enable nova scheduler to use
these host capabilities while scheduling virtual servers.

Problem description
===================

Nova scheduler selects a host that satisfies all the constraints specified by
the user,while scheduling a server. Default nova scheduler (filter scheduler)
has a set of filters which implement hard constraints with respect to
hardware capabilities like number of vCPUs, available RAM and free disk
space as python modules. Hardware capability constraints outside of these
standard filters are applied through host aggregate meta-data. Vendor
specific capabilities are tagged to each host or a set of hosts
(host aggregates) and host aggregate filter will match these host specific
tags with user specified required tags to schedule a given server.
User specifies required tags as “extra specs” in user request. There is no
standardized way to expose these capabilities and it’s up to the
administrator to define these tags and same functionality can end up having
different tags across different vendor devices.

Use Cases
---------

* While scheduling a virtual server, a user would desire to pick the compute
  host that supports a capability such as port mirroring with a specified
  granularity on vSwitch or NIC level. Hence we represent the capability to
  have values for NIC and/or vSwitch.

  This also address the case when the user would desire to pick a compute node
  where both the vSwitch and the NIC need to support a feature. For example,
  for performance reasons the user might want to use the Generic Segmentation
  Offload (also known as TCP Segmentation Offload) to off load TCP segmentation
  in the NIC. For this feature obviously the NIC should support it, but also
  the vSwitch should not do the segmentation in software, and hand over large
  segment correctly (i.e. with the right flags in the NIC descriptors set).
  This use case refers to TSO, but same principle applies to any stateless
  offloads. The actual implementation of this would be through a JSON filter
  or equivalent approach and will be addressed in a separate spec.

* Additionally, while scheduling a virtual server, a user would desire to pick
  a host based on a capability such as port mirroring and not necessarily be
  prescriptive about the fact that the capability is present NIC or vSwitch.
  Hence the HW/SW component is modeled as a leaf node. The user will also be
  returned the HW/SW component which was selected.


Proposed change
===============

We propose to standardize the following NIC and vSwitch capabilities.
Standardizing capabilities involve in defining new `nova.objects.fields.Enum`
classes with each one representing on standard capability.

This spec builds on ongoing effort to standardize the host capabilities as
specified in a recent spec [1] and defines some of the most common NICs and
vSwitch capabilities. In open stack environment virtual switch (vSwitch)
implementation depends on hypervisor technology being used for virtualization.
‘OpenVswitch’ [2] is one of the virtual switches that are used with
KVM/QEMU hypervisor, similarly vSphere Distributed Switch [3] for VMWare
vSphere hypervisor and Hyper-V vSwitch [4] with Microsoft Hyper-V.


Data model impact
-----------------

Open stack capability group will be modified to include network capability
group.

class CapabilityGroup(Enum):
    """Groups of capabilities."""
    SYSTEM = 'sys'
    HARDWARE = 'hw'
    HYPERVISOR = 'virt'
    NETWORK = ‘net’


class HwNicCapabilityGroup(CapabilityGroup):
    “”” Group for NIC capabilities.”””
    PREFIX = CapabilityGroup.HARDWARE + ‘:nic’

    PORTSPEED = ‘portspeed’
    CRYPTO_ACCLN = ‘crypto_accln’
    COMPRESSION_ACCLN = ‘compression_accln’
    STATELESS_PROTOCOL_OFFLOADS = ‘stateless_protocol_offloads’
    STATELESS_TUNNEL_PROTOCOL_OFFLOADS=‘stateless_tunnel_protocol_offloads’
    MULTIQUEUE = ‘multiqueue’
    VMDQ = ‘vmdq’
    SRIOV = ‘sriov’
    PROGRAMMABLE_PIPELINE = ‘programmable_pipeline’
    DCB = “dcb”


class NetworkCapabilityGroup(CapabilityGroup):
    “””Group for Network capabilities supported either by NIC or
       vSwitch”””
    PREFIX = CapabilityGroup.NETWORK

    MIRRORING = ‘mirroring’
    L2L3Tunnel = ‘l2l3tunnel’
    SERVICE_CHAINING = ‘service_chaining’


class Capability (Enum):
    pass


Mirroring (net:mirroring)
-------------------------

Mirroring feature enables users to perform intrusion detection, network
performance management, traffic analysis, user monitoring etc. by sending a
copy of a packet received or transmitted on a switch/server port to a monitor
port where network analyzer is attached. Monitor port could be either in
same device (switch/server) or in aa different network device connected through
the network.

In a virtualized environment, mirroring may be used to monitor the traffic
on a virtual machine port. In a virtualized environment with KVM hypervisor,
all the virtual machines instantiated on a compute node will have attachment
port on openvswitch (vSwitch for KVM) and user can enable port mirroring on
these ports if needed. Besides the openvswitch, mirroring can be enabled on
compute node NICs as well since some of the NICs also support port mirroring.

Mirroring can be enabled either on all the traffic through a port or traffic
that belongs to a specific network or very specific application traffic.
Granularity of the traffic that is being mirrored is configurable.  Here is
the list of mirroring granularity levels.

Mirroring group (MirroringGroup) represents these granularities while enabling
mirroring. Mirroring can be enabled either on OVS running within hypervisor
or on the NIC level. While instantiating a server, user may have a specific
requirement that mirroring capability should be enabled on NIC or on OVS.
Or user may be OK as long as mirroring is enabled either on NIC or OVS.
Mirroring has two modes: SPAN or RSPAN.

SPAN mirroring represents the case when the monitoring port is local and
RSPAN mirroring represents the case when the monitoring port is remote [6].
For RSPAN, there has to be tunnel to stream the mirroring traffic between
the source and destination.


class MirroringGroup(NetworkCapabilityGroup):
    “””Group for mirroring capabilities, representing level of
       granularity.”””
    PREFIX = super(self).PREFIX + ‘:’ + super(self).MIRRORING

    PORT_MIRRORING = ‘port’
    NETWORK_MIRRORING = ‘net’
    FLOW_MIRRORING = ‘flow’


class Mirroring(Capability):
    """Mirroring Capabilities"""
    PREFIX = MirroringGroup.PREFIX + ':'

    MIRROR_PORT_OVS = self.PREFIX + MirroringGroup.PORT_MIRRORING + ‘:ovs’
    MIRROR_PORT_NIC = self.PREFIX + MirroringGroup.PORT_MIRRORING + ‘:nic’
    MIRROR_NET_OVS = self.PREFIX + MirroringGroup.NETWORK_MIRRORING+‘:ovs’
    MIRROR_NET_NIC self.PREFIX + MirroringGroup.NETWORK_MIRRORING + ‘:nic’
    MIRROR_FLOW_OVS = self.PREFIX + MirroringGroup.FLOW_MIRRORING + ‘:ovs’
    MIRROR_FLOW_NIC = self.PREFIX + MirroringGroup.FLOW_MIRRORING + ‘:nic’


L2/L3 Tunnels (net:l2l3tunnel)
------------------------------

L2/L3 tunnels are configured to extend a network over heterogeneous networks
either be it internet or private enterprise networks. Over the years multiple
tunneling protocols have evolved solving different use cases and different
data packet forwarding technologies. This capability group encompasses all
such tunneling encapsulation protocols.

class L2L3TunnelGroup (NetworkCapabilityGroup):
    “”” L2/L3 tunnel group encompasses multiple tunnel types”””
    PREFIX = NetworkCapabilityGroup.PREFIX + ‘:’ +
             NetworkCapabilityGroup.L2L3Tunnel

    VXLAN = self.PREFIX + ‘:vxlan’          # [7]
    GRE = self.PREFIX + ‘:gre’              # [8]
    NVGRE = self.PREFIX + ‘:nvgre’          # [9]
    MPLS = self.PREFIX + ‘:mpls’            # [10]
    VXLANGPE = self.PREFIX + ‘:vxlangpe’    # [11]
    GENEVE = self.PREFIX + ‘:geneve’        # [12]


class L2L3TunnelOvs(Capability):
    POSTFIX = ‘:ovs’

    VXLAN_TUNNEL = L2L3TunnelGroup.VXLAN + self.POSTFIX
    GRE_TUNNEL = L2L3TunnelGroup.GRE + self.POSTFIX
    NVGRE_TUNNEL = L2L3TunnelGroup.NVGRE + self.POSTFIX
    MPLS_TUNNEL = L2L3TunnelGroup.MPLS + self.POSTFIX
    VXLAN_GPE_TUNNEL = L2L3TunnelGroup.VXLANGPE + self.POSTFIX
    GENEVE_TUNNEL    = L2L3TunnelGroup.GENEVE + self.POSTFIX


class L2L3TunnelNic(Capability):
    POSTFIX = ‘:nic’

    VXLAN_TUNNEL = L2L3TunnelGroup.VXLAN + self.POSTFIX
    GRE_TUNNEL = L2L3TunnelGroup.GRE + self.POSTFIX
    NVGRE_TUNNEL = L2L3TunnelGroup.NVGRE + self.POSTFIX
    MPLS_TUNNEL = L2L3TunnelGroup.MPLS + self.POSTFIX
    VXLAN_GPE_TUNNEL = L2L3TunnelGroup.VXLANGPE + self.POSTFIX
    GENEVE_TUNNEL = L2L3TunnelGroup.GENEVE + self.POSTFIX


Physical Port Speed/Bandwidth (hw:nic:portspeed)
------------------------------------------------

All the network interfaces in a physical server will have a specific port
speed either configured or auto-negotiated. Network interface speed
corresponds to bandwidth it supports. In a typical virtualized environment,
all the ports configured on Virtual Machine are attached to ‘vSwitch’
and they’re not directly mapped to a physical interface. East-west traffic
between the Virtual Machines may be switched within ‘vSwitch’ itself and
hence may not even traverse through physical interface.

But, in network deployments where hardware acceleration is enabled by SR-IOV,
each virtual server interface is mapped directly to a logical function
(logical segment of a device). In SR-IOV deployments, even the east-west
traffic between the virtual servers within the same physical host
will traverse the physical NIC. In these kind of deployments, users will be
interested to use NIC bandwidth while scheduling a Virtual Server.  The
following capability group represents all the network interface speeds.

class Portspeed (Capability):
    PREFIX = HwNicCapabilityGroup.PREFIX + ‘:’ +
             HwNicCapabilityGroup.PORTSPEED

    PORT_BW_10_MB = self.PREFIX + ‘:10mb’
    PORT_BW_100_MB = self.PREFIX + ‘:100mb’
    PORT_BW_1_GB = self.PREFIX + ‘:1gb’
    PORT_BW_10_GB = self.PREFIX + ‘:10gb’
    PORT_BW_25_GB = self.PREFIX + ‘:25gb’
    PORT_BW_40_GB = self.PREFIX + ‘:40gb’
    PORT_BW_100_GB = self.PREFIX + ‘:100gb’
    PORT_BW_200_GB = self.PREFIX + ‘:200gb’


Cryptographic HW Acceleration (hw:nic:crypto_accln)
---------------------------------------------------

Cryptographic protocols such as IPSEC, SSL etc. often have better performance
with HW implementations as compared to equivalent SW implementations. Several
NICs (special purpose and general) provide HW acceleration capability for
cryptographic functions like Intel QuickAssit [13] and Agilo CX [14]. A
relevant NFV use case is an IPSEC tunnel from an enterprise branch office.

class CryptoAccln (Capability):
    PREFIX = HwNicCapabilityGroup.PREFIX + ‘:’ +
          HwNicCapabilityGroup.CRYPTO_ACCLN

    SYMM_CRYPTO_IPSEC = self.PREFIX + ‘:symm_ipsec’
    SYMM_CRYPTO_SSL = self.PREFIX + ‘:symm_ssl’
    SYMM_CRYPTO_TLS = self.PREFIX + ‘:symm_tls’
    ASYMM_CRYPTO_DIFFIEH = self.PREFIX + ‘:asymm_diffieh’
    ASYMM_CRYPTO_RSA = self.PREFIX + ‘:asymm_rsa’
    SYMM_CRYPTO_ELLIPTICC = self.PREFIX + ‘:symm_elipticc’


Compression HW Acceleration (hw:nic:compression_accln)
------------------------------------------------------

Compression algorithms such as LZS and Deflate can be accelerated through HW.
Several NICs (special purpose and general) provide HW acceleration capability
for compression functions [15]. A relevant NFV use case is WAN acceleration.

class CompressionAccln (Capability):
    PREFIX = HwNicCapabilityGroup.PREFIX +‘:’+
          HwNicCapabilityGroup.COMPRESSION_ACCLN

    COMPRESSION_ALGORITHM_LZS = self.PREFIX + ‘:‘algo_lzs’
    COMPRESSION_ALGORITHM_DEFLATE = self.PREFIX + ‘:algo_deflate’


Stateless protocol offloads (hw:nic:stateless_protocol_offloads)
----------------------------------------------------------------

TCP/UDP/SCTP checksums, Layer 2 CRC, Large send [16], Large receive [17],
Receive side scaling for TCP traffic [18], based on hash-based filters,
can be offloaded to the NIC for overall performance improvement. For
specific non-hash-based filters, please refer to [19] and [20].  Similarly,
the stateless protocol offloads, vlan tagging and q-in-q [21] can be offloaded
in the NIC so that no CPU cycles are spent on dealing with this type of
tagging provided the NIC descriptors can be set correctly, and the software
stack to avoid doing the tag insertion/extraction.

class StatelessProtocolOffloads (Capability):
    PREFIX = HwNicCapabilityGroup.PREFIX + ‘:’ +
         HwNicCapabilityGroup.STATELESS_PROTOCOL_OFFLOADS

    STATELESS_PROTO_OFFLOAD_TCP_CHECKSUM = self.PREFIX + ‘:tcp_checksum’
    STATELESS_PROTO_OFFLOAD_UDP_CHECKSUM = self.PREFIX + ‘:udp_checksum’
    STATELESS_PROTO_OFFLOAD_SCTP_CHECKSUM = self.PREFIX + ‘:sctp_checksum’
    STATELESS_PROTO_OFFLOAD_LARGE_SEND = self.PREFIX + ‘:large_send’
    STATELESS_PROTO_OFFLOAD_LARGE_RECV = self.PREFIX + ‘:large_recv’
    STATELESS_PROTO_OFFLOAD_RECV_SIDE_SCALING = self.PREFIX +
                                         ‘:recv_side_scaling’
    STATELESS_PROTO_OFFLOAD_LAYER2_CRC = self.PREFIX + ‘:layer2_crc’
    STATELESS_PROTO_OFFLOAD_FLOW_DIRECTOR_FILTER = self.PREFIX +
                                             ‘:flow-director-filter’
    STATELESS_PROTO_OFFLOAD_VLAN_TAG = self.PREFIX + ‘:vlan_tag’
    STATELESS_PROTO_OFFLOAD_QINQ = self.PREFIX + ‘:qinq’


Stateless tunnel protocol offloads (hw:nic:stateless_tunnel_protocol_offloads)
------------------------------------------------------------------------------

The goal of stateless tunnel protocol offloads is to make sure that the above
described stateless protocol offloads are supported for tunnel protocols
encapsulated packet [22]. Today there have been NICs identified to be able to
handle stateless offloads for VXLAN, GRE and GENEVE.

Class StatelessTunnelProtocoOffloads (Capability):
    PREFIX = HwNicCapabilityGroup.PREFIX + ':'
             HwNicCapabilityGroup.STATELESS_TUNNEL_PROTOCOL_OFFLOADS

    STATELESS_PROTO_OFFLOAD_VXLAN = self.PREFIX + ‘:vxlan’
    STATELESS_PROTO_OFFLOAD_GRE = self.PREFIX + ‘:gre’
    STATELESS_PROTO_OFFLOAD_GENEVE = self.PREFIX + ‘:geneve’


Service chaining (net:service_chaining)
---------------------------------------

Service chaining is the process of interconnection of various network functions
(physical or virtual) such as firewall, WAN acceleration etc. to offer an
end-to-end service to a customer. It is also known as a VNF forwarding graph
in ETSI NFV terminology [23]. NSH header [24] facilitates carrying of shared
metadata between network devices and network functions, and between
network functions.

Class ServiceChainingOvs (Capability):
    PREFIX = NetworkCapabilityGroup.PREFIX + ‘:’ +
             NetworkCapabilityGroup.SERVICE_CHAINING
    POSTFIX = ‘:ovs’

    SERVICE_CHAINING_NSH_BYPASS_OVS = self.PREFIX + ‘:nsh_bypass’ +
                                      self.POSTFIX
    SERVICE_CHAINING_NSH_PROCESS_OVS = self.PREFIX + ‘:nsh_process’ +
                                       self.POSTFIX
    SERVICE_CHAINING_NSH_ADD_BASE_OVS = self.PREFIX + ‘:nsh_add_base’ +
                                        self.POSTFIX
    SERVICE_CHAINING_NSH_ADD_OPTIONAL_OVS = self.PREFIX+‘:nsh_add_optional’
                                            + self.POSTFIX


Class ServiceChainingNic (Capability):
    PREFIX = NetworkCapabilityGroup.PREFIX + ‘:’ +
             NetworkCapabilityGroup.service_chaining
    POSTFIX = ‘:nic’

    SERVICE_CHAINING_NSH_BYPASS_NIC = self.PREFIX + ‘:nsh_bypass’ +
                                      self.POSTFIX
    SERVICE_CHAINING_NSH_PROCESS_NIC = self.PREFIX + ‘:nsh_process’ +
                                       self.POSTFIX
    SERVICE_CHAINING_NSH_ADD_BASE_NIC = self.PREFIX + ‘:nsh_add_base’ +
                                        self.POSTFIX
    SERVICE_CHAINING_NSH_ADD_OPTIONAL_NIC = self.PREFIX +
                                     ‘:nsh_add_optional’ + self.POSTFIX


Multiqueue (hw:nic:multiqueue)
------------------------------

Multiqueue allows to segment a physical NIC port to logical queues so that a
multicore system can assign queues to cores for parallel packet processing
through IRQ affinity [25]. Then, through queue assignment technologies
(based on RSS 5 tuple hashing or more advanced techniques such as
flow-director perfect filters) the NIC can spread packets across these
queues – these are discussed in the stateless protocol offload section.

Class Multiqueue (Capability):
    PREFIX = HWNicCapabilityGroup.PREFIX + ‘:’ +
             HWNicCapabilityGroup.MULTIQUEUE
    MULTIQUEUE = self.PREFIX


Vmdq (hw:nic:vmdq)
------------------

Virtual Machine Device Queues (VMDq) is a technology in the HW NIC that
offloads network I/O management burden from the hypervisor [26]. Multiple
queues and sorting intelligence in the HW NIC support enhanced network traffic
flow in the virtual environment, freeing processor cycles for application work.
As data packets arrive at the network adapter, a Layer 2 classifier/sorter in
the network controller sorts and determines which VM each packet is
destined for based on MAC addresses and VLAN tags. It then places the packet
in a receive queue assigned to that VM. The hypervisor’s switch merely routes
the packets to the respective VM instead of performing the heavy lifting work
of sorting data. Thus, VMDq improves platform efficiency for handling
receive-side network I/O and increases CPU utilization for application
processing.

Class Vmdq (Capability):
    PREFIX = HWNicCapabilityGroup.PREFIX + ‘:’ + HWNicCapabilityGroup.vmdq
    VMDQ = self.PREFIX


SR-IOV (hw:nic:sriov)
---------------------

The goal of the PCI-SIG SR-IOV specification is to standardize on a way of
bypassing the hypervisor involvement in data movement by providing
independent memory space, interrupts, and DMA streams for each virtual
machine. SR-IOV architecture is designed to allow a device to support
multiple Virtual Functions (VFs) and much attention was placed on minimizing
the hardware cost of each additional function. SR-IOV-capable devices
provide configurable numbers of independent VFs, each with its own PCI
Configuration space. The hypervisor assigns one or more VF to a virtual
machine. Memory Translation technologies provide hardware assisted techniques
to allow direct DMA transfers to and from the VM, thus bypassing the software
switch in the hypervisor [27]. The SR-IOV companion reference [28] elaborates
on anti-spoofing. The SR-IOV QoS reference [29] elaborates on rate-limiting
and other QoS functionality. The SR-IOV promiscuous reference [30] elaborates
on promiscuous mode support in Intel adapters.

Class SrIov (Capability):
    PREFIX = HWNicCapabilityGroup.PREFIX + ‘:’ + HWNicCapabilityGroup.SRIOV

    SRIOV_VF_QOS_TX_RATE_LIMIT = self.PREFIX + ‘:vf_qos_tx_rate_limit’
    SRIOV_VF_QOS_RX_RATE_LIMIT = self.PREFIX + ‘:vf_qos_rx_rate_limit
    SRIOV_VF_QOS_MULTI_QUEUE_PER_VF = self.PREFIX +
                                      ‘:vf_qos_multi_queue_per_vf’
    SR-IOV_CONFIGURABLE_MTU = self.PREFIX + ‘:configurable_mtu’
    SR-IOV_ANTI-SPOOFING_MAC = self.PREFIX + ‘:antispoofing_mac’
    SR-IOV_ANTI-SPOOFING_VLAN = self.PREFIX + ‘:antispoofing_vlan’
    SR-IOV_UNKNOWN_UNICAST = self.PREFIX + ‘:unknown_unicast’


Programmable_pipeline (hw:nic:programmable_pipeline)
----------------------------------------------------

A more generic way to think of NPUs and FPGAs is a programmable pipeline.
As an example, Programmable ConnectX-3 Pro Adapter [31] has an FPGA which
can be programmed to deliver functions such as DoS attack prevention,
IPSEC encryption etc.

Class ProgrammablePipeline (Capability):
    PREFIX = HWNicCapabilityGroup.PREFIX + ‘:’ +
             HWNicCapabilityGroup.PROGRAMMABLE_PIPELINE

    PROGRAMMABLE_PIPELINE = self.PREFIX


Datacenter Bridging (hw:nic:dcb)
--------------------------------

Data center bridging [32] encompasses a collection of technologies targeted to
enhance Ethernet reliability which is originally designed to be best-effort
network. New data center architectures evolved to have I/O consolidation and
enhanced Ethernet features. I/O consolidation technology like FCoE allows
Fibre channel communication over Ethernet, which leads to dramatic
reduction in number of network adapters, low power consumption and lower TCO.
Fibre channel traffic forces Ethernet to be loss less and data center bridging
technologies enhance Ethernet to be loss less.

IEEE 802.1Qbb Priority Flow Control (PFC) extends the granularity of PAUSE
frame to accommodate the priority queues (.1p).  Using PFC, link is divided
into eight virtual queues and PAUSE can be applied to each individual queue.
With the capability to enable PAUSE on a per-user-priority basis,
a lossless lane for Fibre Channel can be created while retaining packet-drop
congestion management for IP traffic. This mechanism allows storage traffic
to share the same link as non-storage traffic.

IEEE 802.1Qaz Enhanced Transmission Selection (ETS) is used to assign traffic
to a particular virtual lane using IEEE 802.1p class of service (CoS) values
to identify which virtual lane traffic belongs to. Using PFC and ETS allows
administrators to allocate resources, including buffers and queues, based on
user priority, which results in a higher level of service for critical traffic
where congestion has the greatest effect.

IEEE 802.1Qau Quantized Congestion Notification (QCN) enhances the granularity
of Congestion Notification Messages transmission from a switch. Switches always
monitors their output queues and whenever an output queue reaches threshold
(low/high thresholds), they send CNM packets back to the sender (one for every
n packets, where n depends on output queue threshold). Each CNM packet
contains enough information like destination address, priority etc. so that the
sender reduce the transmission rate of that specific traffic.

Class DataCenterBridging (Capability):
    PREFIX = HwNicCapabilityGroup.PREFIX + ‘:’ + HwNicCapabilityGroup.DCB

    DCB_PFC   = DataCenterBridging.PREFIX + ‘:pfc’
    DCB_ETS   =  DataCenterBridging.PREFIX + ‘:ets’
    DCB_QCN =  DataCenterBridging.PREFIX + ‘:qcn’


OpenFlow TTP (net:openflowttp)
------------------------------

Openflow protocol [33] has separated networking devices data path from the
control plane by describing packet processing pipe line (as a sequence of
forwarding tables) independent of control protocols. But vendors use
completely different forwarding pipelines based on switch ASIC specifications
and there is no way standardize the forwarding plane.

Table Type Patterns specification [34] enables users to define abstract
switch models. TTP is one type of Network Data Models that are used to
describe open flow-logical switch. Each TTP includes unique ID, required
OF protocol features, Forwarding Table Map, Meter Table, Flow Table along with
Flow Types and Group Table with group entries. Controller and switches
can use OF-Config protocol to synchronize controller and switch TTP contexts.
ONF Forwarding Abstraction Group defined some standard TTPs [35], which can be
used for negotiation between controller and switch.

Class OpenflowTTP (Capability):
    PREFIX = NetworkCapabilityGroup.PREFIX + ‘:’ +
         NetworkCapabilityGroup.OPENFLOWTTP

    OFDPA = self.PREFIX + ‘:ofdpa’
    IPv4_ROUTER = self.PREFIX + ‘:ipv4-router’


The above are merely examples. Not complete by any means, but meant to
illustrate how to codify the various capabilities within Nova.

Alternatives
------------

None.


REST API impact
---------------

None.


Security impact
---------------

None.


Notifications impact
--------------------

None.


Other end user impact
---------------------

None.


Performance Impact
------------------

None.


Other deployer impact
---------------------

Deployer need to populate inventory table for some of the new standard
resource classes defined in this spec, if these capabilities need to be
considered for scheduling a server. Some capabilities shall be discovered
during host initialization phase, remaining capabilities need to be
configured by the administrator.


Developer impact
----------------

Developers will need to be aware of the new modeling for capabilities
and start consuming these standard capabilities in their modules while
processing the constraints associated with these capabilities instead
of relying on `metadata` and other free-form data modeling.


Implementation
==============


Assignee(s)
-----------

Primary assignee:
 Arun Yerra: Arun_Yerra@DELL.com

Other contributors:
 Ramki Krishnan: Ramki_Krishnan@dell.com
 Joseph Gasparakis: joseph.gasparakis@intel.com


Work Items
----------

1. Add resource class corresponding to each standard hardware capability
2. Populate inventory table for each resource class during host initialization


Dependencies
============

This specification depends on the nova specification [0] that defines the
mechanism to standardize hardware capabilities.


Testing
=======

Not much more than unit testing needed. Much of this blueprint is about
achieving agreement on the various capability groups and codes.


Documentation Impact
====================

None.


References
==========

[1] “Nova capability standardization,”
https://review.openstack.org/cat/309762%2C2%2Cspecs/newton/approved/standardize-capabilities.rst%5E0

[2] “Open Virtual Switch,”
http://openvswitch.org/

[3] “vNetwork Distribute/Standard Switch,”
https://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=1010555

[4] “Hyper-V virtual Switch overview,”
https://technet.microsoft.com/en-us/library/hh831823(v=ws.11).aspx

[5] “Port mirroring with OpenvSwitch,”
http://blog.manula.org/2014/02/port-mirroring-with-openvswitch.html

[6] “Configuring SPAN and RSPAN,”
http://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst2960/software/release/12-2_55_se/configuration/guide/scg_2960/swspan.html

[7] “Virtual eXtensible Local Area Network (VXLAN): A Framework for
     Overlaying Virtualized Layer 2 Networks over Layer 3 Networks,”
https://www.rfc-editor.org/rfc/rfc7348.txt

[8] “Generic Routing Encapsulation (GRE),”
https://www.ietf.org/rfc/rfc2784.txt

[9] “NVGRE: Network Virtualization Using Generic Routing Encapsulation,”
https://datatracker.ietf.org/doc/rfc7637/

[10] “Multi-protocol Label Switching Architecture ,”
https://tools.ietf.org/html/rfc3031

[11] “Generic Protocol Extension for VXLAN,”
https://datatracker.ietf.org/doc/draft-ietf-nvo3-vxlan-gpe/

[12] “Geneve: Generic Network Virtualization Encapsulation,”
https://datatracker.ietf.org/doc/draft-ietf-nvo3-geneve/

[13] “Intel® QuickAssist Technology API Programmer’s Guide,”
https://01.org/sites/default/files/page/330684-001us_api_pg.pdf

[14] “Agilio CX Intelligent Server Adapters,”
https://www.netronome.com/products/intelligent-server-adapters/agilio-cx/

[15] “Intel® QuickAssist Technology API Programmer’s Guide,”
https://01.org/sites/default/files/page/330684-001us_api_pg.pdf

[16] “TCP Segmentation or Large Send Offload,”
http://kb.pert.geant.net/PERTKB/LargeSendOffloadLSO

[17] “Large Receive Offload,”
https://www.usenix.org/legacy/event/usenix08/tech/full_papers/menon/menon_html/paper.html

[18] “Receive Side Scaling for TCP,”
http://www.intel.com/content/www/us/en/support/network-and-i-o/ethernet-products/000006703.html

[19] “Design considerations for efficient network applications with
      Intel® multi-core processor-based systems on Linux,”
http://www.intel.com/content/dam/www/public/us/en/documents/white-papers/multi-core-processor-based-linux-paper.pdf

[20]  “Ethernet Flow Director and Memcached Performance,”
http://www.intel.com/content/www/us/en/ethernet-products/converged-network-adapters/ethernet-flow-director.html

[21] “IEEE 802.1ad,” http://www.ieee802.org/1/pages/802.1ad.html”

[22] “Optimizing the Virtual Network with VXLAN Overlay Offloading,”
https://software.intel.com/en-us/blogs/2015/01/29/optimizing-the-virtual-networks-with-vxlan-overlay-offloading

[23] “ETSI NFV Use Cases,”
http://www.etsi.org/deliver/etsi_gs/nfv/001_099/001/01.01.01_60/gs_nfv001v010101p.pdf

[24] “Network Service Header,”
https://datatracker.ietf.org/doc/draft-ietf-sfc-nsh/?include_text=1

[25] “Multiqueue,“
https://www.kernel.org/doc/Documentation/networking/multiqueue.txt

[26] “Virtual Machine Device Queues,”
https://software.intel.com/sites/default/files/d8/e4/1919

[27] “PCI-SIG SR-IOV Primer,”
http://www.intel.com/content/www/us/en/pci-express/pci-sig-sr-iov-primer-sr-iov-technology-paper.html

[28] “Intel® 82599 SR-IOV Driver Companion Guide,”
http://www.intel.com/content/dam/doc/design-guide/82599-sr-iov-driver-companion-guide.pdf

[29] “Intel SR-IOV QoS,”
http://www.intel.com/content/www/us/en/ethernet-products/converged-network-adapters/config-qos-with-flexible-port-partitioning.html

[30] “Intel SR-IOV Promiscuous Mode,”
http://dpdk.org/ml/archives/dev/2016-March/035870.html

[31] “Programmable ConnectX-3 Pro Adapter Card,”
http://www.mellanox.com/related-docs/prod_adapter_cards/PB_Programmable_ConnectX-3_Pro_%20Card_VPI.pdf

[32] “Data Center Bridging Wikipedia,”
https://en.wikipedia.org/wiki/Data_center_bridging

[33] “OpenFlow Switch Specification,”
https://www.opennetworking.org/images/stories/downloads/sdn-resources/onf-specifications/openflow/openflow-spec-v1.4.0.pdf

[34] “OpenFlow Table Type Patterns,”
https://www.opennetworking.org/images/stories/downloads/sdn-resources/onf-specifications/openflow/OpenFlow%20Table%20Type%20Patterns%20v1.0.pdf

[35] “OpenFlow Table Type Patterns Repository,”
https://github.com/OpenNetworkingFoundation/TTP_Repository


History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Newton
     - Introduced
