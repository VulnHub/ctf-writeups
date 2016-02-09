### Solved by superkojiman

Police Records is a 140 point challenge. We're given the following desctipion:

> A Theseus double agent has infiltrated the police force but the police won't give you access to their directory information. Thankfully you found the source code to their directory server. Write a client to interact with the server and find the duplicated badge number! The flag is the badge number.
The server is running on vuln2014.picoctf.com:21212.

Let's take a look at the code for the directory server: 

```python
#!/usr/bin/python

""" Police Directory Service v1.2"""

import socketserver
import sys
import struct
import random
import json

from os import urandom

HOST = 'localhost'
PORT = 21212

def xor(buf, key):
    """ Repeated key xor """

    encrypted = []
    for i, cr in enumerate(buf):
        k = key[i % len(key)]
        encrypted += [cr ^ k]
    return bytes(encrypted)

def secure_pad(buf):
    """ Ensure message is padded to block size. """
    key = urandom(5)
    buf = bytes([0x13, 0x33, 0x7B, 0xEE, 0xF0]) + buf
    buf = buf + urandom(16 - len(buf) % 16)
    enc = xor(buf, key)
    return enc

def remove_pad(buf):
    """ Removes the secure padding from the msg. """
    if len(buf) > 0 and len(buf) % 16 == 0:
        encrypted_key = buf[:5]
        key = xor(encrypted_key, bytes([0x13, 0x33, 0x7B, 0xEE, 0xF0]))
        dec = xor(buf, key)
        return dec[5:20]

def generate_cookie():
    """ Generates random transaction cookie. """
    cookie = random.randrange(1, 1e8)
    return cookie


class TCPConnectionHandler(socketserver.BaseRequestHandler):
    """ TCP Server """

    OFFICERS = json.loads(open("data.json", "r").read())
    client_information = {}

    def get_officer_data(self, entry):
        """ Retrieve binary format of officer. """

        if 0 <= entry and entry < len(self.OFFICERS):
            return json.dumps(self.OFFICERS[entry]).encode("utf-8")
        return None

    def secure_send(self, msg):
        """ Sends msg back to the client securely. """

        cookie = generate_cookie()
        data = struct.pack("!B2L128s", 0xFF, cookie, len(msg), msg)
        encrypted = secure_pad(data)
        self.request.sendall(encrypted)
        return cookie

    def handle(self):
        """ Handle client session"""
        running = True
        cookie = None
        access = False

        while running:
            data = self.request.recv(1024)
            if len(data) > 0:
                if not access:
                    try:
                        code = struct.unpack("!i", data)[0]
                        if code == 0xAA:
                            cookie = self.secure_send(b"WELCOME TO THE POLICE RECORDS DIRECTORY")
                            access = True
                        else:
                            raise Exception
                    except Exception as e:
                        raise e
                        self.request.sendall(b"ACCESS CODE DENIED")
                        running = False
                else:
                    if len(data) % 16 == 0:
                        decrypted = remove_pad(data)
                        try:
                            magic, user_cookie, badge, cmd, entry = \
                                    struct.unpack("!B2LHL", decrypted)
                            if magic != 0xFF or user_cookie != cookie:
                                self.request.sendall(b"INSECURE REQUEST")
                                running = False
                            else:
                                if cmd == 1:
                                    officer = self.get_officer_data(entry)
                                    if officer:
                                        cookie = self.secure_send(officer)
                                    else:
                                        cookie = self.secure_send(b"INVALID ENTRY -- OFFICER DOES NOT EXIST")
                                else:
                                    cookie = self.secure_send(b"INVALID COMMAND")
                        except Exception as e:
                            raise e
                            self.request.sendall(b"MALFORMED REQUEST")
                            running = False
            else:
                running = False
        self.request.close()

class Server(socketserver.ThreadingMixIn, socketserver.TCPServer):
    daemon_threads = True
    allow_reuse_address = True

    def __init__(self, server_address, RequestHandlerClass):
        socketserver.TCPServer.__init__(self, server_address, RequestHandlerClass)

if __name__ == "__main__":
    server = Server((HOST, PORT), TCPConnectionHandler)
    try:
        server.serve_forever()
    except KeyboardInterrupt:
        sys.exit(0)
```

The goal is to write a client that will interact with this directory server. So let's break down what the server is doing. The handle() function handles the client connections so let's start with that. 

```python
data = self.request.recv(1024)
if len(data) > 0:
    if not access:
        try:
            code = struct.unpack("!i", data)[0]
            if code == 0xAA:
                cookie = self.secure_send(b"WELCOME TO THE POLICE RECORDS DIRECTORY")
                access = True
            else:
                raise Exception
        except Exception as e:
            raise e
            self.request.sendall(b"ACCESS CODE DENIED")
            running = False
```

