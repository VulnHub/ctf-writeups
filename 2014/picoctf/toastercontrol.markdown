### Solved by et0x

Toaster Control is a 50 point web exploitation challenge. 

> Daedalus Corp. uses a web interface to control some of their toaster bots. It looks like they removed the command 'Shutdown & Turn Off' from the control panel. Maybe the functionality is still there... 

Visiting the provided link gives you an interface with three buttons, "Blink Lights", "Patrol Mode", and "Make Toast".  Clicking on each parses a handler php file, handler.php, with a variable, "action" appended, and each button performs an associated action.

So **Blink Lights** = "/handler.php?action=Blink Lights"

The problem says to check the functionality of "Shutdown & Turn Off". (note the ampersand)

Now you can't simply use "handler.php?action=Shutdown & Turn off", because the php file will see everything after the "&" as more arguments for the request.  You have to url encode it, so...

/handler.php?action=Shutdown %26 Turn Off

Sure enough it works, yielding the following:

Shutdown code: **flag_c49bdkeekr5zqgvc20vc**
