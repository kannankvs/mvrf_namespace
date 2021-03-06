# Management VRF Design Document

## Introduction
Management VRF is a subset of Virtual Routing and Forwarding (VRF), and provides a separation between the management network traffic and the data plane network traffic. For all VRFs the main routing table is the default table for all data plane ports. With management VRF a second routing table "management", is used for routing through the management ethernet ports of the switch. 

The design for Management VRF leverages Linux Stretch kernel(4.9) Namespace concept for implementing management VRF on SONiC. 

## Requirements
| Req. No | Description                                                                                                | Priority | Comments |
|---:     |---                                                                                                         |---       |---       |
| 1       | Develop and implement a separate Management VRF that provide management plane and Data Plane Isolation
| 2       | Management VRF should be associated with a separate L3 Routing table and the management interface
| 3       | Management VRF and Default VRF should support IP services like `ping` and `traceroute` in its context
| 4       | Management VRF should provide the ability to be polled via SNMP both via Default VRF and Management VRF
| 5       | Management VRF will provide the ability to send SNMP traps both via the Management VRF and Default VRF
| 7       | Dhcp Relay  - Required on Default VRF
| 8       | Dhcp Client - Required on both default VRF and management VRF
| 9       | SSH services should work in both the Management VRF context and Default VRF context
| 10      | TFTP services should work in both the Management VRF context and Default VRF context
| 11      | NTP service should work in Management VRF context.
| 12      | `wget`, `cURL` and HTTPS services should work in both Management VRF context and Default VRF context.
| 13      | `apt-get` package managers should be supported in Management VRF context.
| 14      | TACACS+ should be supported in Management VRF.

The Scope of this design document is only limited to management VRF. "eth0" is the only port that is part of management VRF; Front Panel Ports (shortly called FPP) are not part of management VRF; design can be extended in future if required.

Note that the document uses the words VRF and Namespace interchangeably, both meaning the VRF.

## Design
Namespaces are a feature of the Linux kernel that partitions kernel resources such that one set of processes sees one set of resources while another set of processes sees a different set of resources. 
Without namespace (NS), the set of network interfaces and routing table entries are shared across the linux operating system. Network namespaces virtualize these shared resources by providing different and separate instances of network interfaces and routing tables that operate independently of each other, thereby providing the required isolation for management traffic & data traffic.

By default, linux comes up with the default namespace, all interfaces will be part of this default namespace. Whenever management VRF support is required, a new namespace by name "management" is created and the management interface "eth0" is moved to this namespace. Rest of the ports remain in the default VRF. Following linux commands shall be internally used for creating the same. Commands used in this document are numbered as C1, C2, C3, etc., that are referred in other sub-sections of this document.

    C1: ip netns add management
    C2: ip link set dev eth0 netns management

The default namespace (also called default VRF) enables access to the front panel ports (FPP) and the the management namespace (also called management VRF) enables access to the management interface.
Each new VRF created will map to a corresponding Linux namespace of the same name. After creating the namespace is created, all the configuration related to the namespace happens using the "ip netns exec <vrfname>" command of linux. 
For example, IP address for the management port eth0 is assigned using the command "ip netns exec management ifconfig eth0 10.11.150.19/24" and the default route is added to the management routing table using "ip netns exec management ip route add default via 10.11.150.1". When interfaces are moved from one namespace to another namespace, previously configured IP address & related routes are removed by linux. Design takes care of reconfiguring them using the "ip netns exec" command.
 
     C3: ip netns exec management ifconfig eth0 <eth0_ip/mask>
     C4: ip netns exec management ip route add default via <def_gw_ip_addr>

All the application daemons like SSHD,SNMPD,etc., are running in the default namespace context.
In order to make the applications to handle packets arriving in both management VRF and in default VRF, these applications use the following "veth pair" solution. The veth devices are virtual Ethernet devices that act as tunnels between network namespaces to create a bridge to a physical network device in another namespace. Logical representation of the default VRF, management VRF and the way they talk to each other using veth pair is shown in the following diagram.

![VethPair](Management%20VRF%20Design%20Document%20NS%20VethPair.svg "VRF Representation with Veth Pair") 

Two new internal interfaces "if1" and "if2" are created and they are attached to the veth pair as peers. "if1" is attached to management NS and "if2" is attached to default NS. Internal IP addresses "ip_addr1" and "ip_addr2" are confgiured to them for internal communication. These internal IP addresses ip_addr1 & ip_addr2 are set to 127.100.100.1 & 127.100.100.2 as an example after reordering the already available DOCKER rules in iptables. Following linux commands are internally used for creating the configuring the veth pair and the related internal interfaces.

    C5: Create if2 & have it in veth pair with peer interface as if1
        ip link add name if2 type veth peer name if1
    
    C6: Configure "if2" as UP.
        ip link set if2 up

    C7: Attach if1 to management namespace
        ip link set dev if1 netns management

    C8: Configure an internal IP address for if1 that is part of management namespace
        ip netns exec management ifconfig if1 127.100.100.1/24

    C9: Configure an internal IP address for if2
        ifconfig if2 127.100.100.2/24



