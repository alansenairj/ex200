# ex200
Notes ex200

Orientação para o ambiente modelo Redhat

![image](https://user-images.githubusercontent.com/20565821/215319790-3da1b4fa-6919-47f3-be5d-fa7cf3b2fc5b.png)


![image](https://user-images.githubusercontent.com/20565821/215319815-6378bcd6-abbe-433f-9405-d66765f01028.png)

Config aplicada ao estudo: 
```
SRV1 - LVM 150 GB disco - httpd dnf server - registry
SRV2 - LVM 120 GB disco - srv sem GUI
SRV3 - LVM 120 GB disco - srv com GUI

3 vCPUs / 3 GB RAM para todas
```
# Montagem de servidor de repo no SRV1
### copiar ISO para o /root do host
```
mount -o loop rhel-8.6-x86_64-dvd.iso /mnt
subscription-manager register
dnf makecache
dnf install httpd -y
systemctl enable httpd --now
hostnamectl set-hostname srv1.example.com
hostname >> /etc/hosts
ip a
vi /etc/hostname
IP HOSTNAME
______

vi /etc/httpd/conf.d/rhel8.conf

<VirtualHost *:80>

        ServerAdmin teste@test
        DocumentRoot /var/www/html/rhel8
        ServerName srv1.example.com
        ErrorLog logs/srv1_error.log
        CustomLog logs/srv1_customlogs.log common

</VirtualHost>


firewall-cmd --get-services
firewall-cmd --add-service=http --permanent
firewall-cmd --reload

mkdir /var/www/html/rhel8
echo "meuteste" >> /var/www/html/rhel8/index.html
systemctl restart httpd
curl srv1.example.com

\cp -r /mnt/* /var/www/html/rhel8/
ls /var/www/html/rhel8/
umount /mnt
rm -rf /var/www/html/rhel8/index.html
mv /etc/httpd/conf.d/welcome.conf /etc/httpd/conf.d/welcome.conf.old
systemctl restart httpd
```
![image](https://user-images.githubusercontent.com/20565821/215324088-42f7e16b-fab4-4dff-9ed2-45708353483e.png)

# Configurar o registry

```
yum install -y podman httpd-tools
htpasswd -bBc /opt/registry/auth/htpasswd teste teste
cat > /etc/containers/registries.conf.d/myregistry.conf <<EOF
[[registry]] 
location = "srv1.example.com:5000"
insecure = true
blocked = false

EOF
```
### regras de f.w. e restart
```
firewall-cmd --add-port=5000/tcp --zone=internal --permanent
firewall-cmd --add-port=5000/tcp --zone=public --permanent
firewall-cmd --reload

systemctl restart podman
```

# Rodar o container
```
podman run --name myregistry \
-p 5000:5000 \
-v /opt/registry/data:/var/lib/registry:z \
-v /opt/registry/auth:/auth:z \
-e "REGISTRY_AUTH=htpasswd" \
-e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
-e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
-d \
docker.io/library/registry:latest
```
## testar - senha teste - pode usar o navegador se preferir
```
curl --location --request GET '192.168.129.115:5000/v2/_catalog' \
--header 'Authorization: Basic teste=' \
--data-raw ''
```

#### resposta
![image](https://user-images.githubusercontent.com/20565821/215327397-29c77220-7afe-4583-8182-3d7659923ba2.png)

```
{
    "repositories": []
}
```
# login
```
[root@srv2 conf.d]# podman login srv1.example.com:5000
Username: teste
Password: 
Login Succeeded!

### promover entrada no DNS do roteador/pihole
#### verificar /etc/resolv.conf 
#### editar /etc/hosts no srv e client  --->
192.168.129.115 srv1.example.com

```
# Lifecycle - apagar
```        
podman stop myregistry
podman rm myregistry
podman rm -af

```
NFS server

```
dnf install nfs-utils
mkdir /backup
chmod 777 /backup
echo "/backup 192.168.129.0/24(rw)" > /etc/exports
exportfs -r

firewall-cmd --add-service=nfs --permanent
firewall-cmd --add-service rpc-bind --permanent
firewall-cmd --add-service mountd --permanent
firewall-cmd --reload

systemctl enable --now rpcbind
systemctl restart rpc-statd.service
systemctl restart nfs-server.service

```

No client
```
 dnf install nfs-utils.x86_64
 showmount -e 192.168.129.115
  
 ```
# autofs - muito confuso as explicações por aí. Fiz mais simples.
```
cp /etc/auto.master /etc/auto.master.first
vi /etc/auto.master
#/misc  /etc/auto.misc
/backup /etc/backup.autofs --timeout=300
-------------------
vi /etc/backup.autofs
backup -fstype-auto 192.168.129.115:/backup
--------------------
systemctl restart autofs
systemctl status autofs
cd backup/
cd backup
ls -larths

```
### múltiplos shares na mesma pasta : direct e indirect -    auto.master ----> /-  /etc/backup.autofs  |    backup.autofs  --->  * -fstype-auto ip:pasta/&

1 - nfs server
mkdir teste{1,2,3}
vi /etc/exports
/backup 192.168.129.0/24(rw)
/backup/teste1 192.168.129.0/24(rw,sync,no_root_squash)
/backup/teste2 192.168.129.0/24(rw)
/backup/teste3 192.168.129.0/24(rw)
/external 192.168.129.0/24(rw)
exportfs -avr
systemctl restart nfs-server

2 - no client
showmount -e 192.168.129.115

## montar autofs direct | indirect

### share direct

/etc/auto.master.d/ ---> criar direct.autofs
/-	/etc/auto.direct

echo "/-    /etc/auto.direct " >> /etc/auto.master.d/direct.autofs

/etc/ -----> criar auto.direct
/external	-rw,sync,fstype=nfs4 192.168.129.115:/external

echo "/external	   -rw,sync,fstype=nfs4 192.168.129.115:/external" > /etc/auto.direct

### share indirect

criar /etc/auto.master.d/indirect.autofs
echo "/backup  /etc/auto.indirect" > /etc/auto.master.d/indirect.autofs

criar  /etc/auto.indirect
echo "*  	-rw,sync,fstype=nfs4	192.168.129.115:/backup/&" > /etc/auto.indirect


