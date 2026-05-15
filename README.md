# TP Reverse Proxy Apache — Debian 12

Documentation complète pour refaire un TP avec :

- 1 serveur Reverse Proxy Apache
- 1 serveur Web1 Apache/PHP
- 1 serveur Web2 Apache/PHP
- 4 noms de domaine locaux
- Redirection via Reverse Proxy

## Architecture

```text
Poste Windows
    |
    | Fichier hosts Windows
    v
ReverseProxy : 172.20.10.10
    |
    |-- web1.fr      -> Web1 : 172.20.10.11
    |-- web2.fr      -> Web1 : 172.20.10.11
    |-- www.web1.fr  -> Web2 : 172.20.10.12
    |-- www.web2.fr  -> Web2 : 172.20.10.12
```
| Machine | Rôle | IP |
|---|---|---|
| ReverseProxy | Reverse proxy Apache | `172.20.10.10/28` |
| Web1 | Serveur Apache/PHP pour `web1.fr` et `web2.fr` | `172.20.10.11/28` |
| Web2 | Serveur Apache/PHP pour `www.web1.fr` et `www.web2.fr` | `172.20.10.12/28` |
| Passerelle | Réseau local | `172.20.10.1` |
| DNS | Résolution DNS | `8.8.8.8` |

Configuration VirtualBox

Pour les 3 VM :

Mode d'accès réseau : Accès par pont
Nom de carte        : même carte réseau sur les 3 VM
Mode promiscuité    : Autoriser les VMs
Câble branché       : coché

Exemple :

ReverseProxy -> Accès par pont
Web1         -> Accès par pont
Web2         -> Accès par pont

Les 3 VM doivent utiliser la même carte réseau physique.

1. Configuration réseau
1.1 ReverseProxy

Sur la VM ReverseProxy :

nano /etc/network/interfaces

Contenu :

auto lo
iface lo inet loopback

auto enp0s3
iface enp0s3 inet static
    address 172.20.10.10
    netmask 255.255.255.240
    gateway 172.20.10.1
    dns-nameservers 8.8.8.8

Appliquer :

systemctl restart networking

Si le service réseau échoue :

reboot

Vérifier :

ip -4 a
ip route
1.2 Web1

Sur la VM Web1 :

nano /etc/network/interfaces

Contenu :

auto lo
iface lo inet loopback

auto enp0s3
iface enp0s3 inet static
    address 172.20.10.11
    netmask 255.255.255.240
    gateway 172.20.10.1
    dns-nameservers 8.8.8.8

Appliquer :

systemctl restart networking

Ou :

reboot
1.3 Web2

Sur la VM Web2 :

nano /etc/network/interfaces

Contenu :

auto lo
iface lo inet loopback

auto enp0s3
iface enp0s3 inet static
    address 172.20.10.12
    netmask 255.255.255.240
    gateway 172.20.10.1
    dns-nameservers 8.8.8.8

Appliquer :

systemctl restart networking

Ou :

reboot
1.4 Tests réseau

Depuis ReverseProxy :

ping -c 3 172.20.10.11
ping -c 3 172.20.10.12

Résultat attendu :

64 bytes from 172.20.10.11
64 bytes from 172.20.10.12

Si les pings ne répondent pas, ne pas continuer Apache. Il faut d'abord corriger le réseau.

2. Configuration du serveur Web1

Web1 héberge :

web1.fr
web2.fr
2.1 Installation Apache + PHP-FPM

Sur Web1 :

apt update
apt install apache2 php8.2-fpm curl -y

Activer les modules Apache nécessaires :

a2enmod proxy_fcgi
a2enmod setenvif

Démarrer PHP-FPM :

systemctl enable php8.2-fpm
systemctl restart php8.2-fpm

Vérifier le socket PHP :

ls /run/php/

Résultat attendu :

php8.2-fpm.sock
2.2 Créer les dossiers des sites
mkdir -p /var/www/web1
mkdir -p /var/www/web2

Créer les pages de test :

echo '<?php echo "WEB1 - PHP " . phpversion(); ?>' > /var/www/web1/index.php
echo '<?php echo "WEB2 - PHP " . phpversion(); ?>' > /var/www/web2/index.php

