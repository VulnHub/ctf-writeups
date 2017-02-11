AlexCTF forensics3
---

### Solved by barrebas

Forenics3 involves a USB pcap. We're asked to extract a secret file. 

Running the pcap through ```xxd``` and perusing the resulting dump shows a PNG header. We're looking for a PNG!

Now we need to extract this file. This is easily done using wireshark. Frame 101 contains the PNG (the length is 61504 bytes). Selecting that packet, highlighting the line that says "Leftover Capture Data" and right-clicking on that line, select "Export selected packet bytes" allows us to export to a file. 


The resulting PNG looks like this:

![](/images/2017/codegate/alexctf/for3/x.png)

The flag therefore is ```ALEXCTF{SN1FF_TH3_FL4G_OV3R_USB}```
