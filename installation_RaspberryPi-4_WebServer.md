# Installation Raspberry Pi 4 WebServer
## Installation de la couche SSL :
- `sudo apt-get update`
- `sudo apt-get upgrade -y`
- `sudo apt install -y ca-certificates curl openssh-server`
- `sudo hostnamectl`

## Installation du serveur WEB :
- `sudo apt-get install apache2`

### fichier de configuration d'apache SSL :
`cd /etc/apache2/sites-available/agc-le-ssl.conf`

```
<VirtualHost _default_:443>
        ServerName agc.ddns.net 
        SSLEngine on
        DocumentRoot /var/www/html/
        ErrorLog ${APACHE_LOG_DIR}/test-mod-error.log
        CustomLog ${APACHE_LOG_DIR}/test-mod-access.log combined
        SSLOpenSSLConfCmd MinProtocol TLSv1.2
        Protocols h2 http/1.1 acme-tls/1
        <Directory />
        #       Options FollowSymLinks  
                Options None
                AllowOverride All

        </Directory>
        <Directory /var/www/multisite.test>
                Options FollowSymLinks MultiViews
                AllowOverride All
                Order allow,deny  
                allow from all  
        </Directory>
        SSLCertificateFile /var/lib/fwupd/pki/client.pem
        SSLCertificateKeyFile /var/lib/fwupd/pki/secret.key
</VirtualHost>
```

### Installation du module mod_md pour apache
mod_md est une solution pour l’obtention de certificat serveur **Let's Encrypt**.

- sudo a2enmod md
Modifié votre fichier `/etc/apache2/sites-available/agc-le-ssl.conf` et ajouter des directive **mod_md**
```
MDCertificateAgreement accepted
LogLevel md:info
MDContactEmail william.sayer@atilf.fr
MDomain agc.atilf.fr
<VirtualHost _default_:443>
        ServerName agc.atilf.fr
        SSLEngine on
        DocumentRoot /var/www/html/
        ErrorLog ${APACHE_LOG_DIR}/test-mod-error.log
        CustomLog ${APACHE_LOG_DIR}/test-mod-access.log combined
        SSLOpenSSLConfCmd MinProtocol TLSv1.2
        Protocols h2 http/1.1 acme-tls/1
        <Directory />
        #       Options FollowSymLinks  
                Options None
                AllowOverride All

        </Directory>
        <Directory /var/www/multisite.test>
                Options FollowSymLinks MultiViews
                AllowOverride All
                Order allow,deny  
                allow from all  
        </Directory>
        #SSLCertificateFile /var/lib/fwupd/pki/client.pem
        #SSLCertificateKeyFile /var/lib/fwupd/pki/secret.key
</VirtualHost>
```

Si ce n'est pas encore activer votre virtual-host :
- sudo a2ensite agc-le-ssl.conf
- sudo systemctl restart apache2.service

Un répertoire **md** est créé dans le répertoire **/etc/apache2**

### Redirection pour éviter l'affichage d'une arborescence

Créer un fichier `index.php` à mettre à chaque endroit où une arborescence de fichiers peut s'afficher et insérer le code suivant :
```
<?php
header("Location: https://agc.atilf.fr/agc/");
?>
```

## rapatriement des données de l'ancien serveur vers le nouveau :
- `sudo scp -r /Library/WebServer/Documents/agc wsayer@192.168.1.38:/tmp --> de l'ancien serveur vers le nouveau`
- `cd /var/www/html` --> sur le serveur cible (le nouveau)
- `sudo mv /tmp/agc .`
- `sudo chown -R www-data:www-data agc/`

## Installation de php : 
La dernière version disponible pour la version d'ubuntu installé.
- `sudo apt-get install php`

## Installation de Mysql :
- `sudo apt-get install mariadb-server`
- `sudo apt-get install php-pdo`
- `sudo apt-get install php-pdo-mysql`
- `sudo systemctl restart apache2.service`
- `sudo mysql_secure_installation`
- `sudo mysql -u root -p`

## Installation de phpMyAdmin :
- `sudo apt-get install phpmyadmin`

Allez dans phpmyadmin via le web est créer les deux bases de données sans créer de tables, nécessaire pour la restauration

## Rappatriement des bases de données :
- `sudo gzip -d agc.sql.gz` 
- `sudo gzip -d media.sql.gz` 
- `sudo mysql -u root -p agc < agc.sql`
- `sudo mysql -u root -p media < media.sql`

## Cas d'erreurs et procédures pour les corriger
Si vous avez fait des modifications comme par exemple un changement de serveur MYSQL (mariadb).

`could not find driver`

Ce message indique que vous n'avez pas le driver **PDO** de **MYSQL**, il faut l'installer :

