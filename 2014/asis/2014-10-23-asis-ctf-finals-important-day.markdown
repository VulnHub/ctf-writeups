### Solved by Swappage

I'M LAAAATE!!

yeh I know i'm late and i hardly keep up with the writeups!, they should advertise that CTFs can be addictive and cause cronic sleep deprevation, which with massive work doesn't match well :(

Important Day was a 100 points challenge where you were provided with a pcap file, and asked to determine when the system was powered on.

By looking at the pcap file it really looked like a portscan, and by a quick google search i ran into a post from back in 2006 on seclists.org discussing about the possibility to guess the time of the last system reboot, by looking at the TCP protocol TSVAL option.
<!--more--!>

![](/images/2014/asis-finals/importantday/scan.png)

Bsically since the TSVAL counter is reset to zero every time the system reboots (at least on many OSes), if you know the frequency at which TSVAL is increased, you can try to guess the boot timestamp.

As the challenge was providing a pcap, we had the capture timestamp, so if we had enaugh TSVAL options to compare, we could try to guess the system uptime.

The tickrate at which TSVAL increases is different from Operating System to Operating System, and with only the pcap i didn't know the  target OS, but i had multiple packets to analyze, so it was probably a matter of math.

I decided to set the following filter

	tcp.options.timestamp.tsval && ip.src == 192.168.100.78

so that i could get only the packets having the tsval option field set, and coming from the target system (not the stanning one)

![](/images/2014/asis-finals/importantday/filtered.png)

And then analyzed how the TSVAL increased compared to the milliseconds in the packet capture timestamp.

By picking packets number number 4034 and 4039 we can notice:

	4034: Timestamp in ms: 1412157739276	TSVAL: 2400803286
	4039: Timestamp in ms: 1412157739438	TSVAL: 2400803327

So with some simple math

	1412157739438 - 1412157739276 = 162
	2400803327 - 2400803286 = 41

we can figure out that TSVAL increases by 1 tick every ~4ms

Ok, awesome
So now if our assumption was correct, we need to figure out what was the timestamp when TSVAL was 0:

so with some other simple math we discover

	(1412157739438 - 2400803327 * 4) / 1000= 1402554526.130

This timestamp corresponds to Thu, 12 Jun 2014 06:28:46 GMT

The flag was in the format of md5(ASIS_date) where date was in the format "%Y:%m:%d:%H:%M"

A quick conversion did the trick

```python
	>>> print(datetime.datetime.fromtimestamp(1402554526).strftime("%Y:%m:%d:%H:%M"))
	2014:06:12:10:58
```

And at this point you'd say: **"Wait, what's this?, why 10:58?"**

And that's the good point, and the fact that really disappointed me, as i spent *a lot of time* with trial and error thinking i was doing things wrong, when at  a certain point, with all the frustration i had accumulated, i went on the CTF IRC channel asking an admin if something was broken, and then he pointed out that i was supposed to provide the date in IRST timezone.

I mean, WHAT? are you kidding me? timestamps in forensics not in UTC? come on, please, guess what would happen if you were to ask logs to an ISP for a specific IP address at a certain timestamp not in UTC.. you might send an innocent on trial because of that.

