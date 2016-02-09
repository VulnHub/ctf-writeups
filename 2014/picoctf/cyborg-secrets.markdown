### Solved by superkojiman

Cyborg Secrets is an 80 point reverse engineering challenge. 

> You found a password protected binary on the cyborg relating to its defensive security systems. Find the password and get the shutdown code! You can find it on the shell server at /home/cyborgsecrets/cyborg-defense or you can download it here.

This is an easy one. Running the binary prompts us for a password:

```
# ./cyborg_defense 
______               _       _             _____                  
|  _  \             | |     | |           /  __ \                 
| | | |__ _  ___  __| | __ _| |_   _ ___  | /  \/ ___  _ __ _ __  
| | | / _` |/ _ \/ _` |/ _` | | | | / __| | |    / _ \| '__| '_ \ 
| |/ / (_| |  __/ (_| | (_| | | |_| \__ \ | \__/\ (_) | |  | |_) |
|___/ \__,_|\___|\__,_|\__,_|_|\__,_|___/  \____/\___/|_|  | .__/ 
                                                           | |    
                                                           |_|
Please include a password command line argument.
```

We can find the password using the strings command:

```
# strings cyborg_defense 
/lib/ld-linux.so.2
libc.so.6
_IO_stdin_used
puts
putchar
printf
strcmp
__libc_start_main
__gmon_start__
GLIBC_2.0
PTRh0
[^_]
ZogH
TODO: REMOVE DEBUG PASSWORD!
DEBUG PASSWORD: 2manyHacks_Debug_Admin_Test
______               _       _             _____                  
|  _  \             | |     | |           /  __ \                 
| | | |__ _  ___  __| | __ _| |_   _ ___  | /  \/ ___  _ __ _ __  
| | | / _` |/ _ \/ _` |/ _` | | | | / __| | |    / _ \| '__| '_ \ 
| |/ / (_| |  __/ (_| | (_| | | |_| \__ \ | \__/\ (_) | |  | |_) |
|___/ \__,_|\___|\__,_|\__,_|_|\__,_|___/  \____/\___/|_|  | .__/ 
                                                           | |    
                                                           |_|
Please include a password command line argument.
Password: %s
2manyHacks_Debug_Admin_Test
Access Denied
Authorization successful.
;*2$"
```

Password is 2manyHacks_Debug_Admin_Test. Passing that as an argument to the binary gives us the shutdown code:

```
# ./cyborg_defense 2manyHacks_Debug_Admin_Test
______               _       _             _____                  
|  _  \             | |     | |           /  __ \                 
| | | |__ _  ___  __| | __ _| |_   _ ___  | /  \/ ___  _ __ _ __  
| | | / _` |/ _ \/ _` |/ _` | | | | / __| | |    / _ \| '__| '_ \ 
| |/ / (_| |  __/ (_| | (_| | | |_| \__ \ | \__/\ (_) | |  | |_) |
|___/ \__,_|\___|\__,_|\__,_|_|\__,_|___/  \____/\___/|_|  | .__/ 
                                                           | |    
                                                           |_|
Password: 2manyHacks_Debug_Admin_Test
Authorization successful.
403-shutdown-for-what
```

The flag is **403-shutdown-for-what**