Droits :

chown -R www-data:www-data /var/www/web1 /var/www/web2
chmod -R 755 /var/www/web1 /var/www/web2
2.3 Vhost web1.fr
nano /etc/apache2/sites-available/web1.fr.conf

Contenu :

<VirtualHost *:80>
    ServerName web1.fr
    DocumentRoot /var/www/web1
    DirectoryIndex index.php

    <Directory /var/www/web1>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php/php8.2-fpm.sock|fcgi://localhost/"
    </FilesMatch>
</VirtualHost>
2.4 Vhost web2.fr
nano /etc/apache2/sites-available/web2.fr.conf

Contenu :

<VirtualHost *:80>
    ServerName web2.fr
    DocumentRoot /var/www/web2
    DirectoryIndex index.php

    <Directory /var/www/web2>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php/php8.2-fpm.sock|fcgi://localhost/"
    </FilesMatch>
</VirtualHost>
2.5 Activer les sites Web1
a2dissite 000-default.conf
a2ensite web1.fr.conf
a2ensite web2.fr.conf
apache2ctl configtest

Si le résultat est :

Syntax OK

Redémarrer :

systemctl restart php8.2-fpm
systemctl restart apache2

Tester localement :

curl -H "Host: web1.fr" http://127.0.0.1
curl -H "Host: web2.fr" http://127.0.0.1

Résultat attendu :

WEB1 - PHP 8.2.x
WEB2 - PHP 8.2.x
3. Configuration du serveur Web2

Web2 héberge :

www.web1.fr
www.web2.fr
3.1 Installation Apache + PHP-FPM

Sur Web2 :

apt update
apt install apache2 php8.2-fpm curl -y

Activer les modules Apache nécessaires :

a2enmod proxy_fcgi
a2enmod setenvif

Démarrer PHP-FPM :

systemctl enable php8.2-fpm
systemctl restart php8.2-fpm

Vérifier :

ls /run/php/

Résultat attendu :

php8.2-fpm.sock
3.2 Créer les dossiers des sites
mkdir -p /var/www/www.web1.fr
mkdir -p /var/www/www.web2.fr

Créer les pages de test :

echo '<?php echo "WWW WEB1 - PHP " . phpversion(); ?>' > /var/www/www.web1.fr/index.php
echo '<?php echo "WWW WEB2 - PHP " . phpversion(); ?>' > /var/www/www.web2.fr/index.php

Droits :

chown -R www-data:www-data /var/www/www.web1.fr /var/www/www.web2.fr
chmod -R 755 /var/www/www.web1.fr /var/www/www.web2.fr
3.3 Vhost www.web1.fr
nano /etc/apache2/sites-available/www.web1.fr.conf

Contenu :

<VirtualHost *:80>
    ServerName www.web1.fr
    DocumentRoot /var/www/www.web1.fr
    DirectoryIndex index.php

    <Directory /var/www/www.web1.fr>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php/php8.2-fpm.sock|fcgi://localhost/"
    </FilesMatch>
</VirtualHost>
3.4 Vhost www.web2.fr
nano /etc/apache2/sites-available/www.web2.fr.conf

Contenu :

<VirtualHost *:80>
    ServerName www.web2.fr
    DocumentRoot /var/www/www.web2.fr
    DirectoryIndex index.php

    <Directory /var/www/www.web2.fr>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php/php8.2-fpm.sock|fcgi://localhost/"
    </FilesMatch>
</VirtualHost>
3.5 Activer les sites Web2
a2dissite 000-default.conf
a2ensite www.web1.fr.conf
a2ensite www.web2.fr.conf
apache2ctl configtest

Si le résultat est :

Syntax OK

Redémarrer :

systemctl restart php8.2-fpm
systemctl restart apache2

Tester localement :

curl -H "Host: www.web1.fr" http://127.0.0.1
curl -H "Host: www.web2.fr" http://127.0.0.1

Résultat attendu :

WWW WEB1 - PHP 8.2.x
WWW WEB2 - PHP 8.2.x
4. Configuration ReverseProxy

