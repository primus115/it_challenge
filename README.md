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

## 4. Ansible - server configuration
Izberi si poljuben configuration management tool (puppet, ansible, salt, …) in z njim:
- namesti ntp streznik, ki bo uporabljal slovenske ntp streznike.
- nastavi ssh streznik, da bo dovoljeval prijavo samo s nasimi SSH kljuci

### Postopek:
Nameščanje ansible orodja na računalnik iz katerega bomo izvajali konfiguracije:
```shell
sudo apt-add-repository -y ppa:ansible/ansible
sudo apt-get update
sudo apt-get install -y ansible
```

Na konec datoteke [/etc/ansible/host](./ansible/host) dodamo serverje katere želimo konfigurirati.
```shell
server_3fs ansible_port=22 ansible_host=37.139.28.164 ansible_ssh_user=primoz
```

Naloge sem razdelil na "taske" definirane v "roles", kateri se kličejo v "playbook"-u.
Ustvarimo potrebno strukturo map in v datoteko [/etc/ansible/roles/ntp/tasks/main.yml](./ansible/roles/ntp/tasks/main.yml) dodamo potrebne definicije:

```shell
- name: Installing ntp
  apt: pkg=ntp state=present
#  notify: 
#  - copy-config

- name: copy-config
  copy:
    src: config/ntp.conf
    dest: /etc/ntp.conf
    owner: root
    group: root
    mode: 0644
    backup: yes
#  notify:
#  - restart-ntp

- name: restart-ntp
  service: name=ntp state=restarted enabled=yes
```
Nova konfiguracija, ki se kopira na strežnik je definirana [tule](config/ntp.conf).


Za definicije okoli sshd-ja pa ustvarimo drugo datoteko: [/etc/ansible/roles/ssh/tasks/main.yml](./ansible/roles/ssh/tasks/main.yml)
```shell
- name: Restrict SSH logins to keys only
  lineinfile:
    dest: /etc/ssh/sshd_config
    state: present
    regexp: '^PasswordAuthentication'
    line: 'PasswordAuthentication no'

- name: Restrict SSH logins to keys only regex2
  lineinfile:
    dest: /etc/ssh/sshd_config
    state: absent
    regexp: '^#PasswordAuthentication'
    line: 'PasswordAuthentication no'


- name: Restart sshd
  service: name=sshd state=restarted enabled=yes
```


Sedaj ustvarimo še [/etc/ansible/playbook/playbook.yml](./ansible/playbook/playbook.yml), v katerem definiramo kater strežnik in katere prej definirane role bomo uporabili:
```shell
- hosts: server_3fs
  become: true #root
  roles:
  - ntp
  - ssh
```

Playbook poženemo z ukazom:
```shell
ansible-playbook -K playbook.yml
```

Na koncu lahko še preverimo delovnje ntp-ja:
```shell
ansible -m shell -a "ntpq -p;date" server_3fs
```


TODO:
# dodaj drop in shrani!!!
preveri dns ker z dropom neha delovati