`sudo apt-get install pdo-php-mysql`

SQLSTATE[HY000] [2002] No such file or directory
Ce message vous indique qu'il y a un problème de permissions au niveau de vos répertoire du site web.
Il faut donc modifier les droits **ACL** au niveau du système de fichier du répertoire racine de votre site web.
```
# cd /var/www/html
# sudo chown -R www-data:www-data agc
```

`SQLSTATE[HY000] [1130] Host 'agc.atilf.fr' is not allowed to connect to this MariaDB server` 
- Procédure pour corriger cette erreur, il faut être root, se connecter à la base de données du serveur mysql utilisé par les requêtes **PHP** de votre serveur **apache**.
```
# mysql -u root -p
mysql> GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'tonmotdepasseroot' WITH GRANT OPTION;
mysql> FLUSH PRIVILEGES;
```

## Résolution de l'arrêt intempestif du service apache :
Lors de l'arrêt du service apache, le `ps` test en temps réel si le service apache est up, donc si la commande `echo $?` est égale à `0`. si cette commande est égale à `1` alors le service apache est redémarré dans les 10 secondes après l'arrêt du service.

- `sudo -s`
- `cd /root`
- `vi apache_monitor.sh`
```
#!/bin/sh

ps auxw | grep apache2 | grep -v grep > /dev/null

if [ $? != 0 ]
then
        /etc/init.d/apache2 start > /dev/null
        mail -s "redemarrage service apache le $(date)" william.sayer@atilf.fr < /root/message
fi
```
- `vi message`
```
Le service httpd a été arrêté pour une raison inconnue.
Le service apache a été redémarré automatiquement. 
--
Le Webmaster
```

## Installation du serveur postfix avec extension mysql :
- `sudo apt install postfix-mysql`

## Configuration de postfix :
- `sudo vi /etc/postifx/master.cf`

Vérifier la ligne 11 si il y a bien un tiret au niveau de la variable `chroot`. Si vous avez un caractère `n` ou `y` alors remplacez-le par un tiret (`-`).

- `sudo vi /etc/postfix/main.cf`

````
# See /usr/share/postfix/main.cf.dist for a commented, more complete version


# Debian specific:  Specifying a file name will cause the first
# line of that file to be used as the name.  The Debian default
# is /etc/mailname.
#myorigin = /etc/mailname

smtpd_banner = $myhostname ESMTP $mail_name (Ubuntu)
biff = no

# appending .domain is the MUA's job.
append_dot_mydomain = no

# Uncomment the next line to generate "delayed mail" warnings
#delay_warning_time = 4h

readme_directory = no

# See http://www.postfix.org/COMPATIBILITY_README.html -- default to 2 on
# fresh installs.
compatibility_level = 2

smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination
myhostname = agc88.ddns.net
mydomain = ddns.net
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
#myorigin = /etc/mailname
myorigin = localhost
mydestination = agc88.ddns.net, localhost
relayhost = smtp.free.fr
mailbox_size_limit = 0
#recipient_delimiter = +
inet_interfaces = all
inet_protocols = ipv4
#mynetworks = 90.6.45.0/24
mynetworks = 192.168.1.0/24
mailbox_command = procmail -a "$EXTENSION"
recipient_delimiter = +
````

## Installation outils réseau :
- `sudo apt-get install net-tools`

