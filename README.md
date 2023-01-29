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


