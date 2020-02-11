## DNS

Dit deel gaat over het opstellen van een correcte domain name server, maar ook van het oplossen van verschillende problemen. Alle voorbeelden komen van een netwerk dat ik heb opgesteld in mijn bachelor.

#### Internet laag

Het eerste wat je moet doen is in /etc/resolv.conf gaan kijken, hier mogelijks de juiste nameserver toevoegen. Als je op een nameserver aan het werken bent, voeg je best je eigen IP-adres toe, of 0.0.0.0, of 127.0.0.1 (loopback-adres aka zichzelf).

`sudo vim /etc/resolv.conf`

Een voorbeeld van resolv.conf:

``````bash
domain cynalco.com #Optioneel, maar we speciferen hier welk domein we zullen gebruiken.
search cynalco.com #Verplicht?, als je de nameserver zelf bent doe je best 'search home'
nameserver 192.168.56.42 #IP van master huidig domein, DNS-reqs naar hier verzonden
nameserver 10.0.2.3 #Optioneel, IP van master ander domein, indien je in een ander domein ook zal moeten dns-reqs versturen
``````

Dit gedeelte hoort nog bij de internet laag, aangezien je nog geen gebruikt maakt van de named applicatie.

#### Applicatie laag

We zullen ons nu bezig houden om de named applicate correct te configureren en op te starten. Named is de naam van de BIND DNS configuratie, waarbij BIND voor Berkeley Internet Name Domain staat. Ik gebruik hierbij versie 8. Handig om te weten is dat BIND op een userlevel draait, voor hackers tegen te gaan, we maken dus gebruik van systemctl (een service die deamons beheert) bij linux.

Begin met een `sudo systemctl status named` gevolgd met`sudo systemctl start named`. Hier zouden we al heel wat informatie kunnen uithalen, indien je aan het debuggen bent. Als de DNS nog niet aangemaakt is dan begin je best eerst met de configuraties in te stellen.

Onthoud dat het ook handig is om met `vim /var/log/messages` of met `tail -F /var/log/messages` te kijken wanneer er fouten oplopen.



####named.conf

We maken gebruik van twee bestanden om de named-deamon te configureren: `/etc/named.conf` en `/var/named/chroot/etc/named.conf`. We kijken in `/etc/named.conf`, een voorbeeld hiervan is:

```````bash
options {
        listen-on port 53 { 127.0.0.1; any; }; # Poort 53 staat voor DNS in firewall
        # Je laat dus het verkeer van deze IPv4 adressen toe in verband met DNS
        # Any laat al het verkeer toe
        listen-on-v6 port 53 { any; }; # Same maar voor IPv6 adressen
        directory       "/var/named"; # Alle files wat hij nodig heeft moeten hierin staan
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        allow-transfer  { any; }; # Staat toe om alle zone informatie door te geven van de
        # master server naar andere servers. Default is altijd allow-any
        allow-query     { any; };

        recursion no;

        rrset-order { order random; };

        dnssec-enable yes;
        dnssec-validation yes;
        dnssec-lookaside auto;

        /* Path to ISC DLV key */
        bindkeys-file "/etc/named.iscdlv.key";

        managed-keys-directory "/var/named/dynamic";

        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";
};

# Optioneel kan je hier nog controls {}; toevoegen om ervoor te zorgen dat de cache van gekende servers op deze server gecleard wordt na enige tijd.

logging { # In dit geval worden alle messages opgeslaan in data/named.run
		  # Meestal gebruiken we /var/log/messages echter.
        channel default_debug {
                file "data/named.run";
                severity dynamic;
                print-time yes;
        };
};

zone "." IN {
        type hint; # Deze server is niet de master server van de volledige zone, dus
        		   # zetten we hier hint
        file "named.ca"; # De zone '.' is verantwoordelijkheid van een andere server, dus
        # maken we een verwijzing naar de root server in named.ca
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";

# Deze server is de master nameserver van 3 verschillende zones, waarbij iedere configuratie van de zone in de 3 files wordt bewaard:
zone "cynalco.com" IN {
  type master; 
  file "cynalco.com";
  notify yes;
  allow-update { none; };
};

zone "56.168.192.in-addr.arpa" IN {
  type master;
  file "56.168.192.in-addr.arpa";
  notify yes;
  allow-update { none; };
};
zone "2.0.192.in-addr.arpa" IN {
  type master;
  file "2.0.192.in-addr.arpa";
  notify yes;
  allow-update { none; };
};
```````

