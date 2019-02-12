# ProjetCarnofluxe

Ceci est un document permettant de retracer les procédure d'installation de l'ensemble des services demandés lors de ce projet

Cette infrastructure a été créer de manière virtuel ce qui peut entrainer des problème lors d'un déploiment de manière physique.

Le but du projet et de mettre en place un réseau administrateur et client. Le réseau administrateur permet d'accéder à l'ensemble des machines qui dirige le réseaux que sa soit sur le plan d'accés Web mais aussi de DNS et de DHCP . 
Le réseau administrateur se compose de 3 serveurs virtuel sous linux : 

- Un premier serveur qui nous servira de DNS maître et de DHCP que l'on nommera master. Celui-ci sera sous l'adressage IP : 192.168.10.5/24.
- Un deuxième serveur qui nous servira de DNS esclave que l'on nommera slave. Celui-ci sera sous l'adressage IP : 192.168.10.6/24.
- Un troisième serveur qui nous servira de Serveur Web sous Apache2 que l'on nomera http. Celui-ci sera sous l'adressage IP : 192.168.10.10/24.

Le réseau client sera quant à lui disponible grâce à un poste client sous windows 10 qui sera adressé automatiquement grâce au DHCP du serveur master.

## Mise en place des 3 serveurs :

Chaque machine à un accés en réseau NAT pour le réseau privé et un accés NAT pour l'accés internet.

Avant la configuration de chaque serveurs, nous devons au préalable modifier les noms des machines enfin de permettre de mettre en place le réseau pour le bon fonctionnement de notre DNS.

	gedit /etc/hostname
On a remplacer le nom par défaut de la machine du serveur master par :

	master.carnofluxe.cesi
Une fois réalisé, nous avons modifier le fichier de host afin de l'adressé correctement avec nos différente machine

	gedit /etc/hosts
Une fois ouvert, on a remplacé les valeurs par défault par :

	127.0.0.1	localhost
	127.0.1.1	master.carnofluxe.cesi
	192.168.10.5	master.carnofluxe.cesi
Puis nous avons fait un reboot de la machine.

	reboot

Ensuite nous avons configuré les interfaces de chaque machines afin de leur donner une adresse IP statique :

	gedit /etc/network/interfaces
Et nous avons rajouter en plus de la configuration par défault :

	allow-hotplug enp0s3
	iface enp0s3 inet static
	address 192.168.10.5
   	netmask 255.255.255.0
   	gateway 192.168.10.255

Ensuite nous avons modifié la configuration du DNS avec :

	gedit /etc/resolv.conf
Et nous avons remplacé les éléments du fichier par :

		domain carnofluxe.cesi
		search carnofluxe.cesi
		nameserver 192.168.10.5			#Adresse de notre machine master
		nameserver 192.168.10.6			#Adresse de notre machine slave
        
Puis pour finir la configuration de l'adressage IP, nous allons installer sur chaque machines le service Apache2 :

	apt-get install apache2
Et nous redémarrons le service Apache2 :

	service apache2 restart
Pour finir nous mettons à jour l'ensemble des paquets et des fonctionnalités de chaque serveur grâce à :

	apt-get update && apt-get dist-upgrade
Ensuite nous installons l'accés SSH des différente machine :

	apt-get install openssh-server
Une fois ces étapes réaliser nous pouvons passer au configurations des différents services sur chaque machine.

## Serveur Web (HTTP)

Dans un premier temps, nous avons créer les répertoires pour contenir les VHosts:

	mkdir /var/www/carnofluxe/
    mkdir /var/www/supervision/
Ensuite on place dans chaque dossier un fichier index.html qui sera notre page internet par défault :

	touch /var/www/supervision/index.html
	touch /var/www/carnofluxe/index.html
Chaque fichier contienne le contenu de la page internet.
Ensuite nous devons créer les VirtualHosts :

	gedit /etc/apache2/sites-available/carnofluxe.cesi.conf
	gedit /etc/apache2/sites-available/supervision.carnofluxe.cesi.conf
Pour le fichier carnofluxe.cesi.conf nous avons mis ce code :

	<VirtualHost carnofluxe.cesi:80>

       ServerName  carnofluxe.cesi
       ServerAlias  carnofluxe.cesi
       DocumentRoot /var/www/carnofluxe/
	   DirectoryIndex index.html

       <Directory /var/www/carnofluxe/>
           Require all granted
           AllowOverride All
       </Directory>
	</VirtualHost>
Pour le fichier supervision.carnofluxe.cesi.conf :

	<VirtualHost supervision.carnofluxe.cesi:80>

       ServerName  supervision.carnofluxe.cesi
       ServerAlias  supervision.carnofluxe.cesi
       DocumentRoot /var/www/supervision/
	   DirectoryIndex index.html

       <Directory /var/www/supervision/>
          Require all granted
          AllowOverride All
       </Directory>
	</VirtualHost>
Ensuite nous avons rendu acessible les 2 sites grâce à la commande :

	a2ensite carnofluxe.cesi
	a2ensite supervision.carnofluxe.cesi
Puis avons préparer la mise à disposition des scripts, nous avons installer cvs2html et curl :

	apt-get install curl python-setuptools git
    git clone https://github.com/dbohdan/csv2html.git
	python setup.py install
Notre serveur Web est fini d'être configuré, nous sommes passer à la configuration des serveurs DNS et DHCP.

## Configuration du serveur DHCP sur le DNS maître (master)

Dans un premier, on a installer le service DHCP :

	apt-get install isc-dhcp-server
