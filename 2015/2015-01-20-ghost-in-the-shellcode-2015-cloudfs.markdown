### Solved by Swappage

This CTF was what I'd call *an humbling experience*; it was an absolutely great
contest, don't get me wrong, but damn! it was hard!, and since I'm not a CTF
veteran, let me say that I learned an important thing: "There is no limit
to the evilness people can put in their effort of creating challanging puzzles".

Ok but let's get back to this specific challenge.
For 200 points we were asked to extract an hidden key from a pcap file.

The name of the challenge was an hint itself: *cloudfs* made me think about
something that was used to transfer a file, and by opening a pcap all i could see
were some HTTP traffic packets to wikipedia, and loots and loots of ICMP ping
requests/responses.

![](/images/2015/gits/cloudfs/pings.png)

Before diving into the pcap itself i ran through some usual checks you do to spot
obvious things, and after having verified that nothing was appended at the bottom of
the pcap file, and that it wasn't a *polyglot* i started looking closer at the packets
themselves.

Fresh from the [Persistence competition](http://blog.vulnhub.com/2014/09/competition-persistence.html)
I figured out pretty quickly that the icmp packets could have been used as a cover channel
to hide something, and in fact if we look at the payload data from a couple of random
icmp requests we could easly notice that the ping pattern was containing custom data.

![](/images/2015/gits/cloudfs/apples.png)

The interesting things start at packet 1040 in the pcap file, where we can see
the header for a bzip2 file

![](/images/2015/gits/cloudfs/bz2.png)

which was repeating onwards every 4 packets till the end of the pcap file.

I've then tried to extract the data payload of the four interesting packets and use
binwalk to see if it could extract something useful.

    # binwalk -e filtered.pcap

    DECIMAL     HEX         DESCRIPTION
    ----------------------------------------------------------------------------------------------------------------
    20          0x14        bzip2 compressed data, block size = 900k

It was actually detecting and extracting a bzip2 file, the header was there, after
all, but if i tried extracting its content it was informing me that the file was
corrupted.

    # bunzip2 14.bz2

    bunzip2: Data integrity error when decompressing.
    Input file = 14.bz2, output file = 14

    It is possible that the compressed file(s) have become corrupted.
    You can use the -tvv option to test integrity of such files.

    You can use the `bzip2recover' program to attempt to recover
    data from undamaged sections of corrupted files.

    bunzip2: Deleting output file 14, if it exists.

It took me a little while to notice that the pings were sent in an incorrect
order, and that it was necessary to reorder them properly.

In fact, let's look at the icmp packet identifier.

![](/images/2015/gits/cloudfs/identifier.png)

Here it's possible to notice that the identifier are ordered as 13 14 16 15

I've tried to extract the packets payload data separately (after all they were
  only 4 packets) and reassemble them by hand.

    # cat 13.raw 14.raw 15.raw 16.raw > reassembled.raw
    # binwalk reassembled.raw -e

    DECIMAL     HEX         DESCRIPTION
    ----------------------------------------------------------------------------------------------------------------
    20          0x14        bzip2 compressed data, block size = 900k

    # ls -la
    total 64
    drwxr-xr-x 2 root root  4096 Jan 20 20:00 .
    drwxr-xr-x 7 root root 36864 Jan 20 20:00 ..
    -rw-r--r-- 1 root root 24556 Jan 20 20:00 14

    # file 14
    14: POSIX tar archive (GNU)

    # mv 14 14.tar; tar xvf 14.tar
    key
    ping/ping.py

and appearently we got the tar archive containing both the key, as well as the
python script that was used to create the cover channel

    # cat key
    key{WhyWouldYouEverUseThis}


```python
import os,sys,socket,struct,select,time,binascii,logging
import ping_reporter

ping_count = 0
ping_bandwidth = 0
log = ping_reporter.setup_log('Ping')
server_list = ['www.wikipedia.org']

def select_server(log,max_timeout=2):
server = ''
log.notice('selecting server')
maxw = len(max(server_list, key=len))
min_delay = max_timeout * 1000 # seconds -> ms
for x in server_list:
delay = min_delay + 1
try: delay = single_ping(x,max_timeout)
finally:
if delay == None: log.notice('%-*s: timed out'%(maxw,x))
else:             log.notice('%-*s: %05.02fms'%(maxw,x,delay*1000))
if delay != None and delay < min_delay:
min_delay = delay
server = x
log.info('selected server: %s (%.02fms)'%(server,min_delay*1000))
return server

def carry_add(a, b):
c = a + b
return (c & 0xFFFF) + (c >> 16)

def checksum(msg):
s = 0
if len(msg)%2: # pad with NULL
msg = msg + '%c'%0
for i in range(0, len(msg)/2*2, 2):
w = ord(msg[i]) + (ord(msg[i+1]) << 8)
s = carry_add(s, w)
return ~s & 0xFFFF

def build_ping(ID, data):
log.trace('ping::build_ping: ID=%d, bytes=%d'%(ID,len(data)))
if ID == 0: raise Exception('Invalid BlockID (0): many servers will corrupt ID=0 ICMP messages')

data = str(data) # string type, like the packed result

# Header is type (8), code (8), checksum (16), id (16), sequence (16)
icmp_type      = 8 # ICMP_ECHO_REQUEST
icmp_code      = 0 # Can be anything, but reply MUST be 0
icmp_checksum  = 0 # 0 for initial checksum calculation
#   icmp_id        = (ID >> 16) & 0xFFFF
#   icmp_sequence  = (ID <<  0) & 0xFFFF
block_id       = ID # append id & seq for 4-byte identifier

header = struct.pack("bbHI", icmp_type, icmp_code, icmp_checksum, block_id)
icmp_checksum = checksum(header+data)
header = struct.pack("bbHI", icmp_type, icmp_code, icmp_checksum, block_id)

# Return built ICMP message
return header+data

def build_socket(RCVBUF=1024*1024):
# By default, SO_RCVBUF is ~50k (kernel doubles to 114688), which only supports
# ~1k blocks with <1ms timing. Raising this to 1m supports >16k blocks. Unfortunately,
# raising it more does little because we can't read/process the events fast enough, so
# the buffer pretty quickly fills, and then start dropping packets again.
log.trace('ping::build_socket')
icmp = socket.getprotobyname("icmp")
try:
icmp_socket = socket.socket(socket.AF_INET, socket.SOCK_RAW, icmp)
except socket.error, (errno, msg):
if errno == 1: # Operation not permitted
msg = msg + (" (ICMP messages can only be sent from processes running as root)")
raise socket.error(msg)
raise # raise the original error
socket.SO_SNDBUFFORCE = 32
socket.SO_RCVBUFFORCE = 33
icmp_socket.setsockopt(socket.SOL_SOCKET, socket.SO_RCVBUFFORCE, RCVBUF)
icmp_socket.setsockopt(socket.SOL_SOCKET, socket.SO_SNDBUFFORCE, RCVBUF)
return icmp_socket

def time_ping(d_socket, d_addr, ID=1):
log.trace('ping::time_ping: server=%s ID=%d'%(d_addr,ID))
data = struct.pack("d",time.time())
return data_ping(d_socket, d_addr, ID, data)

def data_ping(d_socket, d_addr, ID, data):
log.trace('ping::data_ping: server=%s ID=%d bytes=%d'%(d_addr,ID,len(data)))
send_ping(d_socket, socket.gethostbyname(d_addr), ID, data)

def send_ping(d_socket, d_addr, ID, data):
log.trace('ping::send_ping: server=%s ID=%d bytes=%d'%(d_addr,ID,len(data)))
global ping_count, ping_bandwidth
packet = build_ping(ID,data)
d_socket.sendto(packet, (d_addr, 1))
if 1:
ping_count = ping_count + 1
ping_bandwidth = ping_bandwidth + len(packet)

def parse_ip(packet):
log.trace('ping::parse_ip: bytes=%d'%(len(packet)))
if len(packet) < 20: return None
(verlen,ID,flags,frag,ttl,protocol,csum,src,dst) = struct.unpack('!B3xH4BHII',packet[:20])
ip = dict(  version= verlen >> 4,
  length=  4*(verlen & 0xF),
  ID=      ID,
  flags=   flags >> 5,
  fragment=((flags & 0x1F)+frag),
  ttl=     ttl,
  protocol=protocol,
  checksum=csum,
  src=     src,
  dst=     dst)
  return ip

  def parse_icmp(packet,validate):
  log.trace('ping::parse_icmp: bytes=%d'%(len(packet)))
  if len(packet) < 8: return None
  (type, code, csum, block_id) = struct.unpack('bbHI', packet[:8])
  log.debug('ping::parse_icmp: type=%d code=%d csum=%x ID=%d'%(type,code,csum,block_id))
  icmp = dict(type=type,
    code=code,
    checksum=csum, # calculated big-endian
    block_id=block_id)

    if validate:
    t_header = struct.pack('bbHI',type,code,0,block_id)
    t_csum = checksum(t_header+packet[8:])
    icmp['valid'] = (t_csum == csum)

    return icmp

    def parse_ping(packet,validate=False):
    log.trace('ping::parse_ping: bytes=%d validate=%s'%(len(packet),validate))
    if len(packet) < 20+8+1: return None # require 1 block of data
    ip = parse_ip(packet)
    if not ip:                                return None
    if ip['protocol'] != socket.IPPROTO_ICMP: return None # ICMP
    if ip['version'] != socket.IPPROTO_IPIP:  return None # IPv4
    if ip['length']+8+1 > len(packet):        return None # invalid ICMP header

    packet = packet[ip['length']:]
    icmp = parse_icmp(packet,validate)
    if not icmp:                              return None
    if icmp['type'] != 0:                     return None # not an Echo Reply packet
    if icmp['code'] != 0:                     return None # not a valid Echo Reply packet
    if validate and icmp['valid'] != True:    return None # invalid ICMP checksum

    payload = packet[8:]
    log.debug('ping::parse_ping: valid echo reply w/ ID=%d (%d bytes)'%(icmp['block_id'],len(payload)))
    return dict(ip=ip,icmp=icmp,payload=payload)


    def recv_ping(d_socket, timeout, validate=False):
    d_socket.settimeout(timeout)
    try:
    data,addr = d_socket.recvfrom(2048)
    except socket.timeout:
    return None
    parsed = parse_ping(data,validate)
    if not parsed: return None
    parsed['ID']=parsed['icmp']['block_id']
    parsed['address']=addr
    parsed['raw']=data
    log.debug('ping::recv_ping: ID=%d address=%s bytes=%d'%(parsed['ID'],addr,len(data)))
    return parsed

    def read_ping(d_socket, timeout):
    start = time.time()
    while time.time() - start < timeout:
    msg = recv_ping(d_socket,timeout)
    if msg: return msg
    return None

    def receive_ping(my_socket, ID, timeout):
    timeLeft = timeout
    while True:
    startedSelect = time.time()
    whatReady = select.select([my_socket], [], [], timeLeft)
    if whatReady[0] == []: # Timeout
    return

    timeReceived = time.time()
    howLongInSelect = (timeReceived - startedSelect)
    recPacket, addr = my_socket.recvfrom(1024)
    icmpHeader = recPacket[20:28]
    type, code, checksum, packetID = struct.unpack("bbHI", icmpHeader)
    if packetID == ID:
    bytesInDouble = struct.calcsize("d")
    timeSent = struct.unpack("d", recPacket[28:28 + bytesInDouble])[0]
    return timeReceived - timeSent

    timeLeft = timeLeft - howLongInSelect
    if timeLeft <= 0:
    return 0

    def single_ping(dest_addr, timeout):
    my_socket = build_socket()
    my_ID = os.getpid() & 0xFFFF
    time_ping(my_socket, dest_addr, my_ID)
    delay = receive_ping(my_socket, my_ID, timeout)
    my_socket.close()
    return delay

    def verbose_ping(dest_addr, timeout = 2, count = 4):
    for i in xrange(count):
    log.info("ping %s..." % dest_addr,)
    try:
    delay  =  single_ping(dest_addr, timeout)
    except socket.gaierror, e:
    log.error("failed. (socket error: '%s')" % e[1])
    break

    if delay  ==  None:
    log.info("failed. (timeout within %ssec.)" % timeout)
    else:
    delay  =  delay * 1000
    log.info("get ping in %0.4fms" % delay)
    print


    import os, pwd, grp

    if __name__ == '__main__':
    ping_reporter.start_log(log)
    server = select_server(log,2)

    if 1:
    verbose_ping(server)
    else:
    s = build_socket()
    print 'sending 100 pings...'
    for x in range(1,100):
    data_ping(s,server,x,struct.pack('d',x))
    print 'ping cycled...'
    recv_ping(s,1)
    print '100 pings sent'

```

And this one was one of the simpliest challenges in this contest... so well...
congratulations to all the teams that solved a lot of them :)

