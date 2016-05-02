### Solved by Swappage & bitvijays

In For1 we were provided with a memory dump, and we had to recover a flag.

we started loading the memory dump in Volatility, and confirmed that we were dealing with a Windows 10 x64 memory dump.

From the ground up we gave a quick look at the process list, and the only thing we could spot were two processes running in the context of a logged in user
- notepad.exe
- mspaint.exe

bitvijays remembered of an article explaining how it was possible to recover image data from a memory dump, the process was fairly simple, it involved the dump of the interested process memory and opening the resulting file in gimp as raw data.

From there it was just hand work, search through the offsets for something that would make sense.

Here is when we figured out that volatility doesn't like windows 10 that much, in fact the memdump plugin would provide a huge raw memory file of 1.9GB that was definitely too big for a mspaint process.

Since we are playing google CTF, we decided to revert to a google project: and here comes rekall \o/

Rekall really managed to deal with a windows 10 memory dump much more reliably, and was much faster then volatility in processing the dump (and hey, it also has a nice interactive shell and colored text!!)

    [1] dump1.raw 22:20:57> pslist
    ----------------------> pslist()g Address Ranges 0x0 |
      _EPROCESS            Name          PID   PPID   Thds    Hnds    Sess  Wow64           Start                     Exit          
    -------------- -------------------- ----- ------ ------ -------- ------ ------ ------------------------ ------------------------
    0xe00032553780 System                   4      0    126        -      - False  2016-04-04 16:12:33Z     -                       
    0xe00033f1f780 sihost.exe              92    796     10        -      1 False  2016-04-04 16:12:37Z     -                       
    0xe0003389c040 smss.exe               268      4      2        -      - False  2016-04-04 16:12:33Z     -                       
    0xe000349285c0 taskhostw.exe          332    796     10        -      1 False  2016-04-04 16:17:40Z     -                       
    0xe0003381b080 csrss.exe              344    336      8        -      0 False  2016-04-04 16:12:33Z     -                       
    0xe000325ba080 wininit.exe            404    336      1        -      0 False  2016-04-04 16:12:34Z     -                       
    0xe000325c7080 csrss.exe              412    396      9        -      1 False  2016-04-04 16:12:34Z     -                       
    0xe00033ec6080 winlogon.exe           460    396      2        -      1 False  2016-04-04 16:12:34Z     -                       
    0xe00033efb440 services.exe           484    404      3        -      0 False  2016-04-04 16:12:34Z     -                       
    0xe00033f08080 lsass.exe              492    404      6        -      0 False  2016-04-04 16:12:34Z     -                       
    0xe00033ec5780 svchost.exe            580    484     16        -      0 False  2016-04-04 16:12:34Z     -                       
    0xe00034377780 svchost.exe            608    484     17        -      0 False  2016-04-04 16:12:34Z     -                       
    0xe00034202280 svchost.exe            612    484      9        -      0 False  2016-04-04 16:12:34Z     -                       
    0xe00034ade080 svchost.exe            628    484      1        -      1 False  2016-04-04 16:14:43Z     -                       
    0xe000341cb640 dwm.exe                712    460      8        -      1 False  2016-04-04 16:12:34Z     -                       
    0xe00034222780 svchost.exe            796    484     45        -      0 False  2016-04-04 16:12:34Z     -                       
    0xe000342a7780 VBoxService.ex         828    484     10        -      0 False  2016-04-04 16:12:34Z     -                       
    0xe000342ad780 svchost.exe            844    484      8        -      0 False  2016-04-04 16:12:34Z     -                       
    0xe000342c0080 svchost.exe            852    484      6        -      0 False  2016-04-04 16:12:34Z     -                       
    0xe000342dd780 svchost.exe            892    484     18        -      0 False  2016-04-04 16:12:34Z     -                       
    0xe000342bc780 svchost.exe            980    484     17        -      0 False  2016-04-04 16:12:34Z     -                       
    0xe000343e7780 spoolsv.exe           1072    484      8        -      0 False  2016-04-04 16:12:34Z     -                       
    0xe000343e9780 svchost.exe           1092    484     23        -      0 False  2016-04-04 16:12:35Z     -                       
    0xe0003442a780 rundll32.exe          1148    796      1        -      0 False  2016-04-04 16:12:35Z     -                       
    0xe00034494780 CompatTelRunne        1224   1148      9        -      0 False  2016-04-04 16:12:35Z     -                       
    0xe00034495780 svchost.exe           1276    484     10        -      0 False  2016-04-04 16:12:35Z     -                       
    0xe0003259b3c0 taskhostw.exe         1532    796      9        -      1 False  2016-04-04 16:12:37Z     -                       
    0xe0003461d780 svchost.exe           1564    484      5        -      0 False  2016-04-04 16:12:35Z     -                       
    0xe000345da780 wlms.exe              1616    484      2        -      0 False  2016-04-04 16:12:35Z     -                       
    0xe00034623780 MsMpEng.exe           1628    484     24        -      0 False  2016-04-04 16:12:35Z     -                       
    0xe00034b08780 OneDrive.exe          1692   2336     10        -      1 True   2016-04-04 16:12:55Z     -                       
    0xe00033e00780 svchost.exe           1772    484      3        -      0 False  2016-04-04 16:12:37Z     -                       
    0xe000343b2340 cygrunsrv.exe         1832    484      4        -      0 False  2016-04-04 16:12:35Z     -                       
    0xe0003479b780 cygrunsrv.exe         1976   1832      0        -      0 False  2016-04-04 16:12:36Z     2016-04-04 16:12:36Z    
    0xe000347aa780 conhost.exe           2004   1976      2        -      0 False  2016-04-04 16:12:36Z     -                       
    0xe0003472b080 notepad.exe           2012   2336      1        -      1 False  2016-04-04 16:14:49Z     -                       
    0xe000347c1080 sshd.exe              2028   1976      3        -      0 False  2016-04-04 16:12:36Z     -                       
    0xe000339d4340 NisSrv.exe            2272    484      6        -      0 False  2016-04-04 16:12:38Z     -                       
    0xe000336e8780 userinit.exe          2312    460      0        -      1 False  2016-04-04 16:12:38Z     2016-04-04 16:13:04Z    
    0xe000336e3780 explorer.exe          2336   2312     31        -      1 False  2016-04-04 16:12:38Z     -                       
    0xe0003374f780 RuntimeBroker.        2456    580      6        -      1 False  2016-04-04 16:12:38Z     -                       
    0xe00033a39080 SearchIndexer.        2664    484     13        -      0 False  2016-04-04 16:12:39Z     -                       
    0xe00033a79780 ShellExperienc        2952    580     41        -      1 False  2016-04-04 16:12:39Z     -                       
    0xe000349e4780 WmiPrvSE.exe          3032    580      6        -      0 False  2016-04-04 16:16:37Z     -                       
    0xe00033b57780 SearchUI.exe          3144    580     38        -      1 False  2016-04-04 16:12:40Z     -                       
    0xe000348c6780 VBoxTray.exe          3324   2336     10        -      1 False  2016-04-04 16:12:55Z     -                       
    0xe00033e1d780 DismHost.exe          3636   1224      2        -      0 False  2016-04-04 16:12:47Z     -                       
    0xe000348e9780 svchost.exe           3992    484      6        -      0 False  2016-04-04 16:12:52Z     -                       
    0xe00034b0f780 mspaint.exe           4092   2336      3        -      1 False  2016-04-04 16:13:21Z     -                       
                     Out<1> Plugin: pslist                       
    [1] dump1.raw 22:21:17>

by dumping the mspaint.exe process memory this time we get a mich lighter file of 34MB

    [1] dump1.raw 22:22:49> memdump pid=4092
    ----------------------> memdump(pid=4092)
    **************************************************
    Writing 0xe00034b0f780 mspaint.exe  4092 to mspaint.exe_4092.dmp
                     Out<6> Plugin: memdump                      
    [1] dump1.raw 22:24:11>

and opening the file in gimp, by browsing across the offset we could spot the flag.

![](/images/2016/google-ctf/for1/flag.png)


note to self: next time when dealing with windows 10, use rekall instead of volatility.
