### Solved by Swappage

for2 was a forensics challenge were we had a pcap file and our objective was to extract the flag..

Opening the file in wireshark revealed that it's not network traffic, but USB.

i looked at the devices and IDs

    $ tshark -r capture.pcapng usb.bDescriptorType
     83   6.505211         host -> 1.3.0        USB 36 GET DESCRIPTOR Request DEVICE
     84   6.505211        1.3.0 -> host         USB 46 GET DESCRIPTOR Response DEVICE
     86   6.505211         host -> 1.3.0        USB 36 GET DESCRIPTOR Request CONFIGURATION
     87   6.520811        1.3.0 -> host         USB 37 GET DESCRIPTOR Response CONFIGURATION
     89   6.520811         host -> 1.3.0        USB 36 GET DESCRIPTOR Request CONFIGURATION
     90   6.520811        1.3.0 -> host         USB 62 GET DESCRIPTOR Response CONFIGURATION

     $ tshark -r capture.pcapng -2 -R 'usb.bDescriptorType' -T fields -e usb.bus_id -e usb.device_address -e usb.idVendor -e usb.idProduct
     1	3		
     1	3	1133	0x0000c05a
     1	3		
     1	3		
     1	3		
     1	3		

whreshark tells us that 0xc05a is the product ID for a logitech m90/m100 optical mouse.

i had a deja vu, this challenge reminded me of something

I started extracting the payload data from the mouse movements, saving in a file all the coordinates where the mouse button was clicked.

    $ tshark -r capture.pcapng -2 -R 'usb.capdata && usb.device_address == 3' -Tfields -e usb.capdata | head
    00:01:fe:00
    00:01:ff:00
    00:02:00:00
    00:03:00:00
    00:01:00:00
    00:02:00:00
    00:04:ff:00
    00:01:ff:00
    00:03:ff:00
    00:03:fd:00


    $ awk -F: 'function comp(v){if(v>127)v-=256;return v}{x+=comp(strtonum("0x"$2));y+=comp(strtonum("0x"$3))}$1=="01"{print x,y}' mouse_data.txt | tee click_coordinates.txt | head
    -273 -428
    -889 -242
    -890 -241
    -891 -241
    -892 -241
    -893 -241
    -894 -241
    -897 -241
    -898 -241
    -899 -241

with all the coordinates where the mouse *clicked* i tried to use gnuplot to visualize the coordinates as an image

the result was the following:

![](/images/2016/google-ctf/for2/flag.png)
