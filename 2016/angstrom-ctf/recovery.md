### Solved by superkojiman

Simple forensics challenge. Just download the binary and run strings on it to get the flag.

```
root@angstrom-ctf:~/work# strings recovered.img |grep -i flag
flag{waw_ctrl_f_was_sooper_afective}
?Flag
```
