# Superhuman - HackMyVM (Easy)
 
![Superhuman.png](Superhuman.png)

## Übersicht

*   **VM:** Superhuman
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=Superhuman)
*   **Schwierigkeit:** Easy
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 7. Mai 2024
*   **Original-Writeup:** https://alientec1908.github.io/Superhuman_HackMyVM_Easy/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel der "Superhuman"-Challenge war die Erlangung von User- und Root-Rechten. Der Weg begann mit der Enumeration eines Webservers, der eine Datei `/notes-tips.txt` enthielt. Diese Datei enthielt einen Base85-kodierten String (dessen Inhalt im Writeup nicht weiterverfolgt wurde) und einen entscheidenden Hinweis auf ein Gedicht namens `salome_and_??` sowie die Notwendigkeit einer speicherplatzsparenden Erweiterung. Mittels `crunch` wurde eine Wortliste generiert, die zum Auffinden der passwortgeschützten ZIP-Datei `salome_and_me.zip` führte. Das Passwort (`turtle`) wurde mit `zip2john` und `john` geknackt. Die entpackte Datei `salome_and_me.txt` enthielt einen Hinweis auf den Benutzernamen `fred` und das Passwort `schopenhauer`. Dies ermöglichte den SSH-Login als `fred`. Die User-Flag wurde in dessen Home-Verzeichnis gefunden. Die Privilegieneskalation zu Root erfolgte durch Ausnutzung der Linux Capability `cap_setuid+ep`, die für das `/usr/bin/node`-Binary gesetzt war. Ein spezifischer Node.js-Befehl (gefunden auf GTFOBins) erlaubte es, die UID auf 0 (root) zu setzen und eine Root-Shell zu starten.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `gobuster`
*   `curl`
*   `base64` (Decoder)
*   `crunch`
*   `wget`
*   `zip2john`
*   `john` (John the Ripper)
*   `unzip`
*   `cat`
*   `tr`
*   `hydra`
*   `ssh`
*   `dir` (Alias für `ls`)
*   `getcap`
*   `node`
*   `cp`
*   `sh`
*   `ls`
*   `id`
*   Standard Linux-Befehle

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Superhuman" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Web Enumeration:**
    *   IP-Findung mit `arp-scan` (`192.168.2.114`).
    *   `nmap`-Scan identifizierte offene Ports: 22 (SSH - OpenSSH 7.9p1) und 80 (HTTP - Apache 2.4.38). Hostname `family` oder `super.hmv`.
    *   `gobuster` auf Port 80 (unter Hostname `super.hmv`) fand `/index.html`, `/nietzsche.jpg` und `/notes-tips.txt`.
    *   HTML-Kommentar in `/index.html` wies auf `webmaster.hmv` hin (alternativer/irreführender Hostname).

2.  **Information Disclosure & Hint Finding:**
    *   `curl http://super.hmv/notes-tips.txt` lud eine Datei herunter. Diese enthielt einen Base85-kodierten String und einen Hinweis auf ein Gedicht namens `salome_and_??` und eine speicherplatzsparende Erweiterung (Hinweis auf ZIP).
    *   `crunch 13 13 -t salome_and_@@ > dic.txt` erstellte eine Wortliste für Dateinamen basierend auf dem Muster.
    *   `gobuster dir -u "http://super.hmv" -w dic.txt -x zip ...` fand die Datei `http://super.hmv/salome_and_me.zip`.

3.  **Initial Access (fred):**
    *   `wget http://super.hmv/salome_and_me.zip` lud die ZIP-Datei herunter.
    *   `zip2john salome_and_me.zip > sal` extrahierte den Passwort-Hash der ZIP-Datei.
    *   `john --wordlist=/usr/share/wordlists/rockyou.txt sal` knackte den ZIP-Hash und fand das Passwort `turtle`.
    *   `unzip salome_and_me.zip` (mit Passwort `turtle`) entpackte die Datei `salome_and_me.txt`.
    *   Inhalt von `salome_and_me.txt`: Ein Gedicht, das den Benutzernamen `fred` und das Passwort `schopenhauer` enthielt/implizierte.
    *   `hydra -l fred -P [Wortliste_aus_Gedicht] ssh://super.hmv:22` bestätigte das Passwort `schopenhauer`.
    *   Erfolgreicher SSH-Login als `fred` mit Passwort `schopenhauer` (`ssh fred@super.hmv`).

4.  **Privilege Escalation (von `fred` zu `root`):**
    *   User-Flag `Ineedmorepower` in `/home/fred/user.txt` gelesen.
    *   `/usr/sbin/getcap / -r 2>/dev/null` fand heraus, dass `/usr/bin/node` die Capability `cap_setuid+ep` gesetzt hat.
    *   Ausnutzung dieser Capability mit dem GTFOBins-Payload für `node`:
        `/usr/bin/node -e 'process.setuid(0); require("child_process").spawn("/bin/sh", {stdio: [0, 1, 2]})'`
    *   Erlangung einer Root-Shell.
    *   Root-Flag `Imthesuperhuman` in `/root/root.txt` gelesen.

## Wichtige Schwachstellen und Konzepte

*   **Informationsleck in Textdatei:** Eine öffentlich zugängliche Textdatei (`notes-tips.txt`) enthielt entscheidende Hinweise (Dateinamenmuster, Kodierungshinweis).
*   **Passwortgeschütztes Archiv mit schwachem Passwort:** Eine ZIP-Datei war mit einem leicht zu knackenden Passwort (`turtle`) geschützt.
*   **Credentials in Klartext/Gedicht:** Benutzername und Passwort für den SSH-Zugang wurden durch Analyse eines Gedichts in einer Textdatei gefunden.
*   **Linux Capabilities (cap_setuid):** Das `node`-Binary hatte die `cap_setuid`-Capability, die es erlaubte, die effektive Benutzer-ID zu ändern und somit Root-Rechte zu erlangen.
*   **Verwendung von Standard-Passwortlisten:** `rockyou.txt` war erfolgreich beim Knacken des ZIP-Passworts.

## Flags

*   **User Flag (`/home/fred/user.txt`):** `Ineedmorepower`
*   **Root Flag (`/root/root.txt`):** `Imthesuperhuman`

## Tags

`HackMyVM`, `Superhuman`, `Easy`, `Information Disclosure`, `ZIP Cracking`, `SSH`, `Linux Capabilities`, `cap_setuid`, `node.js`, `Password Cracking`, `Linux`, `Web`