Puis nous avons modifier la configuration du DHCP avec son fichier de configuration :

		gedit /etc/dhcp/dhcpd.conf
Et nous avons mis ces lignes de configurations :

	subnet 192.168.10.0 netmask 255.255.255.0{
			range 192.168.10.100 192.168.10.200;
			option domain-name "carnofluxe.cesi";
			option routers 192.168.10.254;
			option broadcast-address 192.168.88.255;
			option domain-name-servers 192.168.10.5, 192.168.10.6;
			default-lease-time 600;
			max-lease-time 7200;
	}
Puis nous relançons le service DHCP :

	/etc/init.d/isc-dhcp-server restart
Et on a vérifier si tout était fonctionnel avec la commande :

	systemctl status isc-dhcp-server
Comme le service était en marche nous sommes passé à la configuration du DNS maître.
## Configuration du DNS maître (master)

Dans un premier temps, nous avons installé le service bind9 :

	apt-get install bind9
Puis nous avons modifié les fichier de configurations named.conf.local :

	gedit /etc/bind/named.conf.local
Et nous avons modifié le fichier par :

	zone "carnofluxe.cesi" {
	type master;
	also-notify { 192.168.10.6; };
	allow-transfer { 192.168.10.6; };
	allow-update { none; };
	allow-query { any; };
	notify no;
	file "/etc/bind/db.carnofluxe.cesi";
	};

	zone "10.168.192.in-addr.arpa" {
		type master;
		also-notify { 192.168.10.6; };
		allow-transfer { 192.168.10.6; };
		allow-update { none; };
		allow-query { any; };
		notify no;
		file "/etc/bind/db.10.168.192.in-addr.arpa";
	};
Ensuite nous avons modifier le fichier de configuration named.conf.options :

	gedit /etc/bind/named.conf.options
Et l'avons remplacé par :

	options {
	directory "/var/cache/bind";
	auth-nxdomain no;
	listen-on-v6 { ::1; };
	listen-on { 192.168.10.5; };
	recursion no;
	allow-query-cache { none; };
	version none;
	additional-from-cache no;
	allow-transfer { 192.168.10.6; };
	};
Puis nous avons créer un fichier db.carnoflux.cesi dans lequel nous y avons mis :

	gedit /etc/bind/db.carnoflux.cesi
    $TTL	604800
	$ORIGIN carnofluxe.cesi.
	@	IN	SOA	master.carnofluxe.cesi. root.carnofluxe.cesi. (
			15		; Serial
			604800		; Refresh
			86400		; Retry
			2419200		; Expire
			604800 )	; Negative Cache TTL
	;
	@	IN	NS	master.carnofluxe.cesi.
	@	IN A	192.168.10.10
	master	IN	A	192.168.10.5
	www	IN	A	192.168.10.10
	supervision	IN	A	192.168.10.10
Puis nous avons créer notre fichier de reverse DNS dans lequel on y a mis :

    gedit /etc/bind/db.10.168.192.in-addr.arpa
    
    $TTL	604800
    $ORIGIN carnofluxe.cesi.
    @	IN SOA	master.carnofluxe.cesi. root.carnofluxe.cesi. (
                17		; Serial
                604800		; Refresh
                86400		; Retry
                2419200		; Expire
                604800 )	; Negative Cache TTL
    ;
    @	IN NS	master.carnofluxe.cesi.
    5	IN PTR	master.carnofluxe.cesi.
Puis nous avons restart le service bind9 par :

	service bind9 restart
Et nous avons vérifier si la configuration était bien fonctionnel:

	named-checkconf -z
Et nous avions terminer de configurer le service DNS maître.
## Configuration du DNS esclave (slave)

On a fait comme pour le maître, nous avons installé bind9 :

	apt-get install bind9
Puis nous modifions le fichier named.conf.local

	gedit /etc/bind/named.conf.local
    
    zone "carnofluxe.cesi" {
	type slave;
	masters { 192.168.10.5; };
	file "/var/lib/bind/db.carnofluxe.cesi";
    };

    zone "carnofluxe.fr" {
        type slave;
        masters { 192.168.10.5; };
        file "/var/lib/bind/db.carnofluxe.fr";
    };

    zone "10.168.192.in-addr.arpa" {
        type slave;
        masters { 192.168.10.5; };
        file "/var/lib/bind/db.10.168.192.in-addr.arpa";
    };
    
Puis nous avons modifié le fichier named.conf.options :

	gedit /etc/bind/named.conf.options
    
    options {
	directory "/var/cache/bind";
	dnssec-validation auto;
	auth-nxdomain no;
	recursion no;
	allow-query-cache { none; };
	version none;
	additional-from-cache no;
	allow-transfer { 192.168.10.5; };
	listen-on-v6 { any; };
	};
Puis nous avons restart le service bind9 :

	service bind9 restart
Et avons verifié si l'ensemble du DNS était fonctionnel avec la commande :

	named-checkconf -z

Voilà comment nous avons configuré les différentes machines de ce projet, ce projet a été réalisé grâce aux ressources : 

- https://www.supinfo.com/articles/single/1714-mise-place-serveurs-dns-maitre-esclave-avec-bind9
- https://www.supinfo.com/articles/single/1709-mise-place-serveur-dns-avec-bind9
- https://openclassrooms.com/fr/courses/2752866-mise-en-place-dun-serveur-dhcp-sous-linux

Réalisé par Florian, Julien, Baptiste et Antoine
Chef de projet :  Florian















