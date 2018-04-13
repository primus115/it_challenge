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


