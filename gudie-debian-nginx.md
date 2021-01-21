# Debian 10 och Nginx server blocks

I guiden används portar (istället för domäner/subdomäner) för olika webbplatser i en delad servermiljö.  

- 80: nginx (default)
- 8081: Hello World
- 8082: Stöd för PHP, MariaDB, och ett CMS som Wordpress 
- 8083: Node.js med nginx som proxy server

I exemplet har servern följande ip-adress: 192.168.100.200

Uppdatera Debian
`apt-get update && apt-get upgrade`

Installera webbservern nginx
`sudo apt install nginx`

Starta en webbläsare och ange ip-adressen *hhtp://192.168.100.200*. Om installationen av nginx är rätt kommer en sida visas men den informationen. 



## Skapa en webbplats för html - site1
- port: 8081 
- sökväg: /var/www/site1 

Skapa mapp för webbplatsens filer
`sudo mkdir -p /var/www/site1`

Skapa index.html "Hello World"
`sudo nano /var/www/site1/index.html`

I index.html anger du kod för en grundläggande html sida. Skriv "Hello World" i body elementet.

*index.html*
```
<html>
<head>
    <title>Hello World</title>
</head>
<body>
    <h1>Hello World</h1>
</body>
</head>
```

Skapa en ngninx konfigurationsfil för site1
`nano /etc/nginx/sites-available/site1.conf`

I filen anges portnummer och sökvägar

*site1.conf*
```
server {
    listen         8081;
    listen         [::]:8081;
    server_name    site1 192.168.100.200;
    root           /var/www/site1;
    index          index.html;

    location / {
      try_files $uri $uri/ =404;
    }
}
```

Aktivera webbplatsen genom att länka filen i nginx 
`sudo ln -s /etc/nginx/sites-available/site1.conf /etc/nginx/sites-enabled/`

Kontrollera konfigurationsfilen
`sudo nginx -t`

Ladda om nginx för att webbplatsen ska visas  
`sudo nginx -s reload`

Starta en webbläsare och ange ip-adressen *hhtp://192.168.100.200:8081*. Nu bör du se Hello World filen du skapade.
 



### Skapa en webbplats med stöd för php, MariaDB, Wordpress - site2
- port: 8082 
- sökväg: /var/www/site2

Den här guiden är uppdelad i 3 delar

#### Del 1: php

Installera stöd för serverspråket php
`sudo apt install php-fpm`

Visa version av php
`php -v`

Ange sökvägar i php inställningar - kontrollera att versionsnummer överensstämmner (redigera ev i koden nedan - se /etc/php/*7.3*/)
`sudo sed -i 's/;cgi.fix_pathinfo=1/cgi.fix_pathinfo=0/g' /etc/php/7.3/fpm/php.ini`

Ladda om php
`sudo systemctl restart php7.3-fpm`

Skapa mapp för webbplatsens filer
`sudo mkdir -p /var/www/site2`

Skapa filem phpinfo.php som visar information om php version
`sudo nano /var/www/site2/phpinfo.php`

I filen används php kommandon `phpinfo();`

*phpinfo.php*
```
<?php
phpinfo();
?>
```

Skapa en ngninx konfigurationsfil för site2
`nano /etc/nginx/sites-available/site2.conf`

I filen anges portnummer och sökvägar för att nginx ska hantera php filer. I index anges index.php före index.html.
Se till att rätt version av php finns i sökvägen för fastcgi_pass `unix:/run/php/php7.3-fpm.sock;`  

*site2.conf*
```
server {
    listen         8082;
    listen         [::]:8082;
    server_name    example.com www.example.com 192.168.100.200;
    root           /var/www/site2;
    index          index.php index.html;

    location / {
      try_files $uri $uri/ =404;
    }

    location ~* \.php$ {
      fastcgi_pass unix:/run/php/php7.3-fpm.sock;
      include         fastcgi_params;
      fastcgi_param   SCRIPT_FILENAME    $document_root$fastcgi_script_name;
      fastcgi_param   SCRIPT_NAME        $fastcgi_script_name;
      include fastcgi.conf;
    }
}
```

Aktivera webbplatsen genom att länka filen i nginx 
`sudo ln -s /etc/nginx/sites-available/site2.conf /etc/nginx/sites-enabled/`

Kontrollera konfigurationsfilen
`sudo nginx -t`

Ladda om nginx för att webbplatsen ska visas  
`sudo nginx -s reload`

Starta en webbläsare och ange ip-adressen *http://192.168.100.200:8082/phpinfo.php*. Nu bör du se en sida som visar information om php versionen.


#### Del 2: MariaDB

Installera databasen MariaDB
`sudo apt install mariadb-server php-mysql`

Kontrollera att du kan logga in i databasen
`sudo mysql -u root`

Avsluta sessionen men databasen
`quit`

Starta en gudie för att öka säkerheten i databasen. Bekräfta och godkänn de alternativen som visas. 
`sudo mysql_secure_installation`


#### Del 3: Wordpress

Wordpress installeras genom att ladda ner en installationsfil till webbplatsen. 
För att behålla möjligheten att växla mellan Wordpress och en php-baserad webbplats utan Wordpress, skapar du en underliggande mapp men namnet cms (Content Management System).
Skapa också en mapp med namnet src för installationsfilen(er).