The server reads in 1024 bytes of input from the client and checks to see if we have an access code. If we don't, it waits for us to send one. An access code is required to interact with the server, so this is the first check that we need to bypass. Fortunately it's relatively simple, it's waiting for us to send 0xAA in network byte order:

```python
code = struct.unpack("!i", data)[0]
if code == 0xAA:
    cookie = self.secure_send(b"WELCOME TO THE POLICE RECORDS DIRECTORY")
    access = True
```

If we set the access code, then the server replies with a welcome message. So we just need to reverse this by using struct.pack(). Here's how it looks:

```python
#!/usr/local/bin/python3
import struct, socket, sys

ip = "vuln2014.picoctf.com"
port = 21212

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect((ip, port))

# send access code 0xAA in network byte order to server
s.send(struct.pack("!i", 0xAA)
buf = s.recv(1024)
print("recv:", buf)
```

Note that I'm using python3 for this. The reason is that the directory_server.py is using python3 as well, and this became apparent later on when I was examining the xor() function. The xor() function performs an xor on two strings using the ^ operator, which only works on python3. Ok, back to the client, let's run it:

```python
$ ./police_client.py
recv: b'\x10\x08\xad\xb6\xde\xfc?\xb6\x16G\x03;\xd6\x7fyFw\x95\x17cF\x1b\x82\x17\x0eWs\x93x~Lw\x9f\x1bk#i\x93\x1baQ\x7f\x85xjJi\x93\x1bzLi\x8fX.\x03;\xd6X.\x03;\xd6X.\x03;\xd6X.\x03;\xd6X.\x03;\xd6X.\x03;\xd6X.\x03;\xd6X.\x03;\xd6X.\x03;\xd6X.\x03;\xd6X.\x03;\xd6X.\x03;\xd6X.\x03;\xd6X.\x03;\xd6X.\x03;\xd6X.\x03;\xd6X.\x03;\xd6X.\x03;\x9e\xe5'
``` 

Great we got something back, but it looks nothing like the welcome message we were expecting. Let's take a look at what the server is sending us. 

```python
cookie = self.secure_send(b"WELCOME TO THE POLICE RECORDS DIRECTORY")
```

If we set the access code correctly, which we did, then it calls secure_send() and passes it the string to send back to the client. Here's what it looks like:

```python
def secure_send(self, msg):
    """ Sends msg back to the client securely. """

    cookie = generate_cookie()
    data = struct.pack("!B2L128s", 0xFF, cookie, len(msg), msg)
    encrypted = secure_pad(data)
    self.request.sendall(encrypted)
    return cookie
```

First it creates a cookie using the generate_cookie() function. generate_cookie() just creates a random value between 1 and 488. Next, a data structure is created containing a byte 0xFF (referred to as magic in the server code), the cookie, the length of the message, and the message itself. This data structure is then passed to a function secure_pad():

```python
def secure_pad(buf):
    """ Ensure message is padded to block size. """
    key = urandom(5)
    buf = bytes([0x13, 0x33, 0x7B, 0xEE, 0xF0]) + buf
    buf = buf + urandom(16 - len(buf) % 16)
    enc = xor(buf, key)
    return enc
```

This pads the data structure to block size and then encrypts it using xor() before returning it. Fortunately we don't have to reverse this as it's already done for us in the remove_pad() function:

```python
def remove_pad(buf):
    """ Removes the secure padding from the msg. """
    if len(buf) > 0 and len(buf) % 16 == 0:
        encrypted_key = buf[:5]
        key = xor(encrypted_key, bytes([0x13, 0x33, 0x7B, 0xEE, 0xF0]))
        dec = xor(buf, key)
        return dec[5:20]
```

So now we know the server is encrypting a data structure containing the message and sending it back to us. We need to decrypt it using the remove_pad() function, and then unpack the decrypted data structure to get the message. We'll copy the function secure_pad(), remove_pad(), and xor() to our client code and unpack the server's reply:

```python
#!/usr/local/bin/python3
import struct, socket, sys
from os import urandom

def xor(buf, key):
    """ Repeated key xor """
    encrypted = []
    for i, cr in enumerate(buf):
        k = key[i % len(key)]
        encrypted += [cr ^ k]

    return bytes(encrypted)


def remove_pad(buf):
    """ Removes the secure padding from the msg. """
    if len(buf) > 0 and len(buf) % 16 == 0:
        encrypted_key = buf[:5]
        key = xor(encrypted_key, bytes([0x13, 0x33, 0x7B, 0xEE, 0xF0]))
        dec = xor(buf, key)

        print("remove_pad() key    :", key)
        print("remove_pad() dec msg:", dec)
        print("\n")
        return dec

def secure_pad(buf):
    """ Ensure message is padded to block size. """
    key = urandom(5)
    buf = bytes([0x13, 0x33, 0x7B, 0xEE, 0xF0]) + buf
    buf = buf + urandom(16 - len(buf) % 16)
    enc = xor(buf, key)
    return enc

ip = "vuln2014.picoctf.com"
port = 21212

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect((ip, port))

# send access code 0xAA in network byte order to server
s.send(struct.pack("!i", 0xAA))
buf = s.recv(1024)
print("recv:", buf)
print("\n")

dec = remove_pad(buf)
magic, cookie, msg_len, msg = struct.unpack("!B2L6s", dec[5:20])

print("magic  :", magic)
print("cookie :", cookie)
print("msg_len:", msg_len)
print("dec msg:", msg)
```

I modified the remove_pad() function slightly to return the entire decrypted data structure instead of just a part of it, and I've added some debugging statements as well. Running it returns:

```text
$ ./police_client.py
recv: b"\xa6\xec\x19\x93\nJ\xdc\xb9\xd5\xb9\xb5\xdfbZ\xad\xf0\x93!2\xb7\xf0\xff62\xda\xe1\x97']\xaa\xfa\x93+>\xbf\x95\x8d'>\xb5\xe7\x9b1]\xbe\xfc\x8d'>\xae\xfa\x8d;}\xfa\xb5\xdfb}\xfa\xb5\xdfb}\xfa\xb5\xdfb}\xfa\xb5\xdfb}\xfa\xb5\xdfb}\xfa\xb5\xdfb}\xfa\xb5\xdfb}\xfa\xb5\xdfb}\xfa\xb5\xdfb}\xfa\xb5\xdfb}\xfa\xb5\xdfb}\xfa\xb5\xdfb}\xfa\xb5\xdfb}\xfa\xb5\xdfb}\xfa\xb5\xdfb}\xfa\xb5\xdfb}\xfa\xb5\xdfb}\xfa\xb5\xdf'!"


remove_pad() key    : b'\xb5\xdfb}\xfa'
remove_pad() dec msg: b"\x133{\xee\xf0\xff\x03\xdb\xa8C\x00\x00\x00'WELCOME TO THE POLICE RECORDS DIRECTORY\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00E\\"


magic  : 255
cookie : 64727107
msg_len: 39
dec msg: b'WELCOM'
```

Ok now we're getting somewhere. We can decrypt the replies from the server. Now we need to send it a request to return information about officers in the directory. The code responsible for this in directory_server.py is:

```python
if len(data) % 16 == 0:
    decrypted = remove_pad(data)
    try:
        magic, user_cookie, badge, cmd, entry = \
                struct.unpack("!B2LHL", decrypted)
        if magic != 0xFF or user_cookie != cookie:
            self.request.sendall(b"INSECURE REQUEST")
            running = False
        else:
            if cmd == 1:
                officer = self.get_officer_data(entry)
                if officer:
                    cookie = self.secure_send(officer)
                else:
                    cookie = self.secure_send(b"INVALID ENTRY -- OFFICER DOES NOT EXIST")
            else:
                cookie = self.secure_send(b"INVALID COMMAND")
```

So once we've set the access code, we can send a query request. We see that the server will remove the padding and decrypt the data we send, and then unpack it into several variables. It makes sure that magic and cookie match the ones that it sent us along with the welcome message. If it doesn't match, the server refuses to send us anything. If the magic and cookie check passes, and the command is set to 1, then the server will take the entry number we provide, grab the officer's records associated with it, and send it back to us. 

The server unpacks the data using the format characters "!B 2L H L" so we need to pack a byte (0xFF), two unsigned longs (cookie, and badge), an unsigned short (cmd), and an unsigned long (entry). Here's the updated script to retrieve one record:

```python
.
.
.
# send access code 0xAA in network byte order to server
s.send(struct.pack("!i", 0xAA))
buf = s.recv(1024)

dec = remove_pad(buf)
magic, cookie, msg_len, msg = struct.unpack("!B2L6s", dec[5:20])

# build the request and send it
print("Retrieving entry 1 record using dummy badge 12345678")
data = struct.pack("!B2LHL", magic, cookie, 1234578, 1, 1)
enc = secure_pad(data)
s.send(enc)

# get a response and decrypt it
buf = s.recv(1024)
dec = remove_pad(buf)
magic, cookie, msg_len, msg = struct.unpack("!B2L6s", dec[5:20])

print("magic  :", magic)
print("cookie :", cookie)
print("msg_len:", msg_len)
print("dec msg:", msg)
```