### INCOMING PACKET ROUTING

Packets arriving via the front panel ports are routed using the default routing table as part of default NS and hence they work normally without any design change.
Packets arriving on management interface need the following NAT based design. By default, such packets are routed using the linux stack running in management NS which is unaware of the applications running in default NS. DNAT & SNAT rules are used for internally routing the packets between the management NS and default NS and viceversa. Default iptables rules shall be added in the management NS in order to route those packets to internal IP of default VRF "ip_address2".

Following diagram explains the internal packet flow for the packets that arrive in management interface eth0.
![Eth0 Incoming Packet Flow](Management%20VRF%20Design%20Document%20NS%20Eth0%20Incoming%20Pkt.svg "Eth0 Incoming Packets Control Flow") 

Following diagram explains the internal packet flow for the packets that arrive in Front Panel Ports (FPP).
![FPP Incoming Packet Flow](Management%20VRF%20Design%20Document%20NS%20FPP%20Incoming%20Pkt.svg "Front Panel Ports Incoming Packets Control Flow") 

**Step1:** 
For all packets arriving on management interface, change the destination IP address to "ip_address2" and route it. This is achieved by creating a new iptables chain "MgmtVrfChain", linking all incoming packets to this chain lookup and then doing DNAT to change the destination IP as given in the following example. Similarly, add rules for each application destination port numbers (SSH, SNMP, FTP, HTTP, NTP, TFTP, NetConf) as required. Rule C12 is just an example for SSH port 22.

    C10: Create the Chain "MgmtVrfChain": 
         ip netns exec management iptables -t nat -N MgmtVrfChain

    C11: Link all incoming packets to the chain lookup: 
         ip netns exec management iptables -t nat -A PREROUTING -i eth0 -j MgmtVrfChain

    C12: Create DNAT rule to change destination IP to ip_address2 (ex: for SSH packets with destination port 22): 
         ip netns exec management iptables -t nat -A MgmtVrfChain -p tcp --dport 22 -j DNAT --to-destination 127.100.100.2   

These rules change the destination IP to ip_address2 before the packets are routed by management namespace. Management VRF routing instance routes these packets via the outport if1. Original destination IP is saved & tracked using the linux conntrack table for doing the appropriate reverse NAT for reply packets. 
When user wants to run any new application, a new rule with the appropriate dport should be added similar to the SSH dport 22 used in the above example C12. This solution of adding application specific dport rule is to restrict and accept only the incoming packets for the applications that are supported in SONiC.

**Step2:** 
After routing, use POST routing SNAT rule to change the source IP address to ip_address1 as given in the following example.

    C13: Add a post routing SNAT rule to change Source IP address:
         ip netns exec management iptables -t nat -A POSTROUTING -o if1 -j SNAT --to-source 127.100.100.1:62000-65000

This rule does source NAT for all packets that are routed through if1 and changes the source IP to ip_address1 and source port to a port number between 62000 and 65000. This source port translation is required to handle the usage of same source port number by two different external sources. Original source IP & port will be saved & tracked using the linux conntrack table for doing the appropriate reverse NAT for reply packets. After changing the source IP  & source port, packets are sent out of if1 and received in if2 by the default namespace. All packets are then routed using the default routing instance. These packets with destination IP ip_address2 are self destined packets and hence they will be handed over to the appropriate application deamons running in the default namespace.


### OUTGOING PACKET ROUTING

Packets that are originating from application deamons running in default namespace will be routed using the default routing table. Applications that need to operate on front panel ports work normally without any design change. Applications that need to operate on management namespace need the following design using DNAT & SNAT rules.

**Applications Spawned From Shell:**

Whenever user wants the applications like "Ping", "Traceroute", "apt-get", "ssh", "scp", etc., to run on management network, "ip netns exec management <actual command>" should be used.

    C14: Execute ping in management VRF
         ip netns exec management ping 10.16.208.58

This command will be executed in the management namespace (VRF) context and hence all packets will be routed using the management routing table and management interface. 

**Applications triggered internally:**

This section explains the flow for internal applications like DNS, TACACS, SNMP trap, that are used by the application daemons like SSH (uses TACACS), Ping (uses DNS), SNMPD (sends traps). Daemons use the internal POSIX APIs of internal applications to generate the packets. If such packets need to travel via the management namespace, user should configure "--use-mgmt-vrf" as part of the server  address configuration.
These application modules use the following DNAT & SNAT iptables rules to send the packets from default VRF context to the management VRF context and to route them through management namespace via management interface. Application specific design enhancement is explained below in the appropriate sections.

   1) Destination IP address of packet is changed to "ip_address1". This results in default VRF routing instance to send all the packets to veth pair, resulting in reaching management namespace. This is an example for tacacs that uses the port number 62000 for the tacacs server, which is explained in detail in tacacs implementation section.

    C15: Create DNAT rule for tacacs server IP address
         ip netns exec management iptables -t nat -A PREROUTING -i if1 -p tcp --dport 62000 -j DNAT --to-destination <actual_tacacs_server_ip>:<dport_of_tacacs_server>

   2) Destination port number of packet is changed to an internal port number. This will be used by management namespace for finding the appropriate DNAT rule in management VRF iptables that is requried to identify the actual destiation IP to which the packet has to be sent.

    C16: Create SNAT rule for source IP masquerade
         ip netns exec management iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
   