Le reverse proxy reçoit les requêtes et les redirige vers Web1 ou Web2.

4.1 Installation Apache + curl

Sur ReverseProxy :

apt update
apt install apache2 curl -y

Activer les modules reverse proxy :

a2enmod proxy
a2enmod proxy_http

Redémarrer Apache :

systemctl restart apache2
4.2 Tester l'accès à Web1 depuis ReverseProxy
curl -H "Host: web1.fr" http://172.20.10.11
curl -H "Host: web2.fr" http://172.20.10.11

Résultat attendu :

WEB1 - PHP 8.2.x
WEB2 - PHP 8.2.x
4.3 Tester l'accès à Web2 depuis ReverseProxy
curl -H "Host: www.web1.fr" http://172.20.10.12
curl -H "Host: www.web2.fr" http://172.20.10.12

Résultat attendu :

WWW WEB1 - PHP 8.2.x
WWW WEB2 - PHP 8.2.x
4.4 Désactiver le site par défaut
a2dissite 000-default.conf
4.5 Reverse proxy pour web1.fr
nano /etc/apache2/sites-available/web1.fr.conf

Contenu :

<VirtualHost *:80>
    ServerName web1.fr

    ProxyPreserveHost On
    ProxyRequests Off

    ProxyPass / http://172.20.10.11/
    ProxyPassReverse / http://172.20.10.11/
</VirtualHost>
4.6 Reverse proxy pour web2.fr
nano /etc/apache2/sites-available/web2.fr.conf

Contenu :

<VirtualHost *:80>
    ServerName web2.fr

    ProxyPreserveHost On
    ProxyRequests Off

    ProxyPass / http://172.20.10.11/
    ProxyPassReverse / http://172.20.10.11/
</VirtualHost>
4.7 Reverse proxy pour www.web1.fr
nano /etc/apache2/sites-available/www.web1.fr.conf

Contenu :

<VirtualHost *:80>
    ServerName www.web1.fr

    ProxyPreserveHost On
    ProxyRequests Off

    ProxyPass / http://172.20.10.12/
    ProxyPassReverse / http://172.20.10.12/
</VirtualHost>
4.8 Reverse proxy pour www.web2.fr
nano /etc/apache2/sites-available/www.web2.fr.conf

Contenu :

<VirtualHost *:80>
    ServerName www.web2.fr

    ProxyPreserveHost On
    ProxyRequests Off

    ProxyPass / http://172.20.10.12/
    ProxyPassReverse / http://172.20.10.12/
</VirtualHost>
4.9 Activer les 4 sites
a2ensite web1.fr.conf
a2ensite web2.fr.conf
a2ensite www.web1.fr.conf
a2ensite www.web2.fr.conf
apache2ctl configtest

Si le résultat est :

Syntax OK

Recharger Apache :

systemctl reload apache2
5. Test complet depuis ReverseProxy

Sur ReverseProxy :

curl -H "Host: web1.fr" http://127.0.0.1
curl -H "Host: web2.fr" http://127.0.0.1
curl -H "Host: www.web1.fr" http://127.0.0.1
curl -H "Host: www.web2.fr" http://127.0.0.1

Résultat attendu :

WEB1 - PHP 8.2.x
WEB2 - PHP 8.2.x
WWW WEB1 - PHP 8.2.x
WWW WEB2 - PHP 8.2.x
6. Configuration du fichier hosts Windows

Sur Windows, ouvrir le Bloc-notes en administrateur.

Ouvrir le fichier :

C:\Windows\System32\drivers\etc\hosts

Ajouter :

172.20.10.10    web1.fr
172.20.10.10    web2.fr
172.20.10.10    www.web1.fr
172.20.10.10    www.web2.fr

Enregistrer.

7. Test depuis Windows

Dans le navigateur Windows :

http://web1.fr
http://web2.fr
http://www.web1.fr
http://www.web2.fr

Résultat attendu :