Kort uitgelegd, wat je dus in deze file terugvind:

- Het DNS verkeer op deze nameserver {options}

- Waar foutmeldingen en andere messages worden opgeslaan {logging}

- Optionele lijst om informatie in te stellen voor administratieve services {controls}

- Overkoepelende zone met verwijzing naar de root server hiervan {zone '.'}

- Alle zone's waar deze nameserver de verantwoordelijke/master voor is {zone "cynalco.com"}

  

####named.ca

Een voorbeeld van `named.ca` om dus een verwijzing te doen naar 1 of meerdere rootservers:

``````bash
.                        3600000    IN		NS   	Guillaume
Guillaume			     3600000    IN		A    	192.36.148.17
www                 	 3600000    IN  	CNAME   Guillaume
``````

Voor iedere root server moet je een NS en A record hebben hierin. Guillaume is dus een root server.

Het puntje staat voor het domein van de huidige zone? 3600000 staat voor de TTL, of wanneer de verbinding mag geschrapt wanneer de laatste connectie met deze server is verstreken. IN staat voor Internet. De 4e kolom staat voor record-type. De Nameserver Guillaume heeft dus als IP-addres 192.36.148.17.

De laatste lijn is voor CNAME of Canonical Name, dit is een gevorderde vorm van DNS records, waar je domeinnaam automatisch wordt aangepast op het einde. Ons domein wordt nu bijvoorbeeld **www**.hogent.be



####cynalco.com

Nu, om een voorbeeld te tonen van een zone configuratiebestand, dit is `/var/named/cynalco.com`, merk op dit is in dezelfde directiory als dat je opgegeven hebt in de options:

```bash
$ORIGIN cynalco.com. # Dit is een specificatie van het huidig domein
$TTL 1W # Zet een default TTL in, dit wordt toegepast voor connecties met alle servers

@ IN SOA golbat.cynalco.com. hostmaster.cynalco.com. (
  16121208
  1D
  1H
  1W
  1D ) # Dit zijn een serie tijdspecificaties, maar geen idee voor wat

# Deze zone bevat twee name servers, namelijk golbat en tamata
# Merk op, golbat is hetzelfde als golbat.cynalco.com
# Pas echter op met golbat., dan wordt het domein niet toegevoegd
# een lege eerste kolom wijst op de huidige zone
                     IN  NS     golbat #master
                     IN  NS     tamatama #slave

# Het type MX is om ervoor te zorgen dat alle mail van onze simple mail transfer protocol server, namelijk sawamular, het domeinnaam krijgen toegevoegd achteraan, voordat de mail wordt verdergestuurd.
@                    IN  MX     10  sawamular.cynalco.com.

# De rest van de entries zijn ofwel andere servers (GEEN NAMESERVER), of clients.
# Er wordt terug gebruik gemaakt van CNAME om twee databases in te stellen, de inside NAT, twee nameservers(namelijk idd golbat en tamata), de internetserver en een smtp server die tergelijkertijd ook een imap server is.
fushigisou           IN  A      192.168.56.2
butterfree           IN  A      192.168.56.12
db1                  IN  CNAME  butterfree
beedle               IN  A      192.168.56.13
db2                  IN  CNAME  beedle
pikachu              IN  A      192.168.56.25
inside               IN  CNAME  pikachu
golbat               IN  A      192.168.56.42
ns1                  IN  CNAME  golbat
mankey               IN  A      192.168.56.56
files                IN  CNAME  mankey
tamatama             IN  A      192.0.2.2
ns2                  IN  CNAME  tamatama
karakara             IN  A      192.0.2.4
www                  IN  CNAME  karakara
sawamular            IN  A      192.0.2.6
smtp                 IN  CNAME  sawamular
imap                 IN  CNAME  sawamular
```

In het zone configuratie bestand vind je dus alle informatie terug over alle servers (zowel nameserver als andere server). Iedere server heeft een naam en een eigen IP-adres. Uiteindelijk zal dus iedere server hierop kunnen surfen naar google.com, waar de master DNS server deze request dan zal vertalen naar het IP-Adres en de juiste poort van een google nameserver. Via de internet server (www) kunnen we dan de juiste informatie van die google server terug achterhalen en op de server die het vroeg gaan weergeven.