When Ping is executed using "ip netns exec management", the namespace context is implicit and hence no user configuration is required to specify the VRF. i.e. when user executes "ip netns exec management ping abcd.com", the ping application is spawned in the management namespace context. This application calls the DNS name resolution API in the management namespace context. DNS in turn uses the management namespace specific configuration file (/etc/netns/management/resolv.conf) if exists, or the default configuration file (/etc/resolv.conf) and sends the packet to the nameserver IP address using the management namespace routing table.
When the same ping is executed in default VRF context without "ip netns", same happens through the default namespace context.
Hence, no changes are required in DNS application.


## Implementation

Implementation of namespace based solution using Linux 4.9 kernel involves the following key design points.
1. Management VRF Creation
2. Applications/services to work on both management network and data network.

### Management VRF Creation
#### Initialization Sequence & Default Behavior
This section describes the default behavior and configuration of management VRF for static IP and dhcp scenarios. After upgrading to this management VRF supported SONiC image, the binary boots in normal mode with no management VRF created. Customers can either continue to operate in normal mode without any management VRF, or, they can run a config command to enable management VRF. 

    C17: config vrf add mgmt

This new management VRF specific command is chosen to keep the management vrf configuration similar of data vrf configuration. The alternate options for this CLI command is explained in Appendix. 

This command configures the tag "MGMT_VRF_CONFIG" in the ConfigDB (given below) and it restarts the "interfaces-config" service. The existing jinja template file "interfaces.j2" is enhanced to check this configuration and create the /etc/network/interfaces file with or without the "eth0" in the configuration. When management VRF is enabled, it does not add the "eth0" interface in this /etc/network/interfaces file. Instead, the service uses a new jinja template file "interfaces_mgmt.j2" and creates a new VRF specific configuration file /etc/network/interfaces.management.
This solution is based on the netns solution proposed at https://github.com/m0kct/debian-netns 
As specified in the solution, additional scripts are added, viz,  "/etc/network/if-pre-up.d/netns", "/etc/network/if-up.d/netns" and "/etc/network/if-down.d/netns. These scripts use the configuration files and follows the sequence of steps explained in the design section that takes care of the following.

    1. Creates the management namespace using command C1
    2. Attaches eth0 to the management namespace using command C2
    3. Configures IP address for eth0 in management namespace and adds the default route in management namespace using commands C3 & C4. This happens only when user had already configured the eth0 IP address and default gateway address using the MGMT_INTERFACE configuration. If this is not configured, it defaults to "dhcp".
    4. Creates the veth pair with two interfaces (if1 & if2) using commands C5, C6, C7
    5. Configures IP addresses for if1 and if2 using commands C8 & C9 
    6. Adds iptables DNAT & SNAT rules as given in C10, C11, C12, C13 & C16. 
    7. As part of DNAT rules, port numbers corresponding to the application deamons SSH, FTP, HTTP, HTTPS, SNMP, TFTP are added to accept packets from those applications. If any other application port number should be accepted in management interface, correponding DNAT rule should be added using the command C12 in linux shell.
    


#### Show Commands
Following show commands need to be implemented to display the VRF configuration.

| SONiC wrapper command             | Linux command                             | Description
|---                                |---                                        |---
| `show mgmt-vrf`                   | `ip netns show`                           | Display management VRF enabled status and linux namespaces
| `show mgmt-vrf interfaces `       | `ip netns exec management ifconfig'       | Displays the interfaces present in management VRF
| `show mgmt-vrf route`             | `ip netns exec management ip route show`  | Displays the routes present in management VRF routing table
| `show mgmt-vrf addresses`         | `ip netns exec management ip address show'| Displays all IP addresses of interfaces in management VRF

All these show commands are implemented using the linux commands directly without any rendering. This is more like a wrapper for linux commands. Output of these commands will be same as what linux displays.

### IP Application Design
This section explains the behavior of each application on the default VRF and management VRF. Application functionality differs based on whether the application is used to connect to the application daemons running in the device or the application is originating from the device.

#### Application Daemons In The Device
All application daemons run in the default namespace. All packets arriving in FPP are handled by default namespace.
All packets arriving in the management ports will be routed to the default VRF using the prerouting DNAT rule and post routing SNAT rule. Appropriate conntract entries will be created.
All reply packets from application daemons use the conntrack entries to do the reverse routing from default namespace to management namespace and then routing through the management port eth0.

#### Applications Originating From the Device
Applications originating from the device need to know the VRF in which it has to run. "ping", "traceroute","dhcclient", "apt-get", "curl" & "ssh" can be executed in management namespace using "ip netns exec management <command_to_execute>", hence these applications continue to work on both management and default VRF's without any change. Applications like TACACS & DNS are used by other applications using the POSIX APIs provided by them. Additional iptables rules need to be added (as explained in following sub-sections) to make them work through the management VRF. 


