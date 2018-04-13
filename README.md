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