We're sending a dummy badge number 12345678 and entry ID 1 to the server. Here's what happens when we run the updated script:

```text
$ ./police_client.py
remove_pad() key    : b';j\xe4\x8cF'
remove_pad() dec msg: b"\x133{\xee\xf0\xff\x04\xaf\x8b\xfa\x00\x00\x00'WELCOME TO THE POLICE RECORDS DIRECTORY\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xa5\xef"


Retrieving entry 1 record using dummy badge 12345678
remove_pad() key    : b'\x90\xc3\x92\x0c\x98'
remove_pad() dec msg: b'\x133{\xee\xf0\xff\x02}y\xd1\x00\x00\x00U{"ACTIVE": false, "NAME": "Aisha Elton", "TITLE": "SECURITY GUARD", "BADGE": 4219177}\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x159'


magic  : 255
cookie : 41777617
msg_len: 85
dec msg: b'{"ACTI'
```

Excellent, we've gotten information about the first officer, which includes the badge number in the dec variable. Using the msg_len, we can extract just the badge number itself:

```python
badge = dec[msg_len+6:msg_len+13]
print("badge:", badge)
```

This gives us:

```text
.
.
.
Retrieving entry 1 record using dummy badge 12345678
remove_pad() key    : b'\xaf\x11Z\xc8\x99'
remove_pad() dec msg: b'\x133{\xee\xf0\xff\x00 \x81|\x00\x00\x00U{"ACTIVE": false, "NAME": "Aisha Elton", "TITLE": "SECURITY GUARD", "BADGE": 4219177}\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x004'


magic  : 255
cookie : 2130300
msg_len: 85
dec msg: b'{"ACTI'
badge: b'4219177'
```

Here's the completed script, which loops through entry IDs 1 to 1000 and stores each badge in a list. As a new badge is extracted, it checks if that badge ID is already in the list. If it isn't, it inserts it, and if it is, then we've found the duplicate badge, which is the flag. 

```python
#!/usr/local/bin/python3
import struct, socket, sys
from os import urandom

def xor(buf, key):
    """ Repeated key xor """
    encrypted = []
    for i, cr in enumerate(buf):
        k = key[i % len(key)]
        encrypted += [cr ^ k]

    return bytes(encrypted)


def remove_pad(buf):
    """ Removes the secure padding from the msg. """
    if len(buf) > 0 and len(buf) % 16 == 0:
        encrypted_key = buf[:5]
        key = xor(encrypted_key, bytes([0x13, 0x33, 0x7B, 0xEE, 0xF0]))
        dec = xor(buf, key)
        return dec

def secure_pad(buf):
    """ Ensure message is padded to block size. """
    key = urandom(5)
    buf = bytes([0x13, 0x33, 0x7B, 0xEE, 0xF0]) + buf
    buf = buf + urandom(16 - len(buf) % 16)
    enc = xor(buf, key)
    return enc

ip = "vuln2014.picoctf.com"
port = 21212
badges = []

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect((ip, port))

# send access code 0xAA in network byte order to server
s.send(struct.pack("!i", 0xAA))
buf = s.recv(1024)

dec = remove_pad(buf)
magic, cookie, msg_len, msg = struct.unpack("!B2L6s", dec[5:20])


for entry in range(1, int(sys.argv[1])):
    # build the request and send it
    data = struct.pack("!B2LHL", magic, cookie, 1234578, 1, entry)
    enc = secure_pad(data)
    s.send(enc)

    # get a response and decrypt it
    buf = s.recv(1024)
    dec = remove_pad(buf)
    magic, cookie, msg_len, msg = struct.unpack("!B2L6s", dec[5:20])

    # extract the badge
    badge = dec[msg_len+6:msg_len+13]
    print("entry: %d, badge: %s" %(entry, badge))

    # do we have a duplicate?
    if badge in badges:
        print("found duplicate badge:", badge)
        break

    # badge has no found duplicate yet, add to the list
    badges.append(badge)
```

When we run this, it receives a listing of officer's badges, until it finds the duplicate badge:

```
$ ./police_client.py 1000
entry: 1, badge: b'4219177'
entry: 2, badge: b'3260685'
entry: 3, badge: b'7083824'
entry: 4, badge: b'4700315'
.
.
.
entry: 899, badge: b' 840952'
entry: 900, badge: b'8929869'
entry: 901, badge: b'1703131'
entry: 902, badge: b'6541803'
entry: 903, badge: b'1430758'
found duplicate badge: b'1430758'
```

The flag is **1430758**