##### TACACS Implementation
TACACS is a library function that is used by applications like SSHD to authenticate the users. When users connect to the device using SSH and if the "aaa" authentication is configured to use the tacacs+, it is expected that device shall connect to the tacacs+ server via management VRF (or default VRF) and authenticate the user. TACACS implementation contains two sub-modules, viz, NSS and PAM. These module code is enhanced to support an additional parameter "--use-mgmt-vrf" while configuring the tacacs+ server IP address. When user specifies the --use-mgmt-vrf as part of "config tacacs add --use-mgmt-vrf <tacacs_server_ip>" command, this is passed as an additional parameter to the config_db's TACPLUS_SERVER tag. This additional parameter is read using the script files/image_config/hostcfgd. This script is enhanced to add/delete the following rules as and when the tacacs server IP address is added or deleted.

If the tacacs server is part of management network, following command should be executed to inform the tacacs module to use the management VRF.

    C18: Configure tacacs to use management VRF to connect to the server
         config tacacs add --use-mgmt-vrf <tacacs_server_ip>

As part of this enhancement, TACACS module maintains a pool of 10 internal port numbers 62000 to 62009 for configuring upto to 10 tacacs server in the device.
During initialization, module maintains this pool of 10 port numbers as "free" and it maintains the next available free port number for tacacs client to use.
It updates the tacacs configuration file /etc/pam.d/common-auth-sonic using the following configuration.

Ex: When user configures "config tacacs  add --use-mgmt-vrf 10.11.55.40", it fetches the next available free port (ex: 62000) and configures the destination IP for tacacs packet as "ip_address1" (ex: 127.100.100.1) with the next available free port (62000) as destination port as follows.

    auth    [success=done new_authtok_reqd=done default=ignore]     pam_tacplus.so server=127.100.100.1:62000 secret= login=pap timeout=5 try_first_pass

With this tacacs configuration, when user connects to the device using SSH, the tacacs application will generate an IP packet with destination IP as ip_address1 (127.100.100.1) and destination port as "dp1" (62000).
This packet is then routed in default namespace context, which results in sending this packet throught the veth pair to management namespace.
Such packets arriving in if1 will then be processed by management VRF (namespace). Using the PREROUTING rule specified below, DNAT will be applied to change the destination IP to the actual tacacs server IP address and the destination port to the actual tacacs server destination port number.

    C12: Create DNAT rule for tacacs server IP address
         ip netns exec management iptables -t nat -A PREROUTING -i if1 -p tcp --dport 62000 -j DNAT --to-destination <actual_tacacs_server_ip>:<dport_of_tacacs_server>
         Ex: ip netns exec management iptables -t nat -A PREROUTING -i if1 -p tcp --dport 62000 -j DNAT --to-destination 10.11.55.40:49

This packet will then be routed using the management routing table in management VRF through the management port.
When the packet egress out of eth0, POSTROUTING maseuerade rule will be applied to change the source IP address to the eth0's IP address.

    C13: Create SNAT rule for source IP masquerade
         ip netns exec management iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

With these rules, tacacs packet is then routed by the management namespace through the management interface eth0. While routing the packet, appropraite conntract entries are created by linux, which in turn will be used for doing the reverse NAT for the reply packets arriving from the tacacs server.
Following diagram explains the internal packet flow for the tacacs packets that are expected to be sent out of management interface.

![Outgoing_Packet_Flow](Management%20VRF%20Design%20Document%20NS%20Eth0%20Outgoing%20Pkt.svg "TACACS+ Outgoing Packets Control Flow") 

#### SNMP
The net-snmp daemon runs on the default namespace. SNMP request packets coming from FPP are directly handed over using default namespace. SNMP requests from management interfaces are routed to default namespace using the DNAT & SNAT (and conntrack entries for reply packets) similar to other applications like SSH.


#### SNMPTrap
SNMP traps originated from the device, the design similar to tacacs will be implemented to route them through management namespace.
SONiC uses netsnmp agent that runs as a docker service. The SNMP daemon generates traps and sends them to the configured trap server IP address in the /etc/snmp/snmpd.conf inside the docker.
Config commands support is not available to configure this trap server IP address/port. A new config command (given below) is added to configure the trap server IP address for sending v1 traps, v2 traps v3 traps. Support is provided to configure only one IP address for each SNMP version. If the trap server is reachable through the managment VRF, users should use "--use-mgmt-vrf" option. When user specifies the --use-mgmt-vrf as part of "config snmptrap modify --use-mgmt-vrf <snmp_version> <tacacs_server_ip>" command, this is saved as an additional parameter in the config_db's SNMP_TRAP_CONFIG tag and the snmp service is restarted. This additional parameter is read as part of the startup script /usr/bin/snmp.sh and the required DNAT rules are added to iptables similar to the tacacs solution explained above. 
If the trap server is part of management network, following command should be executed to inform the netsnmp module to use the management VRF.

    C18: Configure SNMP to use management VRF to send the traps to the server
         config snmptrap modify [--use-mgmt-vrf] <snmp_version> <snmptrap_server_ip> [snmptrap_server_port]
         Examples: 
         config snmptrap modify --use-mgmt-vrf 1 10.11.150.7 (this for sending SNMP version1 traps)
         config snmptrap modify --use-mgmt-vrf 2 10.11.150.7 (for SNMP v2c)
         config snmptrap modify --use-mgmt-vrf 3 10.11.150.7 (for SNMP v3)
          
