###Solved by bitvijays

***Challenge 4: Network Forensics: 200 Points***: You are given a cap file that contains wireless traffics in a location. Find the flag! :-)

Running tshark on the pcap file shows the wireless traffic present in the pcap file.
```
  1   0.000000 86:2f:82:10:f3:d0 -> Tp-LinkT_ff:0f:48 802.11 24 Null function (No data), SN=266, FN=0, Flags=...P...T
  2  -0.000017              -> 86:2f:82:10:f3:d0 (RA) 802.11 10 Acknowledgement, Flags=........
  3   0.066544 Tp-LinkT_ff:0f:48 -> Broadcast    802.11 94 Beacon frame, SN=1922, FN=0, Flags=........, BI=100, SSID=Rome
  4   0.069147 SamsungE_32:46:60 -> Broadcast    802.11 160 Beacon frame, SN=217, FN=0, Flags=........, BI=100, SSID=SD
  5   0.076309 86:2f:82:10:f3:d0 -> Broadcast    802.11 146 Probe Request, SN=270, FN=0, Flags=........, SSID=Rome
  6   0.096769 86:2f:82:10:f3:d0 -> Tp-LinkT_ff:0f:48 802.11 24 Null function (No data), SN=271, FN=0, Flags=.......T
  7   0.096752              -> 86:2f:82:10:f3:d0 (RA) 802.11 10 Acknowledgement, Flags=........
  8   0.096752 Tp-LinkT_ff:0f:48 -> 86:2f:82:10:f3:d0 802.11 24 Null function (No data), SN=0, FN=0, Flags=......F.
  9   0.096792              -> Tp-LinkT_ff:0f:48 (RA) 802.11 10 Acknowledgement, Flags=........
 10   0.126496              -> Tp-LinkT_ff:0f:48 (RA) 802.11 10 Acknowledgement, Flags=........
 11   0.332313 86:2f:82:10:f3:d0 -> Tp-LinkT_ff:0f:48 802.11 24 Null function (No data), SN=272, FN=0, Flags=...P...T
 12   0.332784              -> 86:2f:82:10:f3:d0 (RA) 802.11 10 Acknowledgement, Flags=........
 13   0.953440 86:2f:82:10:f3:d0 -> Tp-LinkT_ff:0f:48 802.11 136 Data, SN=273, FN=0, Flags=.p.....T
 14   0.955456 86:2f:82:10:f3:d0 -> Tp-LinkT_ff:0f:48 802.11 24 Null function (No data), SN=274, FN=0, Flags=.......T
 15   0.955440              -> 86:2f:82:10:f3:d0 (RA) 802.11 10 Acknowledgement, Flags=........
 16   0.955442 Tp-LinkT_ff:0f:48 -> 86:2f:82:10:f3:d0 802.11 24 Null function (No data), SN=0, FN=0, Flags=......F.
 17   0.955456              -> Tp-LinkT_ff:0f:48 (RA) 802.11 10 Acknowledgement, Flags=........
 18   1.133154              -> Tp-LinkT_ff:0f:48 (RA) 802.11 10 Acknowledgement, Flags=........
 19   1.261666 86:2f:82:10:f3:d0 -> Tp-LinkT_ff:0f:48 802.11 112 Data, SN=275, FN=0, Flags=.p.....T
 20   1.290850              -> Tp-LinkT_ff:0f:48 (RA) 802.11 10 Acknowledgement, Flags=........
 21   1.291873 86:2f:82:10:f3:d0 -> Tp-LinkT_ff:0f:48 802.11 92 Data, SN=276, FN=0, Flags=.p.....T
***SNIP***
```
Checking with the tcp filter we find that file named rom-0 is requested via a GET command
```
tshark -r Net_Forensic2.cap -Y tcp
1274  44.347189 192.168.43.61 -> 46.4.232.88  TCP 92 49100→80 [SYN] Seq=0 Win=29200 Len=0 MSS=1460 SACK_PERM=1 TSval=913007 TSecr=0 WS=1024
1280  44.726105  46.4.232.88 -> 192.168.43.61 TCP 84 80→49100 [SYN, ACK] Seq=0 Ack=1 Win=14600 Len=0 MSS=1460 SACK_PERM=1 WS=128
1282  44.726645 192.168.43.61 -> 46.4.232.88  TCP 72 49100→80 [ACK] Seq=1 Ack=1 Win=29696 Len=0
1284  44.726645 192.168.43.61 -> 46.4.232.88  HTTP 186 GET /rom-0 HTTP/1.1 
1287  44.793688  46.4.232.88 -> 192.168.43.61 TCP 72 80→49100 [ACK] Seq=1 Ack=115 Win=14720 Len=0
1298  45.092694  46.4.232.88 -> 192.168.43.61 TCP 349 [TCP segment of a reassembled PDU]
1299  45.092694  46.4.232.88 -> 192.168.43.61 TCP 1532 [TCP segment of a reassembled PDU]
1301  45.092725 192.168.43.61 -> 46.4.232.88  TCP 72 49100→80 [ACK] Seq=115 Ack=278 Win=30720 Len=0
1303  45.092725 192.168.43.61 -> 46.4.232.88  TCP 72 49100→80 [ACK] Seq=115 Ack=1738 Win=33792 Len=0
1304  45.099862  46.4.232.88 -> 192.168.43.61 TCP 1532 [TCP Previous segment not captured] [TCP segment of a reassembled PDU]
1305  45.099862  46.4.232.88 -> 192.168.43.61 TCP 964 [TCP Previous segment not captured] [TCP segment of a reassembled PDU]
1306  45.099862  46.4.232.88 -> 192.168.43.61 TCP 1532 [TCP Out-Of-Order] [TCP segment of a reassembled
***SNIP**
```

