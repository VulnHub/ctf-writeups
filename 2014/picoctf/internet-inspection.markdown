### Solved by superkojiman

Internet Inspection is a 30 point web exploitation challenge. 

> On his computer, your father left open a browser with the Thyrin Lab Website. Can you find the hidden access code?

The webiste mentioned looks like this:

![](/images/2014/pico/internet_inspection/01.png)

The information we need is blurred out, so let's fix that. Using Chrome's Developer Tools, we see that in body > style, there's a table element that blurs out the text:

![](/images/2014/pico/internet_inspection/02.png)

Let's remove the blur and we're left with the following:

![](/images/2014/pico/internet_inspection/03.png)

Much better, but still unreadable. Notice that right after the style section we just modified, there's a div with id value "checkers". It looks like it's responsible for adding the checkere'd effect. 

![](/images/2014/pico/internet_inspection/04.png)

Delete that and we're left with the uncensored message along with our flag:

![](/images/2014/pico/internet_inspection/05.png)

The flag is **flag_dafe3b8b4cb69e88e4271e616ad38a7b5080dd13** 
