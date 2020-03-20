## GN3 Institutional Network Setup Guide with ISP Connectivity
**March 19, 2020**

This guide will walk you through setting up your own institutional network using GNS3, a network emulator software application.

### Install and setup GNS3
Download GNS3 from [here](https://www.gns3.com/software/download). Install on localhost and walk through the setup wizard.

If you choose to install on a virtual machine, you will need to install and import the GNS3 VM also.

You will start off with application interfaces (VPCS) and some switches (layer 2 devices). You will also need to manually create templates for routers (layer 3 devices) in order to set up an network that can talk to machines outside your local network. The Cisco 7200 is a good router to use.

#### Manually set up Cisco 7200 IOS image
1. Download and unzip Cisco IOS images from [here](https://mega.nz/#F!nJR3BTjJ!N5wZsncqDkdKyFQLELU1wQ).
2. Press "+ New Template" on the router interfaces toolbar.
3. Select "Manually create a template" and navigate to "IOS routers" under "Dynamips".
3. Press "New", select "Run this IOS router on my local computer" (or GNS3 VM), and import the Cisco c7200 image.
4. Once imported, go to Slots and make sure slot 0 is set to "C7200-IO-FE" and slot 1 is set to "PA-FE-TX". These are FastEthernet.
5. Under Optimisations in Advanced, set Idle-PC to 0x6050b114.

### Simple LAN setup
1. Place two VPCS interfaces and an Ethernet switch. They may be set up with default names (PC1, PC2, Switch1) and settings.
2. Draw a link between each VCPS interface's Ethernet0 to Switch1 (Ethernet0 and Ethernet1).
3. Start each VCPS service by right-clicking them and selecting "Start". Properly started services will show green circles on both sides of the data links between two devices.

### Router and ISP setup
1. Place two c7200 routers R1 and R2 and a VCPS PC3.
2. Draw a link from Switch1's Ethernet2 to R1's FastEthernet0/0, a link from R1's FastEthernet1/0 to R2's FastEthernet0/0, and a link from R2's FastEthernet1/0 to VCPS PC3's Ethernet0. Let's assume PC3 is a server running on the ISP, and can be accessed by our LAN through R2 (where R1 is the router on our LAN).
3. Start the routers and PC3 by right-clicking them and selecting "Start".

### LAN Router IP configuration
To assign the FastEthernet0/0 interface of router R1 the IP address and mask 10.1.1.1/24, follow these steps:
1. Right-click R1 and select "Console".
2. Run `config t` to enter configure mode.
3. Run `interface f0/0` to select the FastEthernet0/0 interface.
4. Run `ip addr 10.1.1.1 255.255.255.0` to assign the IP address 10.1.1.1 and the subnet mask /24.
5. Run `no shut` to bring the interface up, then `exit` to exit the interface and config mode.
6. On the default shell, you can run `show ip interface brief` to verify the IP addresses are configured properly.

### LAN End device IP configuration
A router R1 with the configuration 10.1.1.1/24 may have two devices configured with the addresses 10.1.1.2 and 10.1.1.3 with gateways to R1's FastEthernet0/0.
1. Right-click PC1 and select "Console".
2. Run `ip 10.1.1.2/24` and verify with `show ip`. This will assign the device the IP address 10.1.1.2.
3. Run `ip 10.1.1.2/24 10.1.1.1` and verify with `show ip`. This will create a gateway to R1's FastEthernet0/0 interface.
4. Do the same for PC2 with 10.1.1.3 instead.

### More complex LANs
To set up more complex network topologies, we can add more application interfaces to communicate with Switch1 (however many Switch1 can support), and add additional switches as needed to support a larger number of interfaces. This may require us to configure different subnet masks to support a larger subnet.

### LAN to WAN configuration
We have set up a single interface on our router R1 that is pingable by the LAN's end devices, but we still need to set up the ISP and configure the LAN router so that it can be reached by other networks.
1. Assign the static IP address and mask 192.1.1.1/24 on FastEthernet1/0 of router R1 (our LAN router). Follow the steps in the LAN Router IP configuration section above. (In an actual network, DHCP will generally assign dynamic IP addresses. For the sake of simplicity, we will just manually assign a static IP address and subnet mask.)
2. Assign the static IP address and mask 192.1.1.2/24 on FastEthernet0/0 of router R2 (our ISP router).
3. Assign the static IP address and mask 10.2.1.1/24 on FastEthernet1/0 of router R2 (our ISP router).
4. Assign the IP address and mask 10.2.1.2/24 on Ethernet0 of PC3, the server on the ISP. Follow the steps in the LAN End device IP configuration setup above.
5. To ping the ISP router from our LAN end devices, we need to setup the routers' default routes to the other router. Run `ip route 0.0.0.0 0.0.0.0 192.1.1.2` on R1's console and `ip route 0.0.0.0 0.0.0.0 192.1.1.1` on R2's console.

### Setup NAT table inside the access network
1. Run `interface f0/0` and `ip nat inside` to set R1's FastEthernet0/0 interface as a NAT inside.
2. Run `interface f1/0` and `ip nat outside` to set R1's FastEthernet1/0 interface as a NAT outside.
3. Exit the interface with `exit`.
4. Create the range of addresses inside to be translated to the FastEthernet1/0 interface's address by typing `access-list 10 permit 10.1.1.0 0.0.0.255` and `ip nat inside source list 10 interface f1/0 overload`. Verify with `debug ip nat`.