web1.fr      -> WEB1 - PHP 8.2.x
web2.fr      -> WEB2 - PHP 8.2.x
www.web1.fr  -> WWW WEB1 - PHP 8.2.x
www.web2.fr  -> WWW WEB2 - PHP 8.2.x
8. Option : PHP 7.4 sur Web2

Si le sujet demande PHP 7.4 pour les sites www.*, installer PHP 7.4 sur Web2.

8.1 Ajouter le dépôt Sury

Sur Web2 :

apt install curl lsb-release ca-certificates apt-transport-https -y
curl -fsSL https://packages.sury.org/php/apt.gpg -o /usr/share/keyrings/sury-php.gpg
echo "deb [signed-by=/usr/share/keyrings/sury-php.gpg] https://packages.sury.org/php/ $(lsb_release -sc) main" > /etc/apt/sources.list.d/php.list
apt update
8.2 Installer PHP 7.4
apt install php7.4 php7.4-fpm -y

Démarrer PHP 7.4 :

systemctl enable php7.4-fpm
systemctl restart php7.4-fpm

Vérifier :

ls /run/php/

Résultat attendu :

php7.4-fpm.sock
8.3 Modifier les vhosts Web2

Modifier :

nano /etc/apache2/sites-available/www.web1.fr.conf

Remplacer :

SetHandler "proxy:unix:/run/php/php8.2-fpm.sock|fcgi://localhost/"

par :

SetHandler "proxy:unix:/run/php/php7.4-fpm.sock|fcgi://localhost/"

Faire pareil :

nano /etc/apache2/sites-available/www.web2.fr.conf

Puis :

apache2ctl configtest
systemctl restart apache2

Tester sur Web2 :

curl -H "Host: www.web1.fr" http://127.0.0.1
curl -H "Host: www.web2.fr" http://127.0.0.1

Résultat attendu :

WWW WEB1 - PHP 7.4.x
WWW WEB2 - PHP 7.4.x
9. Dépannage
Erreur : Site does not exist

Vérifier que le fichier est dans le bon dossier :

ls /etc/apache2/sites-available/

Bon chemin :

/etc/apache2/sites-available/nom-du-site.conf

Mauvais chemin :

/etc/network/sites-available/
Erreur : mauvaise extension

Bon :

web1.fr.conf

Mauvais :

web1.fr.config
Erreur : Invalid command ProxyPreserveHost

Activer les modules proxy :

a2enmod proxy
a2enmod proxy_http
apache2ctl configtest
systemctl reload apache2
Erreur : ProxyPreserverHost ou ProxyPreservevHost

La bonne directive est :

ProxyPreserveHost On
Erreur : Expected </VirtualHost>

Vérifier la dernière ligne du fichier :

</VirtualHost>
Erreur : 503 Service Unavailable

Vérifier PHP-FPM :

systemctl status php8.2-fpm --no-pager
ls /run/php/

Le socket dans le vhost doit correspondre à un fichier existant :

SetHandler "proxy:unix:/run/php/php8.2-fpm.sock|fcgi://localhost/"
Erreur : 403 Forbidden

Vérifier que index.php existe :

ls -la /var/www/web1
ls -la /var/www/web2

Corriger les droits :

chown -R www-data:www-data /var/www/web1 /var/www/web2
chmod -R 755 /var/www/web1 /var/www/web2
Erreur : les VM ne se ping pas

Vérifier VirtualBox :

Accès par pont
Même carte réseau
Mode promiscuité : Autoriser les VMs
Câble branché : coché

Vérifier les IP :

ip -4 a
ip route
10. Commandes utiles

Tester la configuration Apache :

apache2ctl configtest

Redémarrer Apache :

systemctl restart apache2

Recharger Apache :

systemctl reload apache2

Voir les sites disponibles :

ls /etc/apache2/sites-available/

Voir les sites activés :

ls /etc/apache2/sites-enabled/

Activer un site :

a2ensite nom-du-site.conf

Désactiver un site :

a2dissite nom-du-site.conf

Activer un module :

a2enmod nom-du-module

Voir les IP :

ip -4 a

Voir les routes :

ip route

Tester un domaine avec curl :

curl -H "Host: web1.fr" http://127.0.0.1
