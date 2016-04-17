### Solved by z0rex

Amoebananas! was a web challenge worth 20 points.

> The amoeba is a fascinating creature. 


![Screenshot](/images/2016/angstrom-ctf/amoebananas/amoebananas.png)

Looking at the page's source code with `ctrl-u` revealed the flag

```html
<html>
    <head>
        <title>Amoeba Central</title>
    </head>

    <body>
        <h1 style="color:blue;">Welcome to your source for amoeba information!</h1>
        <h2 style="color:green;">A WORLD LEADER IN AMOEBA SCIENCE</h2>

        <img src="https://upload.wikimedia.org/wikipedia/commons/9/90/Ameoba_Under_Light_Microscope.jpg">
        <p>"Amoeba Under Light Microscope" by Iceclanl</p>


        <!-- Your flag is: pseudopods_are_da_bomb -->
    </body>
</html>
```

Flag: `pseudopods_are_da_bomb`