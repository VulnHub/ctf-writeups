### Solved by superkojiman

This is an ARM binary. I decompiled it using [Retargetable Decompiler](https://retdec.com/decompilation/) to get the following for main():

```c
// Address range: 0x88e8 - 0x8a9c
int main(int argc, char ** argv) {
    int32_t * v1;
    memset((int32_t *)&v1, 0, 1024);
    printf("Please, enter your encoded password: ");
    fgets((char *)&v1, 1024, (struct _struct__IO_FILE *)g1);
    strlen((char *)&v1);
    v1 = NULL;
    int32_t size = Base64decode_len((char *)&v1); // 0x898c
    int32_t * mem = malloc(size); // 0x8998
    Base64decode((int32_t)mem, (struct struct_0 *)v1);
    int32_t memcmp_rc = memcmp(mem, (int32_t *)"PASS_QIV1qyLR0hFEQU5KCbfm3Hok5V0VmpinCWseVd2X", size); // 0x89c8
    int32_t result = memcmp_rc;
    if (memcmp_rc == 0) {
        // 0x89d8
        strlen((char *)&v1);
        MD5();
        printf("Great! the flag is EKO{");
        // branch -> 0x8a20
        for (uint32_t i = 0; i < 16; i++) {
            // 0x8a20
            char v2;
            printf("%02x", (int32_t)v2);
            result = i + 1;
            // PHI copies at the loop end
            // loop 0x8a20 end
        }
        // 0x8a60
        puts("}");
        // branch -> 0x8a74
    } else {
        // 0x8a6c
        puts("Access denied");
        // branch -> 0x8a74
    }
    // 0x8a74
    return result;
}
```

We can now see that it's expecting a Base64 encoded password. There's an interesting bit here:

```c
int32_t memcmp_rc = memcmp(mem, (int32_t *)"PASS_QIV1qyLR0hFEQU5KCbfm3Hok5V0VmpinCWseVd2X", size); // 0x89c8
```

So it almost seemed like all we had to do was Base64 encode PASS_QIV1qyLR0hFEQU5KCbfm3Hok5V0VmpinCWseVd2X and we would get the flag. As it turned out, this didn't work. So I stepped through the binary in gdb to see what was going on.

I set a breakpoint at 0x89c8 which was the call to memcmp(). memcmp() takes three arguments; the first two are the strings to compare, and the third is the length to compare. The first argument is stored in r0, and the second in r1. Let's take a look: 

```
gdb$ x/s $r0
0x11008:    "PASS_QIV1qyLR0iFEQU5KCbgm3Hok5V0VmphnCWseVd2X"
gdb$ x/s $r1
0x8c38: "PASS_QIV1qyLR0hFEQU5KCbfm3Hok5V0VmpinCWseVd2X"
```

They look the same, but they're not. 

PASS_QIV1qyLR0**i**FEQU5KCb**g**m3Hok5V0Vmp**h**nCWseVd2X
PASS_QIV1qyLR0**h**FEQU5KCb**f**m3Hok5V0Vmp**i**nCWseVd2X

r1 is what our decoded password looks like, but it's comparing it against r0, which is what the decoded password the binary is expecting. So we just Base64 encode r0 which becomes "UEFTU19RSVYxcXlMUjBpRkVRVTVLQ2JnbTNIb2s1VjBWbXBobkNXc2VWZDJY".

Pass this as the encoded password and we get our flag: 

```
# ./decoder 
Please, enter your encoded password: UEFTU19RSVYxcXlMUjBpRkVRVTVLQ2JnbTNIb2s1VjBWbXBobkNXc2VWZDJY
Great! the flag is EKO{4fa8c8eac431266a25f56a297a73c334}
```

Done!