Extracting this rom0 file by using Export http objects and analyzing it provide us with
```
file rom-0 
rom-0: PDP-11 UNIX/RT ldp
```

Searching google.com for "Decrypt Rom-0" brings you to <a href="http://www.routerpwn.com/">routerpwn</a> where you can upload rom-0 file and decompress it as it is a Rom-0 Configuration Decompressor (LZS).

This provides the sp.data contents which contains the WPA password for the wireshark wireless capture
```
Rome4040
TP-LINK
public
public
public
public
```

Providing wireshark with "Rome4040" in the Edit->Preferences->Protocols->IEEE 802.11 -> Decryption Keys -> New -> WPA-PWD and applying it would decrypt the WiFi session.

Going thru the pcap file, we would find lot of GET requests and one POST request. Let's have a closer look at the POST
```
POST /post.php HTTP/1.1
Host: pastebin.com
Connection: keep-alive
Content-Length: 860
Cache-Control: max-age=0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Origin: http://pastebin.com
User-Agent: Mozilla/5.0 (Linux; Android 4.4.2; en-us; SAMSUNG SM-N900 Build/KOT49H) AppleWebKit/537.36 (KHTML, like Gecko) Version/1.5 Chrome/28.0.1500.94 Mobile Safari/537.36
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary5tXKlEDQjaGL5UAN
Referer: http://pastebin.com/
Accept-Encoding: gzip,deflate,sdch
Cookie: __cfduid=ddca1c88fbf427b98903aa6f2e36d65b11454153405; PHPSESSID=oa8ki1i9o7nllhs4an36sfsh54; _ga=GA1.2.899094573.1454153414
Accept-Language: en-US,en;q=0.8

------WebKitFormBoundary5tXKlEDQjaGL5UAN
Content-Disposition: form-data; name="csrf_token_post"
MTQ1NDE1ODE1MGNMdnA1bE1hR3ZoTnR1Ym9xbXd3Vk9xcW5qN0huTXpx
------WebKitFormBoundary5tXKlEDQjaGL5UAN
Content-Disposition: form-data; name="submit_hidden"
submit_hidden
------WebKitFormBoundary5tXKlEDQjaGL5UAN
Content-Disposition: form-data; name="paste_code"
SharifCTF{be02d2a396482969e39d92b6e440f5e3}
------WebKitFormBoundary5tXKlEDQjaGL5UAN
Content-Disposition: form-data; name="paste_format"
1
Content-Disposition: form-data; name="paste_expire_date"
***SNIP***
```
The flag is ***SharifCTF{be02d2a396482969e39d92b6e440f5e3}***