```
sudo mkdir /var/www/site2/cms/
sudo mkdir /var/www/site2/src/
```

Navigera till src mappen
`cd /var/www/site2/src/`

Ladda ner Wordpress
`sudo wget http://wordpress.org/latest.tar.gz`

Packa upp installationsfilen
`sudo tar -xvf latest.tar.gz`

Flytta filerna i wordpress till cms mappan 
`sudo mv wordpress/* ../cms`

Sätt rättigheter till mappen med wordpress. Ett vanligt användarnnamn är `www-data` för den typen av användare. 
`sudo chown -R www-data:www-data /var/www/site2/cms`


**Skapa en databas för wordpress**

Logga in i MariaDB
`mysql -u root -p`

Skapa en databas  
`CREATE DATABASE wordpress;`

Ange rättigheter till databasanvändaren
`GRANT ALL ON wordpress.* TO 'db-user' IDENTIFIED BY 'db-password.';`

Logga ut
`quit;`

Ändra sökvägen till root i inställningsfilen för site2. Den ska peka på underlggande mappen cms: `root /var/www/site2/cms`;
`nano /etc/nginx/sites-available/site2.conf`

*site2.conf*
```
server {
    listen         8082;
    listen         [::]:8082;
    server_name    example.com www.example.com 192.168.100.200;
    root           /var/www/site2/cms;
    index          index.php index.html;

    location / {
      try_files $uri $uri/ =404;
    }

    location ~* \.php$ {
      fastcgi_pass unix:/run/php/php7.3-fpm.sock;
      include         fastcgi_params;
      fastcgi_param   SCRIPT_FILENAME    $document_root$fastcgi_script_name;
      fastcgi_param   SCRIPT_NAME        $fastcgi_script_name;
      include fastcgi.conf;
    }
}
```

Kontrollera konfigurationsfilen
`sudo nginx -t`

Ladda om nginx för att webbplatsen med wordpress ska kunna visas  
`sudo nginx -s reload`

Starta en webbläsare och ange ip-adressen *http://192.168.100.200:8082*. Wordpress har en index.php fil, och nu bör du se en sida som visar att wordpress nu kan konfigureras.
Använd uppgifter som angavs tidigare för databas, användare och lösenord.






(sudo chown -R "$USER":www-data /var/www/html/example.com/)

sudo ufw app info "Nginx Full"

sudo ufw allow in "Nginx Full"



---------------------------------------------------------------------------------------------


### Skapa en webbplats för Node.js med nginx som proxy - site3
- port: 8083 
- sökväg: /var/www/site3 


#### Installera Node.js och npm (Node Packet Manager)

Välj version - se nodesource https://github.com/nodesource/distributions/blob/master/README.md

Ange version
`curl -sL https://deb.nodesource.com/setup_15.x | bash -`

Hämta installationsfil för Node.js
`apt-get install -y nodejs`

Installera Node.js
`sudo apt install nodejs npm`

Kontrollera version
`node -v`

Förbered installation av moduler i Node.js  (vissa paket via npm)
`sudo apt install build-essential`

Skapa mapp för webbplatsens filer
`sudo mkdir -p /var/www/site3`

#### Node.js applikation

Skapa en Hello World (en javascript fil)
`nano /var/www/site3/server.js`

I filen använder du inbyggda modulen http för att visa filer. Se till att applikationen internt använder ett eget portnummer - ex 5000.

*server.js*
```
const http = require('http');

const hostname = 'localhost';
const port = 5000;

const server = http.createServer((req, res) => {
        res.statusCode = 200;
        res.setHeader('Content-Type', 'text/plain');
        res.end('Hello World - nodejs\n');
});

server.listen(port, hostname, () => {
        console.log(`Server running at http://${hostname}:${port}/`);
});
```

#### Se till att Node.js startar automatiskt

Installera pm2 (globalt)
`sudo npm install -g pm2`

Starta applikationen med pm2
`pm2 start server.js`

Ange att pm2 automatiskt startar upp appen vid omstart av system 
`pm2 startup systemd`


#### Konfigurera nginx som proxy server
I konfigurationsfilen ange att data hanteras till det portnummer för Node.js appen 

Skapa en ngninx konfigurationsfil för site3 
`nano /etc/nginx/sites-available/site3.conf`

*site3.conf*
------------
server {
    listen         8080;
    listen         [::]:8080;
    server_name    example.com www.example.com 192.46.235.70;

    location / {
        proxy_set_header   X-Forwarded-For $remote_addr;
        proxy_set_header   Host $http_host;
        proxy_pass         http://localhost:5000;
    }
}
------------


Aktivera webbplatsen genom att länka filen i nginx 
`sudo ln -s /etc/nginx/sites-available/site3.conf /etc/nginx/sites-enabled/`

Kontrollera konfigurationsfilen
`sudo nginx -t`

Ladda om nginx för att webbplatsen ska visas  
`sudo nginx -s reload`

Starta en webbläsare och ange ip-adressen *http://192.168.100.200:8083*. Nu bör du se en sida som visar Node.js appen.