## Installation de letsencrypt pour permettre la création d'un certificat serveur :
- `sudo apt-get install snapd`  --> gestionnaire de paquet qui doit déjà être installé sur la version Ubuntu 21.10
- `sudo snap install core`
- `sudo snap refresh core`
- `sudo snap install` --classic certbot
- `sudo ln -s /snap/bin/certbot /usr/bin/certbot`
- `sudo certbot --apache -d agc88.ddns.net`
- `sudo apachectl -M`
- `sudo a2enmod ssl`
- `sudo a2enmod rewrite`
- `sudo a2ensite agc.conf` 
- `sudo a2ensite agc-le-ssl.conf`
- `systemctl restart apache2 ou systemctl reload apache2`
`
## Installation de gcc et make pour permettre la compilation d'application comme no-ip :
- `sudo apt-get install gcc`
- `sudo apt-get install make`

## no-ip :
Download l'application no-ip duc sur le site de no-ip

- `sudo mv noip-duc-linux.tar.gz /usr/local/src/`
- `sudo cd /usr/local/src/`
- `sudo tar xzf noip-duc-linux.tar.gz`
- `cd noip-2.1.9-1/`
- `sudo make`
- `sudo make install`
- `Create the configuration file: /usr/local/bin/noip2 -C`

Si vous redémarrez le **Raspberry pi4** ou si il redémarre suite à une coupure de courant, le DNS dynamique ne sera plus reconnu lors de l'accès à un serveur WEB sur le **Raspberry Pi**.
Pour récupérer le nom **DNS** de votre site **WEB**, il faut aller sur le site [no-ip](https://www.noip.com/login?utm_campaign=getting-started&utm_medium=notice&utm_source=email) et supprimer le **No-IP Hostnames** et le recréé. Vous obtenez une nouvelle **adresse IP** et votre site **WEB** est à nouveau disponible.

## Sur le site de no-ip, dans votre espace client, il faut lancer la configuration du host :
- No-IP Hostnames / cliquez sur Last Update après avoir créer un host.
- Une fenêtre s'affiche : **No Dynamic Update Detected for: agc.blogsyte.com**, cliquez sur le bouton **Configure Now**
- Select Hostname, cliquez sur le bouton **Next Step**
- Connection Details, remplir les champs suivants : 
  * Router Brand : livebox --> affichera : **Orange Livebox**, qu'il faut sélectionner
  * Software/device : Raspberry --> affichera : **Raspberry Pi (Web Server)**, qu'il faut sélectionner
  * Appuyez sur le bouton **Next Step**
- Is there a computer always running on your network? --> Sélectionnez **Yes**
- Sur le **Raspberry Pi 4**, veuillez suivre la procédure indiquée pour télécharger l'application **no-ip DUC**
- quand l'installation est terminée sur le **Raspberry Pi**, appuyez sur le bouton **Next Step**
- **Port Forwarding**, tester le port `80` ou `443` pour vérifier qu'ils sont bien ouvert sur le **Raspberry Pi**. 
  * Si il affiche un message vert **port 443 is open**, alors le tour est joué, votre site est dorénavant accessible de l'extérieur via une adresse publique de **no-ip**. Plutôt un nom **DNS** car l'adresse IP publique change toutes les 24 heures car c'est du DNS dynamique, **DyDNS**.

`Àttention :` Si vous redémarrez votre **Raspberry pi**, vous serez amené à lancer la commande suivante pour que votre site web soit de nouveau accessible.
- `sudo noip2 -C`

```
Auto configuration for Linux client of no-ip.com.

Please enter the login/email string for no-ip.com  william.sayer@wanadoo.fr
Please enter the password for user 'william.sayer@wanadoo.fr'  ************

2 hosts are registered to this account.
Do you wish to have them all updated?[N] (y/N)  y
Please enter an update interval:[30]  
Do you wish to run something at successful update?[N] (y/N)  y
Please enter the script/program name  (appuyez uniquement sur la touche ENTER)

New configuration file '/usr/local/etc/no-ip2.conf' created.

```

## Brancher un disque dur externe en USB 3 pour la sauvegarde du site WEB :
### Pré-requis :
- Un disque dur formatté en **FAT** ou **NTFS** branché à votre **Raspberry Pi** en **USB**.
- Un accès SSH à votre machine.

### Installation :
- `sudo apt-get update`
- `sudo apt-get upgrade`
- `sudo apt-get install ntfs-3g`

### Configuration :
#### Identification du disque dur :
- `sudo fdisk -l`

![commande fdisk](fdisk-1.png)

- `sudo blkid /dev/sda1`


`/dev/sda1: LABEL="backup" BLOCK_SIZE="512" UUID="06A23944A239398F" TYPE="ntfs" PARTUUID="5afddfb0-01"`

#### Création du dossier cible :
- `sudo sudo mkdir /media/usb`
- `sudo chown -R wsayer:wsayer usb` --> droit de l'utilisateur du Raspberry Pi
- `sudo vi /etc/fstab`

Insérez la ligne suivante en indiquant l'**UUID** du disque dur USB que l'on a identifié à l'aide de la commande **blkid /dev/sda1**.\
`UUID="06A23944A239398F" /media/usb       ntfs    defaults,auto,rw,nofail 0       1`

- `sudo mount -a` --> monter le disque dur

Vérifier que le disque dur est bien monté :

- `df -h`

![visualisation disque /dev/sda1](df.png)

## Scripts divers
### gestion des droits du site web

Voici un petit script qui permet d'appliquer les acl **apache** sur tous les dossiers et fichiers du site WEB après une copie d'un fichier du local au distant.\
Pour ma part, j'utilise un serveur WEB local sur mon macbook pro, je ne travail qu'en local et après mes modifications, j'effectue les copies via **ssh** en utilisant la commande `scp`vers mon serveur WEB distant.

`chright_agc.sh`
```
#!/bin/bash

sudo -s chown -R www-data:www-data /var/www/html/agc
sudo -s chown -R www-data:www-data /var/www/html/images
```

