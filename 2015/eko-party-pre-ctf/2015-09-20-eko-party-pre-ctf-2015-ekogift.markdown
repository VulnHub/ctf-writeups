### Solved by superkojiman

We're given a 32-bit binary. Load it in IDA and find the string "Great your flag is %s\n". The xref is 0x00401CA5. Examine the flowchart and it's just a bunch of if statements that check if each char in the string matches the expected char. Convert the hex value of the expected char to it's letter equivalent to get the flag **EKO{this_is_a_gift}**