By default, no trap server IP address is configured and hence no snmp traps will be sent by default. Users need to use the above commands to configure the SNMP trap server IP addresses. If users do not want to send snmptraps, they can revert back to default using the command "config snmptrap modify --use-mgmt-vrf 2 default (for SNMP v2c)".

As part of this enhancement, SNMP module uses three internal port numbers 62101 (for snmp version1), 62102 (for snmp v2c) and 62103 (for snmp v3). When user configures the snmptrap server IP address using --use-mgmt-vrf and if management VRF is already created, the startup script (/usr/bin/snmp.sh) shall add the DNAT rule similar to the tacacs solution explained earlier. It also updates the sonic snmp configuration file /etc/sonic/snmp.yml with the internal IP address 127.100.100.1 and internal port (62101 for v1, 62102 for v2c and 62103 for v3) in the variable applicable for the particular snmp version that user has configured. Sample configuration that is updated in snmp.yml is given below.

    snmp_rocommunity: public
    snmp_location: public
    v1_trap_dest: 20.21.22.23:162
    v2_trap_dest: default
    v3_trap_dest: 127.100.100.1:62103 

Startup script then starts the SNMP docker service which in turn uses the snmp internal startup script (file: /usr/bin/start.sh inside snmp docker) that use the snmpd.conf.j2 template file and generates the netsnmp configuration file /etc/snmp/snmpd.conf used by netsnmp.
Example snmpd.conf is given below.

    #   send SNMPv1  traps
    trapsink 20.21.22.23:162 public
    #   send SNMPv2c traps
    #trap2sink    localhost public
    #   send SNMPv2c INFORMs
    informsink 127.100.100.1:62103 public

In this example, v1 trap server is configured without "--use-mgmt-vrf" and hence the snmpd.conf does not use the internal IP address. It uses the actual IP address as it is. In case of v2, user has not configured and hence it uses the default value, which is why the "trap2sink" line in snmpd.conf is commented (dont send v2 traps). In case of v3, it is configured with "--use-mgmt-vrf" and hence the snmpd.conf uses the local IP 127.100.100.1 and local port 62103.
With this snmp configuration, when netsnmp sends the v3 traps, it will generate an IP packet with destination IP as ip_address1 (127.100.100.1) and destination port as "dp1" (62103).
This packet is then routed in default namespace context, which results in sending this packet throught the veth pair to management namespace.
Such packets arriving in if1 will then be processed by management VRF (namespace). Using the PREROUTING rule specified below, DNAT will be applied to change the destination IP to the actual trap server IP address and the destination port to the actual trap server destination port number.

    C12: Create DNAT rule for snmp trap server IP address
         ip netns exec management iptables -t nat -A PREROUTING -i if1 -p udp --dport 62103 -j DNAT --to-destination <actual_snmptrap_server_ip>:<dport_of_snmptrap_server>
         Ex: ip netns exec management iptables -t nat -A PREROUTING -i if1 -p udp --dport 62103 -j DNAT --to-destination 10.11.150.7:162

This packet will then be routed using the management routing table in management VRF through the management port.
When the packet egress out of eth0, POSTROUTING maseuerade rule will be applied to change the source IP address to the eth0's IP address.

    C13: Create SNAT rule for source IP masquerade
         ip netns exec management iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

With these rules, snmptrap packet is then routed by the management namespace through the management interface eth0. 


#### DHCP Client 
DHCP client is triggered internally as part of restarting the networking service. When management VRF is enabled and if the user has not configured a static IP address for the interface, the script "/etc/if-up.d/netns" takes care of executing "ip netns exec management ifup -i /etc/network/interfaces.management" that triggers the DHCP in the management namespace context. 

#### DHCP Relay 
DHCP relay is expected to work via the default VRF. DHCP Relay shall receive the DHCP requests from servers via the front-panel ports and it will send it to DHCP server through front-panel ports. No changes are reqiured.

#### DNS
DNS being a POSIX API library funntion, it is always executed in the context of the application that calls this.
DNS uses the common configuration file /etc/resolv.conf for all namespaces and it also has facility to have namespace specific configuration file. 
Whenever users wants DNS through management VRF (namespace), user should create the management namespace specific configuration file "/etc/netns/<namespace_name>]/resolv.conf" and configure the nameserver IP address in it.
When applications like Ping are triggered in the management namespace using "ip netns exec management <command>" (ex: "ip netns exec management ping google.com"), it uses the DNS POSIX API that is executed in the management namespace context which uses the configuration file "/etc/netns/management/resolv.conf" specific to the namespace. When namespace specific resolv.conf file does not exist, it uses the common configuration file /etc/resolv.conf.
Similarly when DHCP automatically fetches the domain-name-server IP address from DHCP server, it will udpate the appropriate resolv.conf file based on the context in which the DHCP client is executed.


