### Solved by historypeats

> They are following me. They are after my token. I have to hide it somewhere. I’m not very good at hiding.
>
> tumorous.9447.plumbing

##Write Up

When we navigate to the URL provided (http://tumorous.9447.plumbing), we can see some timestamped logs claiming that the user is new to git.

```
3/2/2004: I learned how to html, yay! 
12/4/2004: I learned how to use git, yay! 
13/4/2004: Hidden my 'repository' so people can't access it. I have a feeling I will need to protect something soon. 08/9/2004: Forged a token from the whispering iron. It is very dear to me, I should protect it. 
10/9/2004: I put my token in a text file to protect it from alien mind readers from planet Zblaargh. 
10/9/2004: I can't forget my token. What do I do? 
11/9/2004: I panicked and deleted the token. It is the work of evil doers. 
12/9/2004: My token is lost. My life has no meaning now. I'm going to watch Louie season 4.
```
When I see someone new to something, I immediately think a configuration error or some sort of bad practice. In this case, it happened to be a bad practice with exposing their .git folder.

I was able to discover the http://tumorous.9447.plumbing/.git/ folder by running dirb against the host. Manually browsing this directory returned a 403 forbidden error. So it appears that directory indexing was disabled. However, running dirb a second time, using the .git directory as the "root", I was able to discover that the http://tumorous.9447.plumbing/.git/index file was available. Knowing that the .git/index file was accessible, I was able to use the rip-git.pl (https://github.com/kost/dvcs-ripper) script to read the index file and download the contents. All that was left was to read the token file.

```
$ cat token
9447{IM_SITTING_ON_A_PILE_OF_GOLD}
```
