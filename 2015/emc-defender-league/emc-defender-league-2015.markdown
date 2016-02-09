### Solved by bitvijays

Vulnhub-ctf team played in the Hack IM 2015 and we ended up at 22nd position and got a chance to go onsite @ Nullcon Goa. As bitvijays is in India, he represented the team at Goa. It was 3 hour challenge with a test round. However, a lot of time was spent in setting up the VMs and networks. We were provided a virtual machine image running Ubuntu Operating system with some vulnerable services with the correct source code and modified source code. There was also a goodies server hosting flags for extra points. We had to run a player-ident binary which would identify us from others. There were total of 30 rounds, a bot would connect to all the services open at some time and provide you points if the service responded. 100 points for each services.

Everything which could go wrong will go wrong according to Murphy's law. We setup-ed almost everything like setting up of Nessus, OpenVAS scanner, read the hardening guide of NSA for linux etc on my Kali-linux host machine, but when we reached the venue, my ethernet network adapter on Kali linux failed to detect the network. Everything planned failed. Backup plan? Run windows. Windows was terribly slow, but we had no choice. Surprisingly, network on my Windows OS was working fine. we lost all the points in the first 3-4 rounds because of the delay in extraction of 3 GB tar.gz file on Windows with winrar. As everything was extremely slow, we created a snapshot of my PlayerVM after identfying it and whenever my VM stopped responding (someone hacked my VM) just reverted it back quickly. We also solved a goody challenge which was simple using sqlmap which gave us 4000 points.

###Suggestions for Next Year
1. Make sure everything is up and running, even your network adapter.
2. Carry a 8/16 GB USB 3.0 for faster copying of the VM image provided by the organizers.
3. Although, we were provided wireless internet connection, but only had one laptop and ethernet was connected to game network, either you should know how to route data on Windows/Linux. Or Have a small tab/laptop to surf the internet.
4. Create a snapshot of your VMs, just incase anyone hacks into it, just revert it for quick uptime.
5. Scan your network fast and findout the other bad people IPs and use iptables to block traffic from them.
6. The above would atleast defend you for a decent good amount of time and you can work on the services to analyse and exploit them.
7. Check the original source and the modified source to find out where the code has been modified and how it can be exploited.

Just by doing the above, we reached the fifth position.

Overall, It was an awesome experience, met members of Segfault, Reboot and BiOs team. Thanks to everyone in my team. It was definately a learning experience.

