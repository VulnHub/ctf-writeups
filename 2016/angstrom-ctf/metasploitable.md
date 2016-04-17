### Solved by z0rex

metasploitable was a forensics challenge worth 120 points.

> Our Wordpress blog has been hacked! Fortunately, the network capture from our
> intrusion detection system may provide some clues. Can you help us figure out
> what the hacker did? 

I downloaded the attached file `owned.pcap` and had a quick look at it in wireshark.

Here I could quickly see that there was a zip file being transfered in the TCP
stream, so I used `foremost` to extract it.

```
# foremost -v -i owned.pcap 
Foremost version 1.5.7 by Jesse Kornblum, Kris Kendall, and Nick Mikus
Audit File

Foremost started at Mon Apr 11 20:03:41 2016
Invocation: foremost -v -i owned.pcap 
Output directory: /opt/challenges/ctf/angstrom/forensics/metasploitable/output
Configuration file: /etc/foremost.conf
Processing: owned.pcap
|------------------------------------------------------------------
File: owned.pcap
Start: Mon Apr 11 20:03:41 2016
Length: 101 KB (103452 bytes)
 
Num  Name (bs=512)         Size  File Offset     Comment 

foundat=uwuXqhEVYK/style.cssPK
�/W�)����Ή�-,p�3�g���F޿����A4x�ǎg,��}��49�H�^�Oc�����9��F��l�(A,P.%ר@�\������3
�w��!<8=�j� ���K��J�ݡV���׹����J���&�k73���;�Ǳ�j��,�$�l�(ax���Xk?���*�XU��B¶~<E=^�:k���D�>�}��Ш�D�7�e�V����¨���
                                                                                                                       �8,����[���)}�\�}K&ӟ����d�L�~XE�>�IE]�<Շ�֌�=�&r�9a
�6>����+���aB=olb�cr'@�n��5WH�����O����g��������ve;\�ʘ��.�?PKW@Ɨ�޼ʎ���2)u��{����.x�^�������صO#��ގ��H)�eI �]S)��N`����zOcJ!�M�4���m#�=                       �ǎ-�Ʊ�`p~-�-A1�3����Z�
0:  00000002.zip           4 KB            1179      
*|
Finish: Mon Apr 11 20:03:41 2016

1 FILES EXTRACTED
    
zip:= 1
------------------------------------------------------------------

Foremost finished at Mon Apr 11 20:03:41 2016
```

As we can see here, one zip file were extracted successfully.

Extracting the zip then reveals two files; `style.css` and `biZGmbfxpr.php`

The css file were empty, but `biZGmbfxpr.php` contained a PHP reverse shell. This
lead me to believe that there was more to be found in wireshark. I picked a stream
and followed it, and hit a match right away on stream 2.

**Wireshark filter**

```
tcp.stream eq 2
```

There were lots and lots of PHP code, but in frame 152 I found this

```php
<?php

