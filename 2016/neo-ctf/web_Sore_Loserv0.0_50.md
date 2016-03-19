###Solved by bitvijays

Sore Loser v0.0
50

The best way to deal with rejection is to become a 1337 Haxor
http://162.243.0.171/rip.php

We were provided by a webpage
```
Dear Joshua,

The Admissions Committee has completed its review of your application. I am very sorry to tell you that you were not admitted to the MIT Class of 2020.

Please understand that this is in no way a judgment of you as a student or as a person, since our decision has more to do with the applicant pool than anything else – many of our applicants are not offered admission simply because we don't have enough space in our entering class. This year we had over 19,000 candidates for fewer than 1,600 offers of admission, from which will come our 1,100 freshmen. Since all of our decisions are made at one time and all available spaces have been committed, all decisions are final.

I wish you the very best in all of your future endeavors.

Sincerely,

Stuart Schmill
Dean of Admissions
```

Checking the View Source for this we find one link bit.ly/22nVxpY
```
        <header>
            <h1 id="logo"><a href="http://bit.ly/22nVxpY" shape="rect">MIT Admissions</a></h1>
            <h4 id="mit-home" class="mh-hide m-hide"><a href="http://mit.edu" title="Go to the MIT Homepage" shape="rect">Massachusetts Institute of Technology</a></h4>
        </header>
```

which results to 
```
Dear Joshua,

The Admissions Committee has completed its review of your application. Congratulations! You have been accepted to the MIT Class of 2020.

This year we had over 19,000 candidates for fewer than 1,600 offers of admission, from which will come our 1,100 freshmen. Congratulations again, and by the way, the flag is flag{i_wish_this_w@s_m3_RIP}

I wish you the very best in all of your future endeavors.

Sincerely,

Stuart Schmill
Dean of Admissions
```

The flag is **flag{i_wish_this_w@s_m3_RIP}**
