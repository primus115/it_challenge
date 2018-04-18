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

Na konec datoteke [/etc/ansible/hosts](./ansible/hosts) dodamo serverje katere želimo konfigurirati.
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
Nova konfiguracija, ki se kopira na strežnik je definirana [tule](./ansible/config/ntp.conf).


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
Preden zaganjamo playbook, ne pozabimo kopirati svoj public key na strežnik!


Sedaj ustvarimo še [/etc/ansible/playbook/playbook.yml](./ansible/playbook.yml), v katerem definiramo kater strežnik in katere prej definirane role bomo uporabili:
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

## 5. Docker
Pripravi 2 docker containerja (z Dockerfile in docker-compose.yml):
1. container naj ima postavljen Apache Cassandra 3.11
2. container naj ima nginx z mini go aplikacijo, ki:
	- se poveze na cassandra 
	- naredi keyspace `projekt1`, ce le ta ne obstaja
	- naredi v keyspaceu tabelo `test`, ce le ta ne obstaja, ki vsebuje polje “cas”
	- doda v tabelo trenuten cas
	- vsebino tabele izpise v HTML obliki

### Postopek:

Na official [repositoriju](https://hub.docker.com/r/_/cassandra/) oz [tu](https://github.com/docker-library/cassandra/tree/88f7b82386e788634f4a0f31711c92c268640df9/3.11) najdemo Dockerfile za specifično verzijo.
Lokalno shranim obe datoteki: [Dockerfile](./docker/Dockerfile) in [docker-entrypoint.sh](./docker/docker-entrypoint.sh).
Zaradi zagona bash skripte je potrebno dodati pot do skripte v PATH sistemsko spremenljivko.

Izvršimo gradnjo docker slike in kontejnerja iz te slike:

```shell
sudo docker build .
docker run --name cassandra-dev -d cassandra:3.11

```

### Plan rešitve:
Ker sem bil žal primoran reševati naloge ponoči, mi je zmanjkalo časa za implementacijo dokončne rešitve 5. naloge.
Bom pa vseeno tu nakazal, kako sem se stvari hotel lotiti:

- kreirati še en Dockerfile z nginx in golang
- kreirati docker-compose.yml s katerim bi definiral zagon vseh service-ov in portov
- kreirati web server kot npr: https://www.sohamkamani.com/blog/2016/11/22/docker-server-busybox-golang/ (seveda na nginx+golang kontejnerju)
- s pomočjo klienta [gocql](https://github.com/gocql/gocql) realizirati vpise v db in branje celotne tabele z naslednjimi query-i:
```shell
CREATE KEYSPACE IF NOT EXISTS projekt1 WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 1};
USE projekt1;
CREATE TABLE IF NOT EXISTS test (cas timestamp,  PRIMARY KEY (cas));
INSERT INTO test (cas) values (dateOf(now()));
SELECT * FROM test
```

- implementirati zapis html tabele v smislu:
```go
var tmpl = `<tr><td>%s</td></tr>`
fmt.Printf("<table>")
names := []string{"john", "jim"}
for _, v := range names {
      fmt.Printf(tmpl, v)
}
fmt.Printf("</table>")
```
Glede na usmeritve za nginx, bi še spremenil naslednje vrstice v datoteki "src/http/ngx_http_header_filter_module.ci" in naredil compile sourca in .deb paket:  
```shell
static char ngx_http_server_string[] = "Server: MyDomain.com" CRLF;
static char ngx_http_server_full_string[] = "Server: MyDomain.com" CRLF;
```
[Vir](https://stackoverflow.com/questions/246227/how-do-you-change-the-server-header-returned-by-nginx?utm_medium=organic&utm_source=google_rich_qa&utm_campaign=google_rich_qa) rešitve.

Za public free docker register bi uprabil: (https://hub.docker.com/)



