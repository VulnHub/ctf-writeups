### Solved by bitvijays

In the excercise, we have to find the attacker's last name and the two IP address he has spoofed. 
>The police have retrieved a network trace of some suspicious activity. Most of the traffic is users viewing their own profiles on a social networking website, but one of the users on the network downloaded a file from the Thyrin Labs VPN and spoofed their IP address in order to hide their identity. Can you figure out the last name of person that accessed the Thyrin files, and the two source IP addresses they used?
[Example valid flag format: "davis,192.168.50.6,192.168.50.7"]

You could either download the PCAP file or view it on cloudshark. We downloaded the PCAP file and as it is mentioned in the challenged, Attacker spoofed their IP address, the MAC address should be the same. If we look for the ARP (Address Resolution Protocol) Packets, display filter "arp", we can easily see that MAC address: 08:00:27:2b:f7:02 sent Who has 192.168.50.10 with two IP Address 192.168.50.3, 192.168.50.4. So, we got the two IP address which were spoofed.

![](/images/2014/pico/spoof_proof/trafficARP.png)

Since the IP of the attacker before spoofing was 192.168.50.3, if we use display filter "ip.addr==192.168.50.3", We can easily see the name of the user which is john johnson.

![](/images/2014/pico/spoof_proof/trafficIP.png)

Combining all the information such as last name and two spoofed IP address in the valid flag format, we get the flag.

The flag is **johnson,192.168.50.3,192.168.50.4**.
