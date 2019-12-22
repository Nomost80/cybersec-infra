Il faut tout noter dans l'archi.

Step 1 : Reconnaissance
Step 2 : Scanning
Step 3 : Gain d'accès
Step 4 : Maintient d'accès
Step 5 : Effacer les traces
Step 6 : Reporting
    Description de vulné
    Sévérité
    Appareils affectés
    Type de vulné
    Remèdes et solutions

Zones des réseaux:
* Internet
* DMZ
* Production
* Intranet
* Management

Type de vulné :
active
passive
host based
internal
external
application
network
wireless network

Méthode :
Faire un scope, identifier le périmètre avec les infos données

Audit : analyser l'archicture, configuration avancée

recon-ng : creator-reporting module et créer un workspace pour chaque entreprise
https://github.com/leebaird/discover

désactiver cdp lldp

scanning
nikto
nmap, mascan, netstat
scapy
étaler le scannage sur le temps pour éviter l'IDS
zone transfert dns
wordlist Kali 
shodan
robtex
BURP proxy
wfuzz
injection de commande ping
sqlmap
beef
yersinia vlan hopping
pass the hash

reporting : faraday, maltago

cat /etc/services
nmap -sS -sV --version-all ...
ping 127.0.0.1; cat /etc/passwd

http://www.dvwa.co.uk/
https://github.com/Hack-with-Github/Awesome-Hacking
