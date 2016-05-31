### Solved by superkojiman

Hey it's a MUD! Exploring this by hand isn't feasible since there's a timer of some sort that kicks us out if we're not fast enough. So I wrote a python script that would do the navigation for me. Not much to this really other than just exploring. 

```
#!/usr/bin/env python
from pwn import *

r = remote("p.tjctf.org", 8006)

# north, south, east, west, up, down, examine item

print r.recv()

print "sending: south"
r.sendline("south")
print r.recv()

print "sending: examine item"
r.sendline("examine paper")
print r.recv()

print "sending: south"
r.sendline("south")
print r.recv()

print "sending: up"
r.sendline("up")
print r.recv()

print "sending: west"
r.sendline("west")
print r.recv()

print "sending: examine item"
r.sendline("examine uvlight")
print r.recv()

print "sending: east"
r.sendline("east")
print r.recv()

print "sending: down"
r.sendline("down")
print r.recv()

print "sending: examine item"
r.sendline("examine rug")
print r.recv()

print "sending: examine item"
r.sendline("examine cashregister")
print r.recv()

print "sending: north"
r.sendline("north")
print r.recv()

print "sending: examine item"
r.sendline("examine paper")
print r.recv()

print "sending: north"
r.sendline("north")
print r.recv()

print "sending: north"
r.sendline("north")
print r.recv()


print "sending: up"
r.sendline("up")
print r.recv()

print "sending: west"
r.sendline("west")
print r.recv()

print "sending: west"
r.sendline("west")
print r.recv()

# town
print "sending: west"
r.sendline("west")
print r.recv()

print "sending: examine item"
r.sendline("examine journal2")
print r.recv()

print "sending: east"
r.sendline("east")
print r.recv()

# town
print "sending: east"
r.sendline("east")
print r.recv()

print "sending: east"
r.sendline("east")
print r.recv()

print "sending: examine item"
r.sendline("examine paintedeye")
print r.recv()

print "sending: down"
r.sendline("down")
print r.recv()

print "sending: examine item"
r.sendline("examine book")
print r.recv()


print "sending: up"
r.sendline("up")
print r.recv()

print "sending: west"
r.sendline("west")
print r.recv()

print "sending: west"
r.sendline("west")
print r.recv()

print "sending: south"
r.sendline("south")
print r.recv()

print "sending: examine item"
r.sendline("examine tree")
print r.recv()
```

As I went along and examined things, I started to unlock pieces of the flag: 

* rug:                tjctf{y0u_m1ght_h@v3_
* paper w/uvlight:    YOUR_A1M_IS_G3TT1NG_B3TT3R}
* journal2:           b33n_m1$sing_t3h_
* tree:               fl4g_but_

And so I ended up with:

tjctf{y0u_m1ght_h@v3_b33n_m1$sing_t3h_fl4g_but_YOUR_A1M_IS_G3TT1NG_B3TT3R}

