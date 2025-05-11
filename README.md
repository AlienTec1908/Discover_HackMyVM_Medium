# Discover - HackMyVM (Medium)

![Discover.png](Discover.png)

## Übersicht

*   **VM:** Discover
*   **Plattform:** [HackMyVM](https://hackmyvm.eu/machines/machine.php?vm=Discover)
*   **Schwierigkeit:** Medium
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 30. September 2022
*   **Original-Writeup:** https://alientec1908.github.io/Discover_HackMyVM_Medium/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser "Medium"-Challenge war es, Root-Zugriff auf der Maschine "Discover" zu erlangen. Die Enumeration deckte einen Apache-Webserver (Port 80) und mehrere GlassFish-bezogene Dienste (Ports 4848, 8080 etc.) auf. Ein per `gobuster vhost` gefundener vHost `log.discover.hmv` offenbarte eine Command Injection Schwachstelle im `username`-Parameter, die über eine PHP-Webshell zu einer Reverse Shell als `www-data` führte. Als `www-data` wurde eine `sudo`-Regel entdeckt, die das Ausführen von `/opt/overflow` als Benutzer `discover` erlaubte. Diese Datei war anfällig für einen Buffer Overflow, der durch einen Hinweis (`hint`-Datei) und einen Python-Payload ausgenutzt wurde, um eine Shell als `discover` zu erlangen. Schließlich erlaubte eine weitere `sudo`-Regel `discover` das Ausführen von `/opt/glassfish6/bin/asadmin` als `root`. Durch Erstellen einer neuen GlassFish-Domain, Aktivieren von Secure Admin und Deployen einer bösartigen WAR-Datei (erstellt mit `msfvenom`) wurde eine Root-Shell erlangt.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `gobuster`
*   `nikto`
*   `wfuzz`
*   `msfvenom`
*   `netcat` (nc)
*   `python` (für http.server und Skripte)
*   `curl`
*   `wget`
*   `sudo` (auf Zielsystem)
*   `asadmin` (GlassFish Admin CLI, als Exploit-Vektor)
*   Standard Linux-Befehle (`vi`, `cat`, `ls`, `find`, `echo`, `chmod`, `file`, `stty`, `cd`, `id`, `whoami`, `pwd`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Discover" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Web Enumeration:**
    *   IP-Findung mittels `arp-scan` (Ziel: `192.168.2.112`).
    *   `nmap`-Scan identifizierte Apache (80/tcp) und diverse GlassFish-Dienste (u.a. Admin auf 4848/tcp, HTTP-Proxy auf 8080/tcp).
    *   `gobuster dir` auf Port 8080 fand nur `index.html`. `nikto` auf Port 4848 zeigte viele potenzielle, aber generische Schwachstellen.
    *   `gobuster vhost` fand den vHost `log.discover.hmv`.
    *   `wfuzz` auf `log.discover.hmv/index.php` (mit `linux_commands.txt` als Wortliste für den `username`-Parameter) deckte eine Command Injection Schwachstelle auf.

2.  **Initial Access (LFI/RCE & Reverse Shell als `www-data`):**
    *   Die Command Injection auf `log.discover.hmv` wurde bestätigt (`username=id` -> `uid=33(www-data)`). Im Quellcode der `index.php` (ausgelesen via RCE) wurde das Passwort `ufoundmypassword` für den Benutzer "Discover" gefunden.
    *   Eine PHP-Webshell (`<?php system($_GET['cmd']); ?>`) wurde in `/dev/shm/ben.php` auf dem Zielsystem via RCE erstellt.
    *   Mittels `curl` oder `wget` wurde ein PHP-Reverse-Shell-Skript (`rev.php`) von einem lokalen Python-HTTP-Server des Angreifers nach `/dev/shm/rev.php` auf das Ziel geladen.
    *   Das Skript `rev.php` wurde via RCE (`php /dev/shm/rev.php`) ausgeführt, was eine Reverse Shell als `www-data` zum Netcat-Listener des Angreifers etablierte.

3.  **Privilege Escalation (von `www-data` zu `discover` via Buffer Overflow):**
    *   Als `www-data` zeigte `sudo -l`, dass `/opt/overflow` als Benutzer `discover` ohne Passwort ausgeführt werden durfte.
    *   Die Datei `/opt/overflow` war nur ausführbar, nicht lesbar. Der Name deutete auf einen Buffer Overflow hin.
    *   Eine Datei `/opt/hint` enthielt einen Exploit-String-Anfang (`AAAAAAAAAAAAAAAAAAAAAAAA"+"\x5d\x06\x40\x00`).
    *   Der Exploit-String wurde über ein Python-Skript (`sys.stdout.write(...)`) generiert und als Argument an `sudo -u discover /opt/overflow $(python3 /dev/shm/neu.py)` übergeben.
    *   Dies führte zu einem erfolgreichen Buffer Overflow und startete eine Shell als Benutzer `discover`. Die User-Flag wurde gelesen.

4.  **Privilege Escalation (von `discover` zu `root` via GlassFish `asadmin`):**
    *   Als `discover` zeigte `sudo -l`, dass `/opt/glassfish6/bin/asadmin` als `root` ohne Passwort ausgeführt werden durfte.
    *   Mittels `sudo /opt/glassfish6/bin/asadmin create-domain domain3` wurde eine neue GlassFish-Domain mit dem Admin-Benutzer `benni` und Passwort `hacker123` (Passwort wurde im Prozess geändert) erstellt.
    *   Die Domain wurde gestartet (`start-domain domain3`) und Secure Admin für Fernzugriff aktiviert (`enable-secure-admin`).
    *   Eine bösartige WAR-Datei (`shell.war`) mit einem `java/jsp_shell_reverse_tcp`-Payload (erstellt mit `msfvenom`) wurde auf das Zielsystem (`/dev/shm/shell.war`) heruntergeladen.
    *   Die `shell.war` wurde über das GlassFish-Admin-Webinterface (Port 4848) auf der `domain3` deployt.
    *   Beim Aufrufen der deployten Anwendung wurde die Reverse Shell ausgelöst und eine Verbindung als `root` zum Netcat-Listener des Angreifers hergestellt. Die Root-Flag wurde gelesen.

## Wichtige Schwachstellen und Konzepte

*   **Command Injection:** Im `username`-Parameter der `index.php` auf dem vHost `log.discover.hmv`.
*   **Information Disclosure:** Passwort (`ufoundmypassword`) im Quellcode. Buffer-Overflow-Hinweis in `/opt/hint`.
*   **Buffer Overflow:** In der Datei `/opt/overflow`, ausnutzbar über eine `sudo`-Regel.
*   **Unsichere `sudo`-Regeln:**
    *   `www-data` durfte `/opt/overflow` als `discover` ausführen.
    *   `discover` durfte `asadmin` als `root` ausführen.
*   **GlassFish `asadmin` Exploit:** Erstellen einer neuen Domain und Deployen einer bösartigen WAR-Datei zur RCE als Root.
*   **PHP Webshell / Reverse Shell:** Mehrfach verwendet für initialen Zugriff und Codeausführung.
*   **Beschreibbares `/dev/shm`:** Genutzt zum Ablegen von Payloads.

## Flags

*   **User Flag (`/home/discover/User.txt`):** `c7d0a8de1e03b25a6f7ed2d91b94dad6`
*   **Root Flag (`/root/Root.txt`):** `7140a59e697f44b8a8581cc85df76f4c`

## Tags

`HackMyVM`, `Discover`, `Medium`, `Web`, `Apache`, `GlassFish`, `Command Injection`, `RCE`, `Buffer Overflow`, `sudo`, `asadmin`, `WAR Deployment`, `PHP Webshell`, `Reverse Shell`, `Privilege Escalation`, `Linux`