$o3cef481="\x62\141\x73\145\66\x34\x5f\144\x65\x63\x6f\x64\x65";@eval($o3cef481(
"Ly9OR05ObS8vMktlZW9PR29XVWM0SEZRSUVTTlRmWVVORUdwcnVmNkhWenpsbklDaG5SUmV5Wkg5R3k
xMmw5U1RvMC9OdHA1dD06N3EyczEzMG8KJG8zY2VmNDgxPSJceDYyIjskYWMxMzkzYTA9Ilx4NzMiOyR
iYzFmNzMwYj0iXDE2MCI7JHVjZTVlYzZiPSJceDY3IjskZTE3ZjEyYmQ9IlwxNjIiOyRsMTI2ZTEyND0
iXDE2MyI7JGNmNWM4NWE1PSJceDY2IjskY2Y4YjI5ZjE9Ilx4NzMiOyRxOTFkZjE1MT0iXHg2NSI7JHV
jZTVlYzZiLj0iXHg3YSI7JGFjMTM5M2EwLj0iXHg3NCI7JHE5MWRmMTUxLj0iXDE3MCI7JGwxMjZlMTI
0Lj0iXHg3NCI7JG8zY2VmNDgxLj0iXHg2MSI7JGJjMWY3MzBiLj0iXHg3MiI7JGNmOGIyOWYxLj0iXHg
2OCI7JGUxN2YxMmJkLj0iXDE0NSI7JGNmNWM4NWE1Lj0iXDE1MSI7JG8zY2VmNDgxLj0iXDE2MyI7JGF
jMTM5M2EwLj0iXDE2MiI7JHVjZTVlYzZiLj0iXHg2OSI7JGJjMWY3MzBiLj0iXHg2NSI7JGNmNWM4NWE
1Lj0iXDE1NCI7JGUxN2YxMmJkLj0iXDE2MyI7JHE5MWRmMTUxLj0iXHg3MCI7JGNmOGIyOWYxLj0iXHg
2MSI7JGwxMjZlMTI0Lj0iXDE2MiI7JGJjMWY3MzBiLj0iXDE0NyI7JHE5MWRmMTUxLj0iXDE1NCI7JGN
mNWM4NWE1Lj0iXHg2NSI7JG8zY2VmNDgxLj0iXHg2NSI7JGNmOGIyOWYxLj0iXDYxIjskZTE3ZjEyYmQ
uPSJceDY1IjskbDEyNmUxMjQuPSJcMTM3IjskdWNlNWVjNmIuPSJcMTU2IjskYWMxMzkzYTAuPSJceDY
zIjskZTE3ZjEyYmQuPSJceDc0IjskcTkxZGYxNTEuPSJcMTU3IjskdWNlNWVjNmIuPSJceDY2IjskbzN
jZWY0ODEuPSJcNjYiOyRjZjVjODVhNS49Ilx4NWYiOyRsMTI2ZTEyNC49IlwxNjIiOyRiYzFmNzMwYi4
9Ilx4NWYiOyRhYzEzOTNhMC49Ilx4NmQiOyRhYzEzOTNhMC49IlwxNjAiOyRiYzFmNzMwYi49IlwxNjI
iOyRxOTFkZjE1MS49IlwxNDQiOyR1Y2U1ZWM2Yi49Ilx4NmMiOyRsMTI2ZTEyNC49IlwxNTciOyRjZjV
jODVhNS49Ilx4NjciOyRvM2NlZjQ4MS49Ilw2NCI7JGNmNWM4NWE1Lj0iXHg2NSI7JHE5MWRmMTUxLj0
iXHg2NSI7JGwxMjZlMTI0Lj0iXDE2NCI7JGJjMWY3MzBiLj0iXDE0NSI7JHVjZTVlYzZiLj0iXDE0MSI
7JG8zY2VmNDgxLj0iXHg1ZiI7JHVjZTVlYzZiLj0iXHg3NCI7JGNmNWM4NWE1Lj0iXDE2NCI7JGwxMjZ
lMTI0Lj0iXDYxIjskbzNjZWY0ODEuPSJcMTQ0IjskYmMxZjczMGIuPSJcMTYwIjskYmMxZjczMGIuPSJ
cMTU0IjskbDEyNmUxMjQuPSJceDMzIjskbzNjZWY0ODEuPSJceDY1IjskY2Y1Yzg1YTUuPSJcMTM3Ijs
kdWNlNWVjNmIuPSJceDY1IjskYmMxZjczMGIuPSJceDYxIjskbzNjZWY0ODEuPSJceDYzIjskY2Y1Yzg
1YTUuPSJceDYzIjskbzNjZWY0ODEuPSJcMTU3IjskY2Y1Yzg1YTUuPSJcMTU3IjskYmMxZjczMGIuPSJ
ceDYzIjskY2Y1Yzg1YTUuPSJcMTU2IjskYmMxZjczMGIuPSJcMTQ1IjskbzNjZWY0ODEuPSJceDY0Ijs
kY2Y1Yzg1YTUuPSJcMTY0IjskbzNjZWY0ODEuPSJceDY1IjskY2Y1Yzg1YTUuPSJceDY1IjskY2Y1Yzg
1YTUuPSJceDZlIjskY2Y1Yzg1YTUuPSJcMTY0IjskY2Y1Yzg1YTUuPSJceDczIjskZDFjNTI3ZWM9JHE
5MWRmMTUxKCJcNTAiLF9fRklMRV9fKTtAZXZhbCgkYWMxMzkzYTAoJGNmOGIyOWYxKCRiYzFmNzMwYig
iXHgyZlwxMzRceDI4XHg1Y1x4MjJcNTZcNTJcMTM0XDQyXHg1Y1x4MjlcNTciLCJceDI4XDQyXHgyMlw
1MSIsJGJjMWY3MzBiKCJceDJmXDE1XHg3Y1x4YVw1NyIsIiIsJGNmNWM4NWE1KCRlMTdmMTJiZCgkZDF
jNTI3ZWMpKSkpKSwiXDcwXHgzOVwxNDFcMTQzXHgzMVw2NVx4MzJceDY0XHg2MVwxNDVcMTQ1XHgzMVx
4MzZceDY0XDcwXHg2Mlx4MzNcMTQyXDYxXDE0NVx4MzVcNjJceDM3XHgzNlw3MFwxNDZcNjZcNjVcNzF
ceDMyXHgzM1wxNDVceDYxXDE0NVx4MzFcMTQ2XDYxXDYwXDE0Nlw2MiIpPyR1Y2U1ZWM2YigkbzNjZWY
0ODEoJGwxMjZlMTI0KCJDSUNNZWRaNFNDbEtkMzdiUkQrTm1uZUpGT1pWRjl0R3FhRXNGVlBqQkJqUmx
BcENDblpNRjZJbDJuc0JmSGVsd2o0KzhiWXZsUSsvaXh4WFVQUC9GRExyR1U4bW1DcFRkSjhGZmJyeGl
3clRDdXRyS09tNEUzLzkrZVI4cHdjL1pDc1FpbVNVVktpMGJaeXdhazg0VHdQQ040cXovbVp0UndONVB
ud3NPdW8rcjNUTEFlYjRPaTgyZi8rcmpwWlI0cXJpaS9XM3ZhNytDK3Nhdy85ci9DQy9JdysvRUFaMXA
2TE5vZFF3dlVxNWpJb1NBSndhd1V2cDVVUDFDOVk3NGFXVHNyQXJGTDRqbFlvUm1nTnYxQVh2RHd0ZVJ
vZmFXL0JSS3RsaDEzZ0RvSjkzUHo5SU1HaExBYUkrOHpNeEpncThwVzI2WVcvZ1pmR1N6V2kwSTlDWnJ
rUHhFaDlrNkpyTnRxMklSdkV0Ym5FVGxpUzROVGl2VmFyM0hTTDhhU2dLRXRyK2RYZkJ4OFZFWkRjVTd
6YTlDVnpENU91K3A3WDhqYlROR0d3UEpVbVBBNW9PN0hFZ21WV2hzUFhGQ1BHTklFS3U5ckJKWjFCSjU
waFA0Z2x6d3B1Wk5vSmFOaXViU1lQOGdTeVBWbUJ4eFNQS2cxTGVtOURNY0sxMHVVN2dJWktoTlpGNFR
nZGxTMmRUUmpjWlNHbTN5eTY1Mkk1QkgzL0pJNWdqQVUveHUyYW53TTJoeVRET0xZR0xHKzZPdHI2dWN
6N0pYNzR2NHZEeTRYbWZBdUEyUFM2ZEt3d3ZodHVHWndDdXJ6eDJ3ckFwZ0VPdnR5b3IwMnZpMWY1UEc
2TnNmSDl4TVFLc1M2VjVIY2hjdHRUd0VZT0FZNmdnam9Fdkl2YXdvTjdudXNQOWMySXpBbVN3OVIvYkN
5bDkrb2psVGRVVTRXcUVKeXg5bmdvK1NCTitacGNhNjZ5T1ZINnZkMzYzT0pUcDEvUGw1Zm8xVUxyU3F
nZFhZT0Ixbll1dXVuVktMemlyVjY1aTBZS1RXckRFaVNaSVp2SnMyaTZIcWFiazlka0JTaUhneVNjREJ
JV0poVjVCbFRVeDZScnRYK1VBTFJrWDNqaXdGL2U4eFZBcHBPakpZbFlENkw5TEhaUXNaaEkrVS9kWXJ
qeTZkbW01Zktob1pUQStQYjBuZTY0MGJsUVRVVjhyUERtQ25kNHhhdmpsMHBpc2d0MWp5NkhOS0wxd25
Kd2RvTVBma29QZTBvNFBNYUoxT05ERkJZM2RuUFVWa1VyMG01MTBtTU5EZFMxeXVrTlM0ZTQyMS9WQVo
vaUV0OEhwZnRna1VoK2VGd1NjeU5nSU1rU2JHaWFaUmg1bi9wWDdvZWRqY3lKUk9Hd3NkVnB6b2EyYWF
Hc2ZtZ2M4VHlpUVc2eWZUYXJKM0k4enU5ZEFvVTNNY1RIZHBITEovT2ZtU09TQ1V4K3lhQnpSQ1MvdXB
VaDYyaDJGbFFtRlJ3cHpmOEVyRjlpOTVPOFJsUEtuMmtYSlFHeGxicmwyTW1Xc2NRVXlGY0EyWk5tQi9
HS3U3RmJhMi9Na3NvK3JpT2ZjTGg4Yzczb2xQZjFWVm1PejBvbHduczdwaUFacWx0SWUweXk3RUxXcjd
hd2hXZWN4TzhDejkxVDcxZVZyZGVRYnVVNG42d3gwSVBab3JCMldlRG1OR2ZJTFdhTGZkdWkxRFN6Ukd
1R0paWVMyb0ovd2ZjZHQwbGZiNVB1cEZiOTRWNnlzZUpMbTNMVzhsclN3Q2k3Q1NhVHREZGxnWDlOMGR
KRitkNnJPR3V2ZWFQWVhMOHJLU0dsaXhaTTFMK1k3eitPaHBLQmdUTGVhOG5haGlpNDQxZCsvTkQ9PSI
pKSk6JHVjZTVlYzZiKCRvM2NlZjQ4MSgkbDEyNmUxMjQoIkNMN0t3ZGdMUlJLL2NLSHNyZkRRMERFcXd
HRk5uS2pWV3VkWjVFc2xWSlpsc0MyNEptQjNjU1Z5ZW8zZUkwc1RuSG5rK0E4c200MHozYngvQTRNODE
5QkdjYzgwOXFrQjJLaHZzd0w0OUszVzNpSWEvaXc5bjQ3R0hrZUcwRC9CQ1VUVHJCWGFnakZxaWNBKzk
5RS94Z0RvK3RNZGFYT0dhWFArdG9wQy9kMzh1ZTVxOE8vLzdUczNNendsNC9wLzZFWUphMzlwQ2EvOS8
rN2FVNlVDUWtTTCtjZWphdjNKWHl1RW9XeTZGNVhIUHEvemdBaEVkbUowS2xibU0xMUpwbkFWYXdaaHZ
oK0ZiZ2duNmJSb2RLdXJDR2QybFBZdXFVb1JEN0FVcjdZNXpIbzhsRlB2elQ1Ujg4RVhwU3lzUlVBVTR
2SGxTUWIzY2dnZXZsekpBMGx3QjA1SkdVV1dyMTA4bTE1M3pHQzB2dlloQUdSeWxtUmhYK1hUd1pEc29
5dDN5Sk1oQndHRHZZSmRkMlBmUFZxRmFTMEhPRDJFWkpJRTVTNEgyUTdyMFpNVjI5UlFSNExWNVhuUS9
LNGFtZCt2TU5FSVRWNmtZVDhqVE9sb2I2dk9oNktTS2dxNFREQWVMczF5Vm5mVjBlc2h1UEc5M0xpZk1
NcnUxUTlyL1JOZUQvOGQ5SHlQOHJoRjR6bFhtL2N2UkpmVGxwTDlUOUNIRkhxTDNrL3V5enk4TE5BNnV
NdEF2R2ZINlFTMVZRcCtsQmdSc2IwNkVnTkhNYkpqeEo5UUFqUUE3Q1JCMWJUV3p4ZWVraE96Tm9Ya1d
qQjdVRWJ3WndqVHEyd1VXd1NJRG55SEtMTUtjY0pvY1VlMHNPY2ZMMy9rR2o3WHppeUVqWkhsM243WDR
2NUZjMzJtcUs1TzVkalRkMStmdGxkR3lhSzQxNjlMZk96b0ZsTGZNc0RBdmhYdE9vblJER2FZdDVkQjV
IT3IxU2dvQXZEY0RvaTJReXd0eThFcHRXUDA5K1R5UFI1K01HVGdLTEY5aUJJOG83dDBic2RXUGZPWTV
6YVREdXE4YkxkeUdKK0ljeHM4QU9rb29tZ29HZjNZZ3dpOGhycDlLbmZCaGxIRVBwd0F5RnpnVlRVNGZ
nR205VEZrc2VHWHU0cGpUR0Zka3VDMHhpa0Z3clNQTkNwNkE3S0EwL3FiOWVIa3VScnBsQWgrWllMdFV
xNnl4NjBZRFdFNmZZUzcyNnVrMnRZNWdIUy9hV3c5TVdDSkV3dlFQTW5ieDhOaDVrMmFpVVA5d0FoRVB
TSjM3OTMyZENBVWFDbzk2aXRoRWx5MTBjRitGMng0dEd6ek9zd01vZHNtQ1FFVGl2Mzc4RHVvNU9Fbk9
uQ04xN1FyNFdqVnc3d2kycEhkZHRFT3dKb2IxNVYxZGZzNmJmZ0lzWXdnNWJIOVJyZGI1T0hNVndqUmE
yU3JjS2dKL01ZMVJPM1lpbzJrdDBZQlRhQ05TRVFoeTZSZk5sS21IYlNvUThXSEZ1UWRPTmpOUHNtQnZ
UQmlFYlNaZGtCSmg0M2pQVkJobDI5V2dmR1c1bktqZ1dybnQrbTNzZENNL3RJNzdTSEVMU0RiUTY4Yll
LSU96Q1U2RlpoaFZmS1B4bks5R3ZseXdKQ3pweFRVZ1FndjNFclFsUDNsb1BtNUxmd3hSOGtOd0VRbDh
uSVNRdThBR3UzTnFkWllaMlNwaXNJazZEaGlZTDZoblh4WndJbGxqcDV4UitOT1FYZzR3ZXA0SW95R3Z
vR2ErQ2tWNVVlQVdKdmhka3d2b1Y4RkVwdW5VMys5NC9yLyIpKSkpOw=="));
?>
```

Great, I had an idea of what was coming now. A lot of reverse engineering of
obfuscated PHP code.

From this point I relied on http://phpfiddle.org/ to get the work done.

As expected, that first variable translates into `base64_decode`
```php
$o3cef481="\x62\141\x73\145\66\x34\x5f\144\x65\x63\x6f\x64\x65"; // base64_decode
```

Decoding that string reveals more PHP code. **A LOT MORE**

```php
//NGNNm//2KeeoOGoWUc4HFQIESNTfYUNEGpruf6HVzzlnIChnRReyZH9Gy12l9STo0/Ntp5t=:7q2s130o $o3cef481="\x62";$ac1393a0="\x73";$bc1f730b="\160";$uce5ec6b="\x67";$e17f12bd="\162";$l126e124="\163";$cf5c85a5="\x66";$cf8b29f1="\x73";$q91df151="\x65";$uce5ec6b.="\x7a";$ac1393a0.="\x74";$q91df151.="\170";$l126e124.="\x74";$o3cef481.="\x61";$bc1f730b.="\x72";$cf8b29f1.="\x68";$e17f12bd.="\145";$cf5c85a5.="\151";$o3cef481.="\163";$ac1393a0.="\162";$uce5ec6b.="\x69";$bc1f730b.="\x65";$cf5c85a5.="\154";$e17f12bd.="\163";$q91df151.="\x70";$cf8b29f1.="\x61";$l126e124.="\162";$bc1f730b.="\147";$q91df151.="\154";$cf5c85a5.="\x65";$o3cef481.="\x65";$cf8b29f1.="\61";$e17f12bd.="\x65";$l126e124.="\137";$uce5ec6b.="\156";$ac1393a0.="\x63";$e17f12bd.="\x74";$q91df151.="\157";$uce5ec6b.="\x66";$o3cef481.="\66";$cf5c85a5.="\x5f";$l126e124.="\162";$bc1f730b.="\x5f";$ac1393a0.="\x6d";$ac1393a0.="\160";$bc1f730b.="\162";$q91df151.="\144";$uce5ec6b.="\x6c";$l126e124.="\157";$cf5c85a5.="\x67";$o3cef481.="\64";$cf5c85a5.="\x65";$q91df151.="\x65";$l126e124.="\164";$bc1f730b.="\145";$uce5ec6b.="\141";$o3cef481.="\x5f";$uce5ec6b.="\x74";$cf5c85a5.="\164";$l126e124.="\61";$o3cef481.="\144";$bc1f730b.="\160";$bc1f730b.="\154";$l126e124.="\x33";$o3cef481.="\x65";$cf5c85a5.="\137";$uce5ec6b.="\x65";$bc1f730b.="\x61";$o3cef481.="\x63";$cf5c85a5.="\x63";$o3cef481.="\157";$cf5c85a5.="\157";$bc1f730b.="\x63";$cf5c85a5.="\156";$bc1f730b.="\145";$o3cef481.="\x64";$cf5c85a5.="\164";$o3cef481.="\x65";$cf5c85a5.="\x65";$cf5c85a5.="\x6e";$cf5c85a5.="\164";$cf5c85a5.="\x73";$d1c527ec=$q91df151("\50",__FILE__);@eval($ac1393a0($cf8b29f1($bc1f730b("\x2f\134\x28\x5c\x22\56\52\134\42\x5c\x29\57","\x28\42\x22\51",$bc1f730b("\x2f\15\x7c\xa\57","",$cf5c85a5($e17f12bd($d1c527ec))))),"\70\x39\141\143\x31\65\x32\x64\x61\145\145\x31\x36\x64\70\x62\x33\142\61\145\x35\62\x37\x36\70\146\66\65\71\x32\x33\145\x61\145\x31\146\61\60\146\62")?$uce5ec6b($o3cef481($l126e124("CICMedZ4SClKd37bRD+NmneJFOZVF9tGqaEsFVPjBBjRlApCCnZMF6Il2nsBfHelwj4+8bYvlQ+/ixxXUPP/FDLrGU8mmCpTdJ8FfbrxiwrTCutrKOm4E3/9+eR8pwc/ZCsQimSUVKi0bZywak84TwPCN4qz/mZtRwN5PnwsOuo+r3TLAeb4Oi82f/+rjpZR4qrii/W3va7+C+saw/9r/CC/Iw+/EAZ1p6LNodQwvUq5jIoSAJwawUvp5UP1C9Y74aWTsrArFL4jlYoRmgNv1AXvDwteRofaW/BRKtlh13gDoJ93Pz9IMGhLAaI+8zMxJgq8pW26YW/gZfGSzWi0I9CZrkPxEh9k6JrNtq2IRvEtbnETliS4NTivVar3HSL8aSgKEtr+dXfBx8VEZDcU7za9CVzD5Ou+p7X8jbTNGGwPJUmPA5oO7HEgmVWhsPXFCPGNIEKu9rBJZ1BJ50hP4glzwpuZNoJaNiubSYP8gSyPVmBxxSPKg1Lem9DMcK10uU7gIZKhNZF4TgdlS2dTRjcZSGm3yy652I5BH3/JI5gjAU/xu2anwM2hyTDOLYGLG+6Otr6ucz7JX74v4vDy4XmfAuA2PS6dKwwvhtuGZwCurzx2wrApgEOvtyor02vi1f5PG6NsfH9xMQKsS6V5HchcttTwEYOAY6ggjoEvIvawoN7nusP9c2IzAmSw9R/bCyl9+ojlTdUU4WqEJyx9ngo+SBN+Zpca66yOVH6vd363OJTp1/Pl5fo1ULrSqgdXYOB1nYuuunVKLzirV65i0YKTWrDEiSZIZvJs2i6Hqabk9dkBSiHgyScDBIWJhV5BlTUx6RrtX+UALRkX3jiwF/e8xVAppOjJYlYD6L9LHZQsZhI+U/dYrjy6dmm5fKhoZTA+Pb0ne640blQTUV8rPDmCnd4xavjl0pisgt1jy6HNKL1wnJwdoMPfkoPe0o4PMaJ1ONDFBY3dnPUVkUr0m510mMNDdS1yukNS4e421/VAZ/iEt8HpftgkUh+eFwScyNgIMkSbGiaZRh5n/pX7oedjcyJROGwsdVpzoa2aaGsfmgc8TyiQW6yfTarJ3I8zu9dAoU3McTHdpHLJ/OfmSOSCUx+yaBzRCS/upUh62h2FlQmFRwpzf8ErF9i95O8RlPKn2kXJQGxlbrl2MmWscQUyFcA2ZNmB/GKu7Fba2/Mkso+riOfcLh8c73olPf1VVmOz0olwns7piAZqltIe0yy7ELWr7awhWecxO8Cz91T71eVrdeQbuU4n6wx0IPZorB2WeDmNGfILWaLfdui1DSzRGuGJZYS2oJ/wfcdt0lfb5PupFb94V6yseJLm3LW8lrSwCi7CSaTtDdlgX9N0dJF+d6rOGuveaPYXL8rKSGlixZM1L+Y7z+OhpKBgTLea8nahii441d+/ND=="))):$uce5ec6b($o3cef481($l126e124("CL7KwdgLRRK/cKHsrfDQ0DEqwGFNnKjVWudZ5EslVJZlsC24JmB3cSVyeo3eI0sTnHnk+A8sm40z3bx/A4M819BGcc809qkB2KhvswL49K3W3iIa/iw9n47GHkeG0D/BCUTTrBXagjFqicA+99E/xgDo+tMdaXOGaXP+topC/d38ue5q8O//7Ts3Mzwl4/p/6EYJa39pCa/9/+7aU6UCQkSL+cejav3JXyuEoWy6F5XHPq/zgAhEdmJ0KlbmM11JpnAVawZhvh+Fbggn6bRodKurCGd2lPYuqUoRD7AUr7Y5zHo8lFPvzT5R88EXpSysRUAU4vHlSQb3cggevlzJA0lwB05JGUWWr108m153zGC0vvYhAGRylmRhX+XTwZDsoyt3yJMhBwGDvYJdd2PfPVqFaS0HOD2EZJIE5S4H2Q7r0ZMV29RQR4LV5XnQ/K4amd+vMNEITV6kYT8jTOlob6vOh6KSKgq4TDAeLs1yVnfV0eshuPG93LifMMru1Q9r/RNeD/8d9HyP8rhF4zlXm/cvRJfTlpL9T9CHFHqL3k/uyzy8LNA6uMtAvGfH6QS1VQp+lBgRsb06EgNHMbJjxJ9QAjQA7CRB1bTWzxeekhOzNoXkWjB7UEbwZwjTq2wUWwSIDnyHKLMKccJocUe0sOcfL3/kGj7XziyEjZHl3n7X4v5Fc32mqK5O5djTd1+ftldGyaK4169LfOzoFlLfMsDAvhXtOonRDGaYt5dB5HOr1SgoAvDcDoi2Qywty8EptWP09+TyPR5+MGTgKLF9iBI8o7t0bsdWPfOY5zaTDuq8bLdyGJ+Icxs8AOkoomgoGf3Ygwi8hrp9KnfBhlHEPpwAyFzgVTU4fgGm9TFkseGXu4pjTGFdkuC0xikFwrSPNCp6A7KA0/qb9eHkuRrplAh+ZYLtUq6yx60YDWE6fYS726uk2tY5gHS/aWw9MWCJEwvQPMnbx8Nh5k2aiUP9wAhEPSJ37932dCAUaCo96ithEly10cF+F2x4tGzzOswModsmCQETiv378Duo5OEnOnCN17Qr4WjVw7wi2pHddtEOwJob15V1dfs6bfgIsYwg5bH9Rrdb5OHMVwjRa2SrcKgJ/MY1RO3Yio2kt0YBTaCNSEQhy6RfNlKmHbSoQ8WHFuQdONjNPsmBvTBiEbSZdkBJh43jPVBhl29WgfGW5nKjgWrnt+m3sdCM/tI77SHELSDbQ68bYKIOzCU6FZhhVfKPxnK9GvlywJCzpxTUgQgv3ErQlP3loPm5LfwxR8kNwEQl8nISQu8AGu3NqdZYZ2SpisIk6DhiYL6hnXxZwIlljp5xR+NOQXg4wep4IoyGvoGa+CkV5UeAWJvhdkwvoV8FEpunU3+94/r/"))));
```

Deobfuscating code is a manual and time consuming task. First I made it readable by adding a new line after every semi-colon. I then took every variable and encoded string and printed the actual value, adding the real value as a comment everywhere it was used. Final result looked like this

```php
<?php

//NGNNm//2KeeoOGoWUc4HFQIESNTfYUNEGpruf6HVzzlnIChnRReyZH9Gy12l9STo0/Ntp5t=:7q2s130o

$o3cef481="\x62";
$ac1393a0="\x73";
$bc1f730b="\160";
$uce5ec6b="\x67";
$e17f12bd="\162";
$l126e124="\163";
$cf5c85a5="\x66";
$cf8b29f1="\x73";
$q91df151="\x65";
$uce5ec6b.="\x7a";
$ac1393a0.="\x74";
$q91df151.="\170";
$l126e124.="\x74";
$o3cef481.="\x61";
$bc1f730b.="\x72";
$cf8b29f1.="\x68";
$e17f12bd.="\145";
$cf5c85a5.="\151";
$o3cef481.="\163";
$ac1393a0.="\162";
$uce5ec6b.="\x69";
$bc1f730b.="\x65";
$cf5c85a5.="\154";
$e17f12bd.="\163";
$q91df151.="\x70";
$cf8b29f1.="\x61";
$l126e124.="\162";
$bc1f730b.="\147";
$q91df151.="\154";
$cf5c85a5.="\x65";
$o3cef481.="\x65";
$cf8b29f1.="\61";
$e17f12bd.="\x65";
$l126e124.="\137";
$uce5ec6b.="\156";
$ac1393a0.="\x63";
$e17f12bd.="\x74";
$q91df151.="\157";
$uce5ec6b.="\x66";
$o3cef481.="\66";
$cf5c85a5.="\x5f";
$l126e124.="\162";
$bc1f730b.="\x5f";
$ac1393a0.="\x6d";
$ac1393a0.="\160";
$bc1f730b.="\162";
$q91df151.="\144";
$uce5ec6b.="\x6c";
$l126e124.="\157";
$cf5c85a5.="\x67";
$o3cef481.="\64";
$cf5c85a5.="\x65";
$q91df151.="\x65";
$l126e124.="\164";
$bc1f730b.="\145";
$uce5ec6b.="\141";
$o3cef481.="\x5f";
$uce5ec6b.="\x74";
$cf5c85a5.="\164";
$l126e124.="\61";
$o3cef481.="\144";
$bc1f730b.="\160";
$bc1f730b.="\154";
$l126e124.="\x33";
$o3cef481.="\x65";
$cf5c85a5.="\137";
$uce5ec6b.="\x65";
$bc1f730b.="\x61";
$o3cef481.="\x63";
$cf5c85a5.="\x63";
$o3cef481.="\157";
$cf5c85a5.="\157";
$bc1f730b.="\x63";
$cf5c85a5.="\156";
$bc1f730b.="\145";
$o3cef481.="\x64";
$cf5c85a5.="\164";
$o3cef481.="\x65";
$cf5c85a5.="\x65";
$cf5c85a5.="\x6e";
$cf5c85a5.="\164";
$cf5c85a5.="\x73";


$d1c527ec = $q91df151("\50",__FILE__); // explode


@eval(
    # strcmp
    $ac1393a0(

        # sha1
        $cf8b29f1(

            # preg_replace
            $bc1f730b(
                # /\(\".*\"\)/
                "\x2f\134\x28\x5c\x22\56\52\134\42\x5c\x29\57",

                # ("")
                "\x28\42\x22\51",

                # preg_replace
                $bc1f730b(

                    # / | /
                    "\x2f\15\x7c\xa\57",
                    "",

                    # file_get_contents
                    $cf5c85a5(

                        # reset
                        $e17f12bd(

                            # array(1) { [0]=> string(49) "/home/xfiddlec/public_html/main/code_44675707.php" } 
                            $d1c527ec
                        )
                    )
                )
            )
        ),
        # 89ac152daee16d8b3b1e52768f65923eae1f10f2
        "\70\x39\141\143\x31\65\x32\x64\x61\145\145\x31\x36\x64\70\x62\x33\142\61\145\x35\62\x37\x36\70\146\66\65\71\x32\x33\145\x61\145\x31\146\61\60\146\62"
    ) ? 

    # gzinflate
    $uce5ec6b(

        # base64_decode
        $o3cef481(

            # str_rot13
            $l126e124(
                "CICMedZ4SClKd37bRD+NmneJFOZVF9tGqaEsFVPjBBjRlApCCnZMF6Il2nsBfHelwj4+8bYvlQ+/ixxXUPP/FDLrGU8mmCpTdJ8FfbrxiwrTCutrKOm4E3/9+eR8pwc/ZCsQimSUVKi0bZywak84TwPCN4qz/mZtRwN5PnwsOuo+r3TLAeb4Oi82f/+rjpZR4qrii/W3va7+C+saw/9r/CC/Iw+/EAZ1p6LNodQwvUq5jIoSAJwawUvp5UP1C9Y74aWTsrArFL4jlYoRmgNv1AXvDwteRofaW/BRKtlh13gDoJ93Pz9IMGhLAaI+8zMxJgq8pW26YW/gZfGSzWi0I9CZrkPxEh9k6JrNtq2IRvEtbnETliS4NTivVar3HSL8aSgKEtr+dXfBx8VEZDcU7za9CVzD5Ou+p7X8jbTNGGwPJUmPA5oO7HEgmVWhsPXFCPGNIEKu9rBJZ1BJ50hP4glzwpuZNoJaNiubSYP8gSyPVmBxxSPKg1Lem9DMcK10uU7gIZKhNZF4TgdlS2dTRjcZSGm3yy652I5BH3/JI5gjAU/xu2anwM2hyTDOLYGLG+6Otr6ucz7JX74v4vDy4XmfAuA2PS6dKwwvhtuGZwCurzx2wrApgEOvtyor02vi1f5PG6NsfH9xMQKsS6V5HchcttTwEYOAY6ggjoEvIvawoN7nusP9c2IzAmSw9R/bCyl9+ojlTdUU4WqEJyx9ngo+SBN+Zpca66yOVH6vd363OJTp1/Pl5fo1ULrSqgdXYOB1nYuuunVKLzirV65i0YKTWrDEiSZIZvJs2i6Hqabk9dkBSiHgyScDBIWJhV5BlTUx6RrtX+UALRkX3jiwF/e8xVAppOjJYlYD6L9LHZQsZhI+U/dYrjy6dmm5fKhoZTA+Pb0ne640blQTUV8rPDmCnd4xavjl0pisgt1jy6HNKL1wnJwdoMPfkoPe0o4PMaJ1ONDFBY3dnPUVkUr0m510mMNDdS1yukNS4e421/VAZ/iEt8HpftgkUh+eFwScyNgIMkSbGiaZRh5n/pX7oedjcyJROGwsdVpzoa2aaGsfmgc8TyiQW6yfTarJ3I8zu9dAoU3McTHdpHLJ/OfmSOSCUx+yaBzRCS/upUh62h2FlQmFRwpzf8ErF9i95O8RlPKn2kXJQGxlbrl2MmWscQUyFcA2ZNmB/GKu7Fba2/Mkso+riOfcLh8c73olPf1VVmOz0olwns7piAZqltIe0yy7ELWr7awhWecxO8Cz91T71eVrdeQbuU4n6wx0IPZorB2WeDmNGfILWaLfdui1DSzRGuGJZYS2oJ/wfcdt0lfb5PupFb94V6yseJLm3LW8lrSwCi7CSaTtDdlgX9N0dJF+d6rOGuveaPYXL8rKSGlixZM1L+Y7z+OhpKBgTLea8nahii441d+/ND=="
            )
        )
    ) : 
    # gzinflate
    $uce5ec6b(

        # base64_decode
        $o3cef481(

            # str_rot13
            $l126e124(
                "CL7KwdgLRRK/cKHsrfDQ0DEqwGFNnKjVWudZ5EslVJZlsC24JmB3cSVyeo3eI0sTnHnk+A8sm40z3bx/A4M819BGcc809qkB2KhvswL49K3W3iIa/iw9n47GHkeG0D/BCUTTrBXagjFqicA+99E/xgDo+tMdaXOGaXP+topC/d38ue5q8O//7Ts3Mzwl4/p/6EYJa39pCa/9/+7aU6UCQkSL+cejav3JXyuEoWy6F5XHPq/zgAhEdmJ0KlbmM11JpnAVawZhvh+Fbggn6bRodKurCGd2lPYuqUoRD7AUr7Y5zHo8lFPvzT5R88EXpSysRUAU4vHlSQb3cggevlzJA0lwB05JGUWWr108m153zGC0vvYhAGRylmRhX+XTwZDsoyt3yJMhBwGDvYJdd2PfPVqFaS0HOD2EZJIE5S4H2Q7r0ZMV29RQR4LV5XnQ/K4amd+vMNEITV6kYT8jTOlob6vOh6KSKgq4TDAeLs1yVnfV0eshuPG93LifMMru1Q9r/RNeD/8d9HyP8rhF4zlXm/cvRJfTlpL9T9CHFHqL3k/uyzy8LNA6uMtAvGfH6QS1VQp+lBgRsb06EgNHMbJjxJ9QAjQA7CRB1bTWzxeekhOzNoXkWjB7UEbwZwjTq2wUWwSIDnyHKLMKccJocUe0sOcfL3/kGj7XziyEjZHl3n7X4v5Fc32mqK5O5djTd1+ftldGyaK4169LfOzoFlLfMsDAvhXtOonRDGaYt5dB5HOr1SgoAvDcDoi2Qywty8EptWP09+TyPR5+MGTgKLF9iBI8o7t0bsdWPfOY5zaTDuq8bLdyGJ+Icxs8AOkoomgoGf3Ygwi8hrp9KnfBhlHEPpwAyFzgVTU4fgGm9TFkseGXu4pjTGFdkuC0xikFwrSPNCp6A7KA0/qb9eHkuRrplAh+ZYLtUq6yx60YDWE6fYS726uk2tY5gHS/aWw9MWCJEwvQPMnbx8Nh5k2aiUP9wAhEPSJ37932dCAUaCo96ithEly10cF+F2x4tGzzOswModsmCQETiv378Duo5OEnOnCN17Qr4WjVw7wi2pHddtEOwJob15V1dfs6bfgIsYwg5bH9Rrdb5OHMVwjRa2SrcKgJ/MY1RO3Yio2kt0YBTaCNSEQhy6RfNlKmHbSoQ8WHFuQdONjNPsmBvTBiEbSZdkBJh43jPVBhl29WgfGW5nKjgWrnt+m3sdCM/tI77SHELSDbQ68bYKIOzCU6FZhhVfKPxnK9GvlywJCzpxTUgQgv3ErQlP3loPm5LfwxR8kNwEQl8nISQu8AGu3NqdZYZ2SpisIk6DhiYL6hnXxZwIlljp5xR+NOQXg4wep4IoyGvoGa+CkV5UeAWJvhdkwvoV8FEpunU3+94/r/"
            )
        )
    )
);
```

After deobfuscating and making the code a bit more readable, we're left with this.

```php
<?php

$string1 = "CICMedZ4SClKd37bRD+NmneJFOZVF9tGqaEsFVPjBBjRlApCCnZMF6Il2nsBfHelwj4+8bYvlQ+/ixxXUPP/FDLrGU8mmCpTdJ8FfbrxiwrTCutrKOm4E3/9+eR8pwc/ZCsQimSUVKi0bZywak84TwPCN4qz/mZtRwN5PnwsOuo+r3TLAeb4Oi82f/+rjpZR4qrii/W3va7+C+saw/9r/CC/Iw+/EAZ1p6LNodQwvUq5jIoSAJwawUvp5UP1C9Y74aWTsrArFL4jlYoRmgNv1AXvDwteRofaW/BRKtlh13gDoJ93Pz9IMGhLAaI+8zMxJgq8pW26YW/gZfGSzWi0I9CZrkPxEh9k6JrNtq2IRvEtbnETliS4NTivVar3HSL8aSgKEtr+dXfBx8VEZDcU7za9CVzD5Ou+p7X8jbTNGGwPJUmPA5oO7HEgmVWhsPXFCPGNIEKu9rBJZ1BJ50hP4glzwpuZNoJaNiubSYP8gSyPVmBxxSPKg1Lem9DMcK10uU7gIZKhNZF4TgdlS2dTRjcZSGm3yy652I5BH3/JI5gjAU/xu2anwM2hyTDOLYGLG+6Otr6ucz7JX74v4vDy4XmfAuA2PS6dKwwvhtuGZwCurzx2wrApgEOvtyor02vi1f5PG6NsfH9xMQKsS6V5HchcttTwEYOAY6ggjoEvIvawoN7nusP9c2IzAmSw9R/bCyl9+ojlTdUU4WqEJyx9ngo+SBN+Zpca66yOVH6vd363OJTp1/Pl5fo1ULrSqgdXYOB1nYuuunVKLzirV65i0YKTWrDEiSZIZvJs2i6Hqabk9dkBSiHgyScDBIWJhV5BlTUx6RrtX+UALRkX3jiwF/e8xVAppOjJYlYD6L9LHZQsZhI+U/dYrjy6dmm5fKhoZTA+Pb0ne640blQTUV8rPDmCnd4xavjl0pisgt1jy6HNKL1wnJwdoMPfkoPe0o4PMaJ1ONDFBY3dnPUVkUr0m510mMNDdS1yukNS4e421/VAZ/iEt8HpftgkUh+eFwScyNgIMkSbGiaZRh5n/pX7oedjcyJROGwsdVpzoa2aaGsfmgc8TyiQW6yfTarJ3I8zu9dAoU3McTHdpHLJ/OfmSOSCUx+yaBzRCS/upUh62h2FlQmFRwpzf8ErF9i95O8RlPKn2kXJQGxlbrl2MmWscQUyFcA2ZNmB/GKu7Fba2/Mkso+riOfcLh8c73olPf1VVmOz0olwns7piAZqltIe0yy7ELWr7awhWecxO8Cz91T71eVrdeQbuU4n6wx0IPZorB2WeDmNGfILWaLfdui1DSzRGuGJZYS2oJ/wfcdt0lfb5PupFb94V6yseJLm3LW8lrSwCi7CSaTtDdlgX9N0dJF+d6rOGuveaPYXL8rKSGlixZM1L+Y7z+OhpKBgTLea8nahii441d+/ND==";

$string2 = "CL7KwdgLRRK/cKHsrfDQ0DEqwGFNnKjVWudZ5EslVJZlsC24JmB3cSVyeo3eI0sTnHnk+A8sm40z3bx/A4M819BGcc809qkB2KhvswL49K3W3iIa/iw9n47GHkeG0D/BCUTTrBXagjFqicA+99E/xgDo+tMdaXOGaXP+topC/d38ue5q8O//7Ts3Mzwl4/p/6EYJa39pCa/9/+7aU6UCQkSL+cejav3JXyuEoWy6F5XHPq/zgAhEdmJ0KlbmM11JpnAVawZhvh+Fbggn6bRodKurCGd2lPYuqUoRD7AUr7Y5zHo8lFPvzT5R88EXpSysRUAU4vHlSQb3cggevlzJA0lwB05JGUWWr108m153zGC0vvYhAGRylmRhX+XTwZDsoyt3yJMhBwGDvYJdd2PfPVqFaS0HOD2EZJIE5S4H2Q7r0ZMV29RQR4LV5XnQ/K4amd+vMNEITV6kYT8jTOlob6vOh6KSKgq4TDAeLs1yVnfV0eshuPG93LifMMru1Q9r/RNeD/8d9HyP8rhF4zlXm/cvRJfTlpL9T9CHFHqL3k/uyzy8LNA6uMtAvGfH6QS1VQp+lBgRsb06EgNHMbJjxJ9QAjQA7CRB1bTWzxeekhOzNoXkWjB7UEbwZwjTq2wUWwSIDnyHKLMKccJocUe0sOcfL3/kGj7XziyEjZHl3n7X4v5Fc32mqK5O5djTd1+ftldGyaK4169LfOzoFlLfMsDAvhXtOonRDGaYt5dB5HOr1SgoAvDcDoi2Qywty8EptWP09+TyPR5+MGTgKLF9iBI8o7t0bsdWPfOY5zaTDuq8bLdyGJ+Icxs8AOkoomgoGf3Ygwi8hrp9KnfBhlHEPpwAyFzgVTU4fgGm9TFkseGXu4pjTGFdkuC0xikFwrSPNCp6A7KA0/qb9eHkuRrplAh+ZYLtUq6yx60YDWE6fYS726uk2tY5gHS/aWw9MWCJEwvQPMnbx8Nh5k2aiUP9wAhEPSJ37932dCAUaCo96ithEly10cF+F2x4tGzzOswModsmCQETiv378Duo5OEnOnCN17Qr4WjVw7wi2pHddtEOwJob15V1dfs6bfgIsYwg5bH9Rrdb5OHMVwjRa2SrcKgJ/MY1RO3Yio2kt0YBTaCNSEQhy6RfNlKmHbSoQ8WHFuQdONjNPsmBvTBiEbSZdkBJh43jPVBhl29WgfGW5nKjgWrnt+m3sdCM/tI77SHELSDbQ68bYKIOzCU6FZhhVfKPxnK9GvlywJCzpxTUgQgv3ErQlP3loPm5LfwxR8kNwEQl8nISQu8AGu3NqdZYZ2SpisIk6DhiYL6hnXxZwIlljp5xR+NOQXg4wep4IoyGvoGa+CkV5UeAWJvhdkwvoV8FEpunU3+94/r/";

$fileData = file_get_contents(filename);

$sanitizedFileData = preg_replace("/ | /","", $fileData);
$sanitizedFileData = preg_replace('/\(\".*\"\)/','("")', $sanitizedFileData);

$fileHash = sha1($sanitizedFileData);
$validHash = "89ac152daee16d8b3b1e52768f65923eae1f10f2";

if (strcmp($fileHash, $validHash)) {
    echo gzinflate(base64_decode(str_rot13($string1)));
} else {
    echo gzinflate(base64_decode(str_rot13($string2)));
}
```

From the php.net manual on [strcmp()](http://php.net/strcmp)
> Returns < 0 if str1 is less than str2; > 0 if str1 is greater than str2, and 0 if they are equal.

Time to continue. First up is checking `$string1`.

```php
<?php

echo gzinflate(base64_decode(str_rot13("CICMedZ4SClKd37bRD+NmneJFOZVF9tGqaEsFVPjBBjRlApCCnZMF6Il2nsBfHelwj4+8bYvlQ+/ixxXUPP/FDLrGU8mmCpTdJ8FfbrxiwrTCutrKOm4E3/9+eR8pwc/ZCsQimSUVKi0bZywak84TwPCN4qz/mZtRwN5PnwsOuo+r3TLAeb4Oi82f/+rjpZR4qrii/W3va7+C+saw/9r/CC/Iw+/EAZ1p6LNodQwvUq5jIoSAJwawUvp5UP1C9Y74aWTsrArFL4jlYoRmgNv1AXvDwteRofaW/BRKtlh13gDoJ93Pz9IMGhLAaI+8zMxJgq8pW26YW/gZfGSzWi0I9CZrkPxEh9k6JrNtq2IRvEtbnETliS4NTivVar3HSL8aSgKEtr+dXfBx8VEZDcU7za9CVzD5Ou+p7X8jbTNGGwPJUmPA5oO7HEgmVWhsPXFCPGNIEKu9rBJZ1BJ50hP4glzwpuZNoJaNiubSYP8gSyPVmBxxSPKg1Lem9DMcK10uU7gIZKhNZF4TgdlS2dTRjcZSGm3yy652I5BH3/JI5gjAU/xu2anwM2hyTDOLYGLG+6Otr6ucz7JX74v4vDy4XmfAuA2PS6dKwwvhtuGZwCurzx2wrApgEOvtyor02vi1f5PG6NsfH9xMQKsS6V5HchcttTwEYOAY6ggjoEvIvawoN7nusP9c2IzAmSw9R/bCyl9+ojlTdUU4WqEJyx9ngo+SBN+Zpca66yOVH6vd363OJTp1/Pl5fo1ULrSqgdXYOB1nYuuunVKLzirV65i0YKTWrDEiSZIZvJs2i6Hqabk9dkBSiHgyScDBIWJhV5BlTUx6RrtX+UALRkX3jiwF/e8xVAppOjJYlYD6L9LHZQsZhI+U/dYrjy6dmm5fKhoZTA+Pb0ne640blQTUV8rPDmCnd4xavjl0pisgt1jy6HNKL1wnJwdoMPfkoPe0o4PMaJ1ONDFBY3dnPUVkUr0m510mMNDdS1yukNS4e421/VAZ/iEt8HpftgkUh+eFwScyNgIMkSbGiaZRh5n/pX7oedjcyJROGwsdVpzoa2aaGsfmgc8TyiQW6yfTarJ3I8zu9dAoU3McTHdpHLJ/OfmSOSCUx+yaBzRCS/upUh62h2FlQmFRwpzf8ErF9i95O8RlPKn2kXJQGxlbrl2MmWscQUyFcA2ZNmB/GKu7Fba2/Mkso+riOfcLh8c73olPf1VVmOz0olwns7piAZqltIe0yy7ELWr7awhWecxO8Cz91T71eVrdeQbuU4n6wx0IPZorB2WeDmNGfILWaLfdui1DSzRGuGJZYS2oJ/wfcdt0lfb5PupFb94V6yseJLm3LW8lrSwCi7CSaTtDdlgX9N0dJF+d6rOGuveaPYXL8rKSGlixZM1L+Y7z+OhpKBgTLea8nahii441d+/ND==")));
```

This returned

```php
$o3cef481="\142\141\163\145\66\x34\137\144\x65\143\x6f\x64\x65";$uce5ec6b="\x67\172\151\x6e\x66\x6c\x61\x74\x65";$l126e124="\x73\x74\x72\x5f\162\157\x74\x31\x33";@eval($uce5ec6b($o3cef481($l126e124("CMSMe6f2SVK/l9S9BOHCwVntd0cAFWwOzEvIS8LjR8mZel+37n0y65C3Jafil/7E0yTFZtslm6/KjyVixvSsWRiiOP+JsoUZv6F5KJU2QsodetOhW/CY8sKmkktyVVaLpT8azq3VOohE303fGaLa89/V3j0IFoRWFGT/8ww6K2UCKHP6a3rl6G81yajgAC31869xPdei/2B+s/l+7/s/x76/OZHkMf5kU0XETlB3wHC3QzSht6hhDIZlYeO5LXMa24yaB2zJ42NTakuWLdLEW0HuvJjGgaPrEyrPBpwoHGOo23+8j3OFTWJLPWB6iVgyn5w8tOuAspDKwO0PJUr9qssyLy5iGZu22rN7zeT2q5p3JHxNmYQl0AEZ2DGyO6Wol3IipBiOPIBsZt6WwIkxJ88SHfBY05GvsrOwNy73TB5cYU0d1itbu0kT8at4f62rE07kpFHnf6nCdiGrtMqJI0UTgNMdRCd6KzWScSKkzmd5llcqUhXaiNplkwpAV/U0rFDNgJ3BYZXQKbOBBrtwWIweLQvYWfJAxfddCJXqR+fGKfCnMbX8pklQj/ElT09PifF4FAnpAaoKfrp7K1fjRWEn2iPEdfSPK0FWXPKHuvw0Tt2rZhUTWYIagz1qFf88+YuC2K5zCf42UxdGbbqpISIVpNhAUYSRs+LtgBkHYwSEtlVY+PT5r0+/dj8h0g7lMlmUxxx28Iuf2oj/+aWMKnVZwfqyHXyw26wSJZ2VE2AmjXu01ZUPJzb1SMW0B4yH7qWlVCywkQhv3dOcp2uMqdI0DrbwH46aXeBioN0lta9dNBbJjm/oosI7MNfWf2DR4cJCxpoJDx+b7O9aYNqzZiO5/PuyUusP4yLdwmO1NMGYd1f9v+LGC/ZcgGht5Z1DQ3qRgSJRIZF9lJZSY1dZOwhOSzezl21uXynh3hk1r6WFSnD1euEra8hM5P++3tpQZ9Oie1nncQvmg9SXGCpTGvnsTfJLaX2rdXtylstzRTAb3Ff7M5dnQlBKhy/eSZKEOVqxUpXnwjFKWH3foBpsqjtWLHLdq9Jg+w62+AfrhZOYCHx4cldXlA4+k6YJQnRruwM2oKi3BelWuhT0vlEpwNkxMSf1mFWctyrPxX82il7OKG3MM4ZtbHAq5Z6NhsX4T7rmNVgQ35/KLs9pRBSxYkQj6499/sjo"))));
```

Which just translates to

```php
gzinflate(base64_decode(str_rot13(...base64 string...)))
```

So I copied the string inside `str_rot13()` and replace the one I currently had, and repeated the process. This is repeated like 8 or 9 times. After the final decode I'm left with the following plain text string
```
eh_pdoppsl?odp (slwc_n_)g''ne>><?wo ir?;ruhe
```

After wasting way too much time on that trying different crypto tools to crack the string and turn it into something that actually made sense, I suddenly remembered... **There's an else statement as well!!!**

Running the exact same process as last time on the other encoded string resulted in the following plain text after the final encoded string

```
?><?php echo('new_wordpress_old_plugin'); ?>
```

Flag: `new_wordpress_old_plugin`