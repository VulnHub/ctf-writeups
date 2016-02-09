### Solved by superkojiman

A 10 point freebie question. The flag we need is split into two. We're told that the first half can be found somewhere in [BackdoorCTF's Twitter account](https://twitter.com/Backdoorctf), and the other half on their IRC channel #backdoorctf15. For Twitter, I just looked through all the posts until I found the first half of the flag [here](https://twitter.com/BackdoorCTF/status/570822175420129280). For the IRC channel, it was in the topic. 

Twitter: 2041615da55336e54cae0692663c2cf35aaa33188c6fa68c56a813f878d9a24d
IRC: d68e69f3bbebbb346d338cdf937009ba3c69e5bea60158de2d47da09574f9342

Concatenate the strings and get the SHA-256 sum to get the flag:

```
$ printf "2041615da55336e54cae0692663c2cf35aaa33188c6fa68c56a813f878d9a24dd68e69f3bbebbb346d338cdf937009ba3c69e5bea60158de2d47da09574f9342" | shasum -a 256
flag_goes_here_flag_goes_here_flag_goes_here_flag_goes_here_flag  -
```