#### Other Applications
Applications like "apt-get", "ntp", "scp", "sftp", "tftp", "wget" are expected to work via both default VRF & management VRF when users connect from external device to the respective deamons running in the device using the same DNAT & SNAT rules explained earlier.
When these applications are triggered from the device, use "ip netns exec management <command>" to run them in management VRF context. 


## Management VRF Configuration

This section explains the new set of configuration required for management VRF.
ConfigDB schema is enhanced to create a new tag "MGMT_VRF_CONFIG" where the management VRF specific configuration parameters are stored.

### Mangement VRF ConfigDB Schema
The upgraded config_db.json schema to store the flag for enabling/disabling management VRF is as follows.

```
"MGMT_VRF_CONFIG": {
    "vrf_global": {
        "mgmtVrfEnabled": "false" 
    }
}
```
Default value for "mgmtVrfEnabled" is "false".
Users shall enable the management VRF by setting the "mgmtVrfEnabled" to "true". 
The vrfname for the management VRF is fixed as "mgmt". Users cannot modify this vrfname.

### Management VRF Config Commands
Following config command are provided to enable or disable the management VRF and to configure the management vrfname.

```
   config vrf add mgmt
   config vrf del mgmt
```   
A new module "vrf configuration manager" is added to listen for the configuration changes that happen in ConfigDB for "MGMT_VRF_CONFIG".
Its implemented as part of the python script /usr/bin/vrfcfgd.
This subscribes for the MGMT_VRF_CONFIG with the ConfigDB and listens for ConfigDB events that are triggered as and when the configuration is modified. 

### Confguring Management Interface Eth0

There is no config command available to configure the parameters related to MGMT_INTERFACE present in ConfigDB. Minigraph.xml is the alternate way using which the MGMT_INTERFACE parameters can be configured.
This is enhanced to provide the following config commands to configure the eth0 IP address and the associated default route. If IP address have to be obtained via DHCP, this static IP address configuration is not required. This configuration is independent of management VRF configuration.

#### MGMT_INTERFACE ConfigDB Schema
The  existing config_db.json schema for the MGMT_INTERFACE related attributes shown below is reused without any change.

```
"MGMT_INTERFACE": {
    "eth0|ip_address/prefix": {
        "forced_mgmt_routes": {
        },
        "gwaddr": "<gw_ip_addr>" 
    }
}
```
#### MGMT_INTERFACE Config Commands
To configure the management interface IP and the associated default gateway address, following new config commands are added.

```
   config managementip add <ip_address/prefix> <default_gw_addr>
   config managementip del <ip_address/prefix>
```   
This configuration is independent of the management VRF configuration. 

#### SNMP_TRAP_CONFIG ConfigDB Schema
The  newly added config_db.json schema for the SNMP_TRAP_CONFIG related attributes is shown below.

```
    "SNMP_TRAP_CONFIG": {
        "v1TrapDest": {
            "DestIp": "10.11.150.7",
            "DestPort": "162",
        },
        "v2TrapDest": {
            "DestIp": "default",
            "DestPort": "162"
        },
        "v3TrapDest": {
            "DestIp": "10.11.150.7",
            "DestPort": "162"
            "vrf": "mgmt"
        }
    }
```

### Configuration Sequence
When user upgrades to the new SONiC software with management VRF support, the images comes up as usual without management VRF enabled. Users shall follow the following steps to enable & use the management VRF.

**Step1**
If static IP address is required for the management interface eth0, users shall first configure the MGMT_INTERFACE attributes using the commands given above. If DHCP is required, this configuration can be skipped.
Example: config managementip add 10.16.206.92/24 10.16.206.1

**Step2**
Enable management vrf.
Example: config vrf add mgmt
When management vrf is enabled, the VRF configuration manager "vrfcfgd" module listens for this configuration changes and does the following.

1. If the management vrf is enabled from the default disabled state, it restarts the "config-interfaces" service.
2. If the management vrf is disabled from the previous enabled state, it first deletes the previously created management namespace and then it restarts the "interfaces-config" service.

"interfaces-config" service is enhanced to create the required debian configuration files "/etc/network/interfaces" and '/etc/network/interfaces.management" using the jinja templates. It then restarts the networking service that that brings up the interfaces in appropriate namespaces. The newly added scripts "/etc/network/if-pre-up.d/netns" and "/etc/netns/if-up.d/netns" creates the management namespace using linux commands, moves the eth0 interface to management namespace, creates the veth pair and configures the internal interfaces if1 & if2 with internal IP addresses. "interfaces-config" uses additional script to add the requried iptables rules for SNAT & DNAT. When management VRF is disabled, all the processes running in the management VRF are killed, management namespace is deleted and the configurations done earlier are reverted back. Linux takes care of moving eth0 back to default VRF.


## Appendix

### Alternate Solution Options for Bootup Sequence
There are multiple ways in which the management namespace can be created in SONiC. The chosen solution explained in Design section is based on the solution provided at https://github.com/m0kct/debian-netns 
Following alternate solutions were also considered and dropped due to the reasons given below.

**Option1: Use only /etc/network/interfaces**
Understand the current SONiC (Debian) control flow on how the restarting of networking service uses the /etc/network/interfaces and find out how the IP address is configured for the interfaces and how the ifup/ifdown is being called. Find out where the linux commands are called/used and change them to use  “ip netns exec management” in such places based on the namespace to which the interface belongs to. For example, use "ip netns exec" for eth0 specific operations and dont use the same for interfaces that belong to default VRF. Linux configuration always happens in the default VRF context using the debian package “ifupdown”; this code flow will result in modifying this debian package. Instead, since there is an alternate solution without changing debian package, this option is dropped.

**Option2: Avoid the usage of /etc/network/interfaces**
This option is to use “ip netns exec” linux calls instead of following the networking service restart flow. i.e. Modify the existing SONiC control flow and avoid the usage of /etc/network/interfaces and "ifupdown" package to configure the interfaces. Solution is to modify the bootup sequence script “interfaces_config.sh” and make the direct linux shell commands to configure the management VRF and the required DNAT & SNAT rules. Comment out the "systemctl restart networking" so that the control flow that uses the /etc/network/interfaces is avoided. This means that  the existing SONiC solution that is based on /etc/network/interfacecs and networking restart is not used. 

**Disadvantages:** 

1. Current SONiC solution that uses the /etc/network/interfaces and networking restart will be deviating, which is not required in other options
2. If user explicitly executes the "systemctl restart networking" command directly from linux shell, the  VRF configuration will be lost. This needs further investigation on solving all such cases. 

Due to the reasons given above, the solution given at https://github.com/m0kct/debian-netns is chosen and explained earlier in design section.

### Alternate Implementation Options
Some of the implementation choices that are considered and dropped are explained here.

As alternate to the command "config vrf add/del mgmt", it is also possible to enhance the command that is being planned for data vrf implementation https://github.com/Azure/SONiC/pull/242 as follows. 
    
    C17:Option-a: config mgmt-vrf enable/disable
        Examples: "config vrf add mgmt", "config vrf add management"
        This option is dropped in order to use the same format as data VRF configuration that uses a different command 
        syntax given in following options.
    
    C17:Option-b: config vrf add/del <vrfname>. 
        This command syntax is common for both management VRF and data VRF creation/deletion.
        In this option, the fixed VRF-name “mgmt” (or "management") will be treated as management VRF and any VRF-name not 
        matching "mgmt" (and "management") will be treated as data VRF-name.
        If the vrfname is either "management" or "mgmt", it will enable/disable the management VRF.  
        Any name other than "mgmt" or "management" will be treated as data VRF.
        i.e. The vrfname for management VRF is fixed as "mgmt" internally and the CLI users can use the 
        vrfname as either "management" or "mgmt".
        This option looks better than the other options.
       
    C17:Option-c: config vrf add/del [management] <vrfname> OR config vrf add/del [mgmt] <vrfname>. 
        Examples: "config vrf add management blue", "config vrf add management management", "config vrf add management mgmt"
        In this option, the optional keyword "management" or "mgmt" is used to specify that the VRF specified with "vrfname" is
        a management VRF (not a data VRF). The issue in this is that users are free to specify any vrfname as parameter. 
        In case if users give similar name as data VRF, it will confuse the users.
        In the example given, the vrfname "blue" will be treated as management VRF even though the vrfname looks like a data vrfname. 
        In the other example given, the name "management" is repeated twice (first is the optional keyword and 2nd is the 
        vrfname specified by user), which is also confusing.
              
    C17:Option-d: config vrf add/del [--mgmt-vrf] <vrfname>
        Examples: "config vrf add --mgmt-vrf red", "config vrf add --mgmt-vrf management", "config vrf add --mgmt-vrf mgmt"
        This is a slight variation of previous option. Instead of the keyword management/mgmt, this uses "--mgmt-vrf" as optional 
        parameter to specify that the VRF specified with "vrfname" is a management VRF (not a data VRF). The issue in this is 
        that users are free to specify any vrfname as parameter. 
        In case if users give similar name as data VRF, it will confuse the users.
        In the example given, the vrfname "red" will be treated as management VRF even though the vrfname looks like a data vrfname.

In all these options, its only one CLI command that will create management namespace and add eth0 to it internally. It is not required to use any other commands to attach eth0 to the management VRF.

**CLI Conclusion**: C17: Option-b is chosen so that command is in sync with the data vrf command.

### ACL rules design

ACL rules that are currently installed in SONiC are added based on src-ip. Some of the rules needs to be added in the management namespace, below there are 2 design options suggested. Once design is frozen implementation details will be updated.

    a. ACL manager to be enhanced for management VRF design. During management VRF enable/disable events ACL manager will
       install/duplicate the rules from default VRF(NS) to management VRF(NS). Any update/delete to the iptables will be updated
       accordingly in the management VRF. When management VRF is disabled all rules will be deleted automatically no action is required 
       by ACL manager.
  
       Advantages: All ACL rules are added/deleted/modified at one place by ACL manager and that take cares of adding/deleting/modifying 
       the rules for management VRF as well.
       
       Disadvantages: ACL manager is currently independent of management VRF, by changing the desing it becomes dependent on managment 
       VRF events.
       
    b. Adding a new iptables module to listen to iptables events from kernel and handle the management VRF namespace enable/disable 
       events and add and delete ip tables in management VRF.
       
       Advantages: This module is specific to management VRF and will add ACL rules in management VRF chain.
       
       Disadvantages: Needs one more module for managing management VRF iptables events. 

### Show command outputs

    root@sonic:~# show mgmt-vrf

    ManagementVRF : Enabled

    NameSpaces in Linux:
    mgmt (id: 0)
    root@sonic:~# show mgmt-vrf interfaces

    Interfaces in Management VRF:
    eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
            inet 10.16.206.15  netmask 255.255.255.0  broadcast 10.16.206.255
            inet6 fe80::4e76:25ff:fef4:f9f3  prefixlen 64  scopeid 0x20<link>
            ether 4c:76:25:f4:f9:f3  txqueuelen 1000  (Ethernet)
            RX packets 298921  bytes 64239043 (61.2 MiB)
            RX errors 0  dropped 19685  overruns 0  frame 0
            TX packets 7816  bytes 673467 (657.6 KiB)
            TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
            device memory 0xdff40000-dff5ffff

    if1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
            inet 127.100.100.1  netmask 255.255.255.0  broadcast 127.100.100.255
            inet6 fe80::48f0:27ff:fef7:1e8  prefixlen 64  scopeid 0x20<link>
            ether 4a:f0:27:f7:01:e8  txqueuelen 1000  (Ethernet)
            RX packets 4  bytes 520 (520.0 B)
            RX errors 0  dropped 0  overruns 0  frame 0
            TX packets 8  bytes 648 (648.0 B)
            TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

    lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
            inet 127.0.0.1  netmask 255.255.255.0
            inet6 ::1  prefixlen 128  scopeid 0x10<host>
            loop  txqueuelen 1  (Local Loopback)
            RX packets 1  bytes 28 (28.0 B)
            RX errors 0  dropped 0  overruns 0  frame 0
            TX packets 1  bytes 28 (28.0 B)
            TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
            
    root@sonic:~# show mgmt-vrf route

    Routes in Management VRF Routing Table:
    10.16.206.0/24 dev eth0 proto kernel scope link src 10.16.206.15
    127.100.100.0/24 dev if1 proto kernel scope link src 127.100.100.1

    root@sonic:~# show mgmt-vrf addresses

    IP Addresses for interfaces in Management VRF:
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/24 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host
           valid_lft forever preferred_lft forever
    2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
        link/ether 4c:76:25:f4:f9:f3 brd ff:ff:ff:ff:ff:ff
        inet 10.16.206.15/24 brd 10.16.206.255 scope global eth0
           valid_lft forever preferred_lft forever
        inet6 fe80::4e76:25ff:fef4:f9f3/64 scope link
           valid_lft forever preferred_lft forever
    78: if1@if79: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
        link/ether 4a:f0:27:f7:01:e8 brd ff:ff:ff:ff:ff:ff link-netnsid 0
        inet 127.100.100.1/24 brd 127.100.100.255 scope host if1
           valid_lft forever preferred_lft forever
        inet6 fe80::48f0:27ff:fef7:1e8/64 scope link
           valid_lft forever preferred_lft forever

### Design Review Minutes of Meeting December 6th 2018
Attendees: Guohan, Joe, Marcin, Harish, Kannan, Anand

1. ACL rules - Existing SONiC ACL infra adds ACL ip rules in the default VRF. There are specific source ip based rules which are applicable to management. How is this addressed in the design? Dell will analyze and change the design accordingly to address ACL rules. Different design options for ACL design is mentioned in ACL rules design in Appendix section.

2. Show commands - The SONiC wrapper show commands displays the output of linux commands. Show command outputs are mentioned above in Appendix section. Dell to work with MSFT to identify what needs to be customized for SONiC show wrapper command output.

3. Config command to enable/disable VRF - Various CLI options that were discussed during the Design Review were given in the above section. MSFT to review the CLI options and suggest which one to implement. Dell to update design document accordingly.
    
4. Sync up with data VRF team for command schema for management and data VRF since we are going to use command command. Anand to setup meeting with Prince and dell team members.

5. Managementip config command - combine with hostcfgd to listen to configuration change and take action. hostcfgd is now dependent on mgmt enable/disable. Dell to update design docuement.

6. L3mdev - alternate solution https://lwn.net/Articles/670190/ - Dell to followup with nikos regarding this.

