# IT Challenge

## 1. Server update
#### Na strežnik namesti vse zadnje update in fixe

### Postopek:
```shell
sudo apt-get update
sudo apt-get upgrade
```
na podlagi izpisa še opcijsko:
```shell
sudo apt-get dist-upgrade
```
lahko še preveriš celotno zgodovino update-ov:
```shell
less /var/log/apt/history.log 
```

## 2. Firewall - iptables
#### Na strezniku z uporabo iptables nastavi firewall, ki bo imel:

- odprt dostop za tvoj IP naslov do ssh in dns
- odprt dostop za IPnajboljseEkipe (vsi protokoli, vsi porti)
- odprt dostop za tcp port 43210 na katerem poslusa tudi ssh
- zaprt dostop za vse ostalo

### Postopek:
Ker se iptables pravila privzeto ne obdržijo po restartu sistema, si pomagamo z naslednjim orodjem, ki ga namestimo:

```shell
sudo apt-get install iptables-persistent
```
poženemo definicije pravil:  
**zaporedje je važno**, da sami sebe ne zaklenemo :)
```shell
sudo iptables -A INPUT -p tcp -s mojIPnaslov --dport 22 -j ACCEPT
sudo iptables -A INPUT -p tcp -s mojIPnaslov --dport 53 -j ACCEPT
sudo iptables -A INPUT -p udp -s mojIPnaslov --dport 53 -j ACCEPT
sudo iptables -A INPUT -s IPnajboljseEkipe -p all -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 43210 -j ACCEPT
sudo iptables -A INPUT -j DROP
```

za trajno shranjevanje pravil poženemo:
```shell
sudo netfilter-persistent save
```
## 3. DNS setup
#### Na strezniku namesti DNS in ga skonfiguriraj kot avtoritativni streznik za test.3fs.si domeno. <br/>Zona za test.3fs.si naj vsebuje alias za www.test.3fs.si ki naj kaze na ta streznik.

### Postopek:
Najprej namestimo dns bind9:
```shell
sudo apt-get install bind9
```

izklopimo rekurzijo tako, da na koncu v datoteko "/etc/bind/named.conf.options" dodamo novo vrstico "recursion no;":
```shell
		...
        auth-nxdomain no;    # conform to RFC1035
        listen-on-v6 { any; };
        recursion no;
};
```
ustvarimo novi datoteki:  
[/etc/bind/db.test.3fs.si](./dns/db.test.3fs.si)   
[/etc/bind/named.conf.custom-zones](./dns/named.conf.custom-zones)

in dodamo še nov include "/etc/bind/named.conf.custom-zones", v datoteko "/etc/bind/named.conf":
```shell
...
include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
include "/etc/bind/named.conf.default-zones";
include "/etc/bind/named.conf.custom-zones";
```

na koncu naredimo še restart bind9:
```shell
sudo systemctl restart bind9.service
```
# dodaj drop in shrani!!!

