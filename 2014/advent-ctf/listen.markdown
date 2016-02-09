### Solved by bitvijays

This Misc challenge provides you a wav file.

>I couldn't listen it, can you?

Playing the file with any GUI player resulted in the hanging of the application. So, we tried to play the file using a commandline player cvlc which gave us the hint what might be wrong.
``` plain
bitvijays@kali:~/Desktop/Advent$ cvlc listen.wav 
VLC media player 2.0.3 Twoflower (revision 2.0.2-93-g77aa89e)
[0x7f3b6c004768] dummy interface: using the dummy interface module...
[0xaac8e8] main audio output error: too low audio sample frequency (1)
[0xa9cb38] main decoder error: failed to create audio output
```
Checking for the listen.wav headers and WAV file headers.

```
bitvijays@kali:~/Desktop/Advent$ hexdump -C listen.wav | more 
00000000  52 49 46 46 6e 02 05 00  57 41 56 45 66 6d 74 20  |RIFFn...WAVEfmt |
00000010  10 00 00 00 01 00 01 00  01 00 00 00 44 ac 00 00  |........"V..D...|
00000020  02 00 10 00 64 61 74 61  4a 02 05 00 00 00 00 00  |....dataJ.......|
```

![](/images/2014/advent/wave-bytes.gif)

If we see everything is more or less correct but the sample rate is 01 00, which as suggested by vlc is too low audio sample frequency. Changing it to "22 56" i.e 22050 and playing it gives us the flag.
```
bitvijays@kali:~/Desktop/Advent$ hexdump -C listen.wav | more 
00000000  52 49 46 46 6e 02 05 00  57 41 56 45 66 6d 74 20  |RIFFn...WAVEfmt |
00000010  10 00 00 00 01 00 01 00  22 56 00 00 44 ac 00 00  |........"V..D...|
00000020  02 00 10 00 64 61 74 61  4a 02 05 00 00 00 00 00  |....dataJ.......|
```

***The flag is ADCTF_SOUNDS_GOOD***
