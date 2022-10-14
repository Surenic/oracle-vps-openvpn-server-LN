# Hybrid mode for LND/CLN (Raspiblitz) über ein Oracle Cloud Free Tier VPN
Diese Anleitung soll als Hilfestellung dienen, eine bestehende Raspiblitz Full Node, dessen Lightning-Implementierung und gewünschte Dienste über die VPN-Clearnet-Adresse freizugeben. Die Anleitung lehnt sich, gerade was die Einrichtung des OpenVPN-Servers angeht, stark am [Tutorial](https://github.com/TrezorHannes/Dual-LND-Hybrid-VPS#vps-retrieve-the-openvpn-config--certificate) von [TrezorHannes](https://github.com/TrezorHannes) an.

### Voraussetzungen
- Ein Oracle Cloud Free Tier Account ([Link](https://www.oracle.com/cloud/free/)). Dieser erfordert Klarnamen-Einträge, die am Ende der Account-Erstellung mit der Eingabe von Kreditkarten-Daten und einer Ab- und Rückbuchung von 1$ bestätigt werden. Im Verlauf des Tutorials werden wir lediglich Produkte "buchen", die als "immer kostenlos" deklariert werden. Der Free Tier Account hat gewisse Einschränkungen in Sachen Anzahl aktiver Instanzen und Traffic, die hierfür aber mehr als ausreichend sein sollten.
- eine laufende Lightning Node (in diesem Tutorial gehe ich speziell auf Raspiblitz ein. Es lässt sich aber auch auf andere Implementierungen übertragen)
- Macht euch Gedanken darüber, welcher Services ihr über das Clearnet erreichen wollt und notiert euch die benötigten Ports (z.B. LND 9735, CLN 9736, LNbits 5001, Electrum Server 50002 uws.)


## VPS Server Setup

1. Loggt euch in euren Oracle Cloud Account
2. Klickt oben links auf die drei Balken, wählt "Compute" und dann "Instanzen"
3. Hier könnt ihr eine neue Instanz erstellen
4.  - Der Name ist frei wählbar
    - Compartment zeigt euren gewählten Oracle-Account-Namen
    - Platzierung zeigt den Serverstandort
    - Als Image wählt ihr "Canonical Ubuntu"
    - Die Shape bleibt unverändert
    - Unter "Networking" werden automatisch ein Cloud-Netzwerk und ein Subnetzwerk eingerichtet. Dies kann so bleiben. Beim Erstellen einer weiteren Instanz empfehle ich die Errichtung neuer Netzwerke
    - SSH Key: Um via SSH Zugriff auf den Ubuntu Server zu bekommen, braucht ihr einen Public Key, sowie den dazugehörigen Private Key. Wie das geht erfahrt ihr [hier](https://www.oracle.com/webfolder/technetwork/tutorials/obe/cloud/compute-iaas/generating_ssh_key/generate_ssh_key.html#section1s2) Hinweis, benutzt unter Windows die "Windows Power Shell" und nicht puTTY, es sei denn ihr wisst, wie ihr den späteren scp-Befehl mit dem private key nutzt. Ich weiß es nicht :D
5. Ist der Public Key erfolgreich hochgeladen oder eingefügt, klickt ihr auf "Erstellen"


## VPS Server Konfiguration

1. Ist die Instanz hochgefahren, seht ihr unter Instanzzugriff euren Benutzernamen (ubuntu) sowie die Public IP. Notiert diese.
2. Unter Instanzdetails klickt ihr den Link zu eurem Virtuellen Cloud-Netzwerk, im folgenden Fenster auf euer Subnetz und dann wiederum auf eure "Default Security List". Hier müssen nun einige Portfreigaben eingerichtet werden
3. Unter "Impress-Regeln hinzufügen" fügt ihr folgendes ein:
   - Quell-CIDR: 0.0.0.0/0
   - IP-Protokoll: UDP
   - Zielportbereich: 1194
   und klickt auf Impress-Regeln hinzufügen
4. Wiederholt Schritt 3, wählt statt UDP TCP und tragt bei Zielportbereich mit Kommata getrennt alle Ports ein, die ihr später entsprechend der Dienste freigeben wollt. Der Wichtigste ist hier die 9735 und/oder die 9736


## VPS Server-Zugriff und Einrichtung

Nehmt euren SSH-Client zur Hand und verbindet euch mit eurem Server mittels des zuvor erstellten private keys. Im Falle des Linux-Terminals sieht das so aus

`ssh ubuntu@PUBLIC_IP -i PATH_TO_PRIVATE_KEY/PRIVATE_KEY_FILE`

`PUBLIC_IP` ist dabei die zuvor notierte IP eures Servers, der Pfad unter Linux ist standardmäßig `~/.ssh`

Nun führt ihr einige Befehle aus, um die Instanz auf den neuesten Stand zu bringen, Docker sowie die Uncomplicated Firewall (UFW) zu installieren.

```
sudo apt update
sudo apt upgrade -y
sudo apt shutdown -r now
```

Der letzte Befehl startet die Instanz neu. Das kann einige Zeit dauern. Holt euch einen Kaffee und loggt euch nach einer Weile via ssh wieder ein. Dann geht es weiter:

```
sudo apt install docker.io
sudo systemctl start docker.service
sudo apt install ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw allow PORT comment 'CUSTOM'
```

Hinweis: Für `PORT` setzt ihr den gewünschten Port sowie für `CUSTOM` einen entsprechenden Namen ein (z.B LND NODE, LNbits) und wiederholt den Command mit allen gewünschten Ports. Im Zweifel kann dies auch nachträglich geschehen.

`sudo ufw enable`

Es kommt eine Warnung, dass möglicherweise die SSH-Verbindung gekappt wird. Da wir OpenSSH freigegeben haben, sollte dies nicht passieren.

Zu guter Letzt:

`sudo apt install fail2ban` um den SSH-user zu schützen

## OpenVPN installieren

Der Einfachheit halber bedienen wir uns eines OpenVPN Docker-Images ([Link](https://hub.docker.com/r/kylemanna/openvpn/)).

Zum Installieren und Einrichten führt ihr folgende Schritte durch

`sudo export OVPN_DATA="ovpn-data"` Dieser Befehl setzt einen globalen Platzhalter für euer VPN. Um dies nach dem Reboot beizubehalten, fügt ihr den Befehl zusätzlich unter `sudo nano .bashrc` ein, beendet Nano mit STRG+X und bestätigt das Speichern mit Y (YES).

`sudo docker volume create --name $OVPN_DATA`
`sudo docker run -v $OVPN_DATA:/etc/openvpn --rm kylemanna/openvpn ovpn_genconfig -u udp://PUBLIC_IP` hier wieder eure PUBLIC_IP einfügen
`sudo docker run -v $OVPN_DATA:/etc/openvpn --rm -it kylemanna/openvpn ovpn_initpki` dies lässt euch ein notwendiges Zertifikats-Passwort erstellen, welches im folgenden Verlauf nochmal abgefragt wird. Schreibt es euch gut auf.
`sudo docker run -v $OVPN_DATA:/etc/openvpn -d -p 1194:1194/udp -p 9735:9735 -p 9736:9736 --cap-add=NET_ADMIN kylemanna/openvpn` hier kettet ihr alle gewünschten Ports mittels -p PORT:PORT aneinander. In diesem Kommando beispielhaft 9735 und 9736

Der OpenVPN Server läuft nun auf Port UDP 1194. Es müssen nun Benutzer Client-Zugangsdaten erstellt werden

## Erstellung und laden der OpenVPN Konfiguration

`sudo docker run -v $OVPN_DATA:/etc/openvpn --rm -it kylemanna/openvpn easyrsa build-client-full NODENAME nopass` wobe `NODENAME` ein gewünschter Name sein kann. Ich habe hier den Namen der Node gewählt.

`sudo docker run -v $OVPN_DATA:/etc/openvpn --rm kylemanna/openvpn ovpn_getclient NODENAME > NODENAME.ovpn` gibt die benötigte Konfigurationsdatei `NODENAME.ovpn` aus. 

Diese Datei muss nun auf eure Node geladen werden. (Viele Wege führen nach Rom, man kann, sofern man den SSH private Key auf der Node hat, auch jetzt bereits auf die Node switchen und diese direkt darauf laden). Wir nutzen dafür einen Zwischenschritt.

Öffnet ein neues Terminalfenster

`scp 'ubuntu@PUBLIC_IP:/etc/openvpn/NODENAME.ovpn' ./ -i PATH_TO_PRIVATE_KEY/PRIVATE_KEY`

Dies kopiert die Datei zunächst auf euren Rechner

`scp NODENAME.ovpn admin@NODEIP:/admin` kopiert die Datei auf eure Node.

## Einloggen und installieren von OpenVPN Client auf der Node

`ssh admin@NODEIP` bringt euch ins bekannte Blitzmenü. Wählt Exit um ins Terminalfenster zu gelangen

`sudo chmod 600 NODENAME.ovpn` vergibt der File, die hier im admin-Ordner liegen sollte, noch entsprechende Rechte.

```
sudo apt-get install openvpn
$ sudo mv /home/admin/NODENAME.ovpn /etc/openvpn/CERT.conf
$ sudo systemctl enable openvpn@CERT
$ sudo systemctl start openvpn@CERT
```

Diese Befehle verbinden eure Node nun mit eurem VPS VPN Server, es sollte eine entsprechende Ausgabe kommen, die so aussieht:

```
* openvpn@CERT.service - OpenVPN connection to CERT
     Loaded: loaded (/lib/systemd/system/openvpn@.service; enabled; vendor preset: enabled)
     Active: active (running) since Sat 2009-01-03 19:15:00 CEST; 4s ago
       Docs: man:openvpn(8)
             https://community.openvpn.net/openvpn/wiki/Openvpn24ManPage
             https://community.openvpn.net/openvpn/wiki/HOWTO
   Main PID: 1514818 (openvpn)
     Status: "Initialization Sequence Completed"
      Tasks: 1 (limit: 18702)
     Memory: 1.0M
        CPU: 49ms
     CGroup: /system.slice/system-openvpn.slice/openvpn@CERT.service
             `-1514818 /usr/sbin/openvpn --daemon ovpn-CERT --status /run/openvpn/CERT.status 10 --cd /etc/openvpn --config /etc/openvpn/CERT.conf --writepid /run/openvpn/CERT.pid

Jan 03 19:15:00 debian-nuc ovpn-CERT[1514818]: WARNING: 'link-mtu' is used inconsistently, local='link-mtu 1541', remote='link-mtu 1542'
Jan 03 19:15:00 debian-nuc ovpn-CERT[1514818]: WARNING: 'comp-lzo' is present in remote config but missing in local config, remote='comp-lzo'
Jan 03 19:15:00 debian-nuc ovpn-CERT[1514818]: [PUBLIC_IP] Peer Connection Initiated with [AF_INET]PUBLIC_IP:1194
Jan 03 19:15:01 debian-nuc ovpn-CERT[1514818]: TUN/TAP device tun0 opened
Jan 03 19:15:01 debian-nuc ovpn-CERT[1514818]: net_iface_mtu_set: mtu 1500 for tun0
Jan 03 19:15:01 debian-nuc ovpn-CERT[1514818]: net_iface_up: set tun0 up
Jan 03 19:15:01 debian-nuc ovpn-CERT[1514818]: net_addr_ptp_v4_add: 192.168.255.6 peer 192.168.255.5 dev tun0
```

Notiert euch die IP in der letzten Zeile, die zuerst angezeigt wird, im Beispiel hier die `192.168.255.6`. Wir nennen sie ab hier `VPN_IP`.

## Einrichten der Portweiterleitung

Damit der Aufruf von PUBLIC_IP:PORT auch an die Node weitergeleitet wird, müssen auf dem Server noch einige Konfigurationen vorgenommen werden. Hierfür brauchen wir die `VPN_IP`

Hierfür müssen wir direkt in die Docker Container Shell

`sudo docker ps` zeigt euch eure Container, beachtet die `CONTAINER-ID`
`docker exec -it <CONTAINER-ID> sh` gebt hier die entsprechende Container-ID ein

In der Shell angekommen führt ihr folgende Befehle aus

```
sudo iptables -A PREROUTING -t nat -i eth0 -p tcp -m tcp --dport 9735 -j DNAT --to VPN_IP:9735
sudo iptables -A PREROUTING -t nat -i eth0 -p tcp -m tcp --dport 9736 -j DNAT --to VPN_IP:9736
sudo iptables -t nat -A POSTROUTING -d 192.168.255.0/24 -o tun0 -j MASQUERADE
```

Beachtet hierbei, dass ihr entsprechend der gewünschten Ports die Befehle entsprechend wiederholt. Die POSTROUTING IP ist eure VPN_IP, jedoch mit einer 0 an vierter Stelle

Um diese Port Forwards beim Reboot des Servers beizubehalten, muss das Ganze in die /etc/openvpn/ovpn_env.sh eingetragen werden. Wir sind nach wie vor in der Docker Shell

```
cd /etc/openvpn
sudo vi ovpn_env.sh
```

Leider kann Docker kein `nano`, sondern nur `vi`, der etwas komplizierter erscheint. Geht mit den Pfeiltasten bis nach unten und drückt "o" für "open line below". Fügt nun wie gewohnt die Zeilen von eben noch einmal ein (ohne sudo), drück `Esc`, dann `:wq`.

## Bearbeiten der LND/CLN CONF Datei

Im Raspiblitzmenü unter System findet ihr die entsprechenden Konfigurationsdateien, die ihr wie folgt bearbeiten müsst.

LND

```
externalip=PUBLIC_IP:9735
nat=false

tor.active=true
tor.v3=true
tor.streamisolation=false
tor.skip-proxy-for-clearnet-targets=true
```

CLN

```
bind-addr=0.0.0.0:9736
addr=statictor:127.0.0.1:9051/torport=9736
always-use-proxy=false
announce-addr=PUBLIC_IP:9736
```

In beiden Fällen solltet ihr checken, ob gewisse Einträge nicht bereits vorhanden sind und geändert werden müssen.

Nun müsst ihr beim Speichern der Datei den entsprechenden Service neu starten.

Das war's. Eure Nodes und Dienste sollten nun unter der entsprechenden IP erreichbar sein


Wenn euch das Tutorial gefallen hat und alles funktioniert, wie es soll, freue ich mich, wenn ihr meinen LNurlp-Link mal ausprobiert ;)

[<img src=https://raw.githubusercontent.com/Surenic/Hybrid-mode-for-LND-CLN-Raspiblitz-connected-to-an-Oracle-Cloud-Free-Tier-VPN/main/qr.png width="200" height="200">](https://coresln.duckdns.org:5001/lnurlp/1)

Folgt mir auf Twitter!

[<img src=https://upload.wikimedia.org/wikipedia/commons/4/4f/Twitter-logo.svg width="50" height="50">](https://twitter.com/surenic)

Ansonsten freue ich mich auf Verbesserungen, Anregungen und ähnliches
