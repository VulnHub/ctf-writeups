### Solved by historypeats

This challenge was a web application that would change images when you typed them in. The method for communication was websockets between the application endpoint and the front-end. To do this challenge, I used burp to inspect the results of the websockets.

In the input box, when I entered in an invalid file, the response in the web socket yielded the following message:
```
xxd: notanimage.png: No such file or directory
```
It looks like the file we submit to the input box is being used with the external tool, xxd. This hinted me to try some command injection.

I then tried the following:
```
.; ls 
```
This yielded the following response:
```
Dockerfile
colors.jpg
index.html
readmeforkey.txt
server.js
stocks.jpg
sunflower.jpg
unavailable.jpg
whale.jpg
```

We can see that command injection worked, and that the key file is there. All that's left to do is read the file.

```
.; cat readmeforkey.txt
```
And the results...
```
The Key is:

bashing_jpgs

```
Key: bashing_jpgs
