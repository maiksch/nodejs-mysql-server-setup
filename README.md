# Wie man einen Node.js + MySQL Webserver auf Ubuntu aufsetzt

## MySQL

Der erste Schritt besteht darin, auf dem Ubuntu Server die MySQL Datenbank einzurichten.

### Installation

Zunächst wird MySQL installiert.

```
$ sudo apt update
$ sudo apt upgrade
$ sudo apt install mysql-server
```

Im Anschluss können mit dem Secure Installation-Skript einige unsichere default Einstellungen umgestellt werden. Dazu zählt ein sicheres Passwort und keinen Remote-Zugriff für den Root User.

```
$ sudo mysql_secure_installation
```

Zunächst is es weiterhin möglich, sich als Root ohne Passwort anzumelden. Um dies zu verhindern, muss die Authentifizierungsmethode verändert werden. Dazu den folgenden Befehl ausführen und 'password' mit dem zuvor gewählten Passwort ersetzen.

```SQL
mysql> ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'password';
mysql> FLUSH PRIVILEGES;
```

### Datenbank Anlegen

Als nächstes wird die anwendungsspezifische Datenbank angelegt. Das tun wir als nächstes, damit wir im Anschluss einen Benutzer erstellen können, der nur Zugriff auf diese Datenbank hat. DATENBANK wird durch den Datenbanknamen ersetzt.

```SQL
mysql> CREATE DATABASE DATENBANK;
```

### Anwendungsbenutzer anlegen

Jetzt wird der Benutzer angelegt. Dazu 'name' und 'password' mit den entsprechenden Werten füllen. Außerdem wird DATENBANK durch den im vorherigen Schritt gewählten Datenbanknamen ausgetauscht. Dadurch erhält der Benutzer nur Zugriff auf diese Datenbank.

```SQL
mysql> CREATE USER 'name'@'localhost' IDENTIFIED BY 'password';
mysql> GRANT ALL PRIVILEGES ON DATENBANK.* TO 'name'@'localhost' WITH GRANT OPTION;
mysql> FLUSH PRIVILEGES;
```

## Node.js

Nachdem die Datenbank eingerichtet wurde, kann nun der Anwendungsserver aufgesetzt werden. In diesem Fall wird ein Node Server mit der Version 12 verwendet.

### Installation

Zunächst benötigen wir die richtige Version von Node als Personal Package Archive (PPA). PPAs dienen als Repository um bestimmte Versionen eines Packets installieren zu können.

Um Node 12 zu installieren wird also das dazugehörige PPA heruntergeladen und direkt installiert.

```
$ curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -
$ sudo apt install nodejs
```

Hat die Installation geklappt, sollte `node -v` die entsprechende Version anzeigen.

### App auf den Server kopieren

Um das Aufsetzen des Servers so kurz wie möglich zu halten, werden die Javascript-Dateien der App direkt per FTP auf den Server kopiert. Im Idealfall sollte hierfür ein automatisierter Deployment-Prozess aufgesetzt werden.

Als erstes wird per SSH noch der Anwendungsordner angelegt

```
$ cd /opt
$ mkdir app
```

Im Anschluss wird zu einem beliebigen FTP-Client gewechselt, mit dem nun die Javascript-Dateien in den eben erstellten Ordner hochgeladen werden.

Es empfiehlt sich den `node_modules` Ordner dabei nicht mit zu kopieren, sondern stattdessen die Dependencies auf dem Server selbst zu installieren. Dafür die `package.json` und `package-lock.json` in den Anwendungsordner hochladen und dort wieder per SSH `npm install` ausführen.

## Service einrichten

Damit die Anwendung bei jedem Server Neustart automatisch ebenfalls startet, kann ein Service eingerichtet werden.

### Service erstellen

Als erstes muss das Main Script der Anwendung zu einem Executable gemacht werden, damit es vom Service ausgeführt werden kann.

```
$ chmod +x /opt/app/main.js
```

Im Anschluss wird die Konfigurationsdatei für den Service angelegt. Diese liegen generell unter `/etc/systemd/system`.

```
$ sudo nano /etc/systemd/system/api.service
```

Mit Inhalt befüllen.

```
[Unit]
Description=Node.js Application Server for API
Requires=network.target
After=network.target

[Service]
Restart=always
RestartSec=3
User=root
WorkingDirectory=/opt/app
ExecStart=/usr/bin/node main.js

[Install]
WantedBy=multi-user.target
Alias=api.service
```

### Environment aufsetzen

Keine App sollte Passwörter oder andere sensible Informationen direkt im Quellcode stehen haben, sondern diese Daten per Umgebungsvariablen laden. Ein Beispiel dafür sind die Anmeldedaten für die Datenbank oder verwendete Salts zum hashen von Passwortern.

Wird systemd verwendet um den Server als Service laufen zu lassen, müssen die Umgebungsvariablen nicht wie üblich unter /etc/environment, sondern in der Service konfiguration definiert werden.

```
$ sudo systemctl edit api.service
```

Hier wird nun in jeder Zeile eine Umgebungsvariable in Form

```
Environment="NAME=WERT"
```

definiert.

```
[Service]
Environment="DB_ADDRESS=localhost"
Environment="DB_PORT=3306"
Environment="DB_NAME=Datenbank"
Environment="DB_USER=user"
Environment="DB_PASSWORD=geheim"
```

Damit die Änderungen an den Umgebungsvariablen wirksam werden, wird der Service neu gestartet.

```
$ systemctl restart api
```

## Nginx

Nginx dient als Reverse Proxy und verschleiert damit nach außen hin unseren Node Server.

### Installation

Zunächst wird wie gewohnt das Nginx Paket installiert.

```
$ sudo apt install nginx
```

### Firewall Einstellungen

Im Anschluss muss die Firewall so eingerichtet werden, dass die passenden Ports durchgelassen werden. Davür zunächst die uncomplicated Firewall (UFW) erlauben.

```
$ sudo ufw enable
```

Um SSH Verbindungen durch die neue Firewall weiterhin zu ermöglichen, muss der Port 22 freigegeben werden.

```
$ sudo ufw allow ssh
```

Im Anschluss wird Nginx freigegeben. Dafür gibt es spezielle Anwendungsprofile, eine Liste davon kann sich folgendermaßen angezeigt werden:

```
$ sudo ufw app list
```

Für HTTP Verbindungen wird dann das Nginx HTTP Profil gewählt. Somit ist es nun möglich über den Port 80 den Server aus dem Internet zu erreichen.

```
$ sudo ufw allow 'Nginx HTTP'
```

Ob dies funktioniert, kann getestet werden, indem die öffentliche IP-Adresse des Servers im Browser eingegeben wird. Wenn die "Welcome to nginx" Seite erscheint, hat alles funktioniert.

### Einrichten als Reverse Proxy

Abschließend soll nun Nginx alle Verbindungen an den Node Server weiterleiten. Dafür wird er als Reverse Proxy konfiguriert.

Die Konfigurationsdateien liegen unter `/etc/nginx/sites-available`. Dort muss zunächst eine neue Konfigurationsdatei angelegt werden.

```
$ sudo nano app-reverse-proxy.conf
```

Und mit folgendem Inhalt gefüllt werden. Achte darauf, unter `proxy_pass` den richtigen Port der Node Anwendung anzugeben. Üblicherweise ist dies 3000.

```
server {
        listen 80;
        listen [::]:80;

        access_log /var/log/nginx/app-access.log;
        error_log /var/log/nginx/app-error.log;

        location / {
                proxy_pass http://127.0.0.1:3000;
        }
}
```

Danach wird diese Konfiguration in das `sites-enabled` verzeichnet kopiert. Dazu ist es empfohlen, einen symbolic link zu verwenden anstatt die Datei tatsächlich doppelt zu speichern.

```
$ ln -s /etc/nginx/sites-available/app-reverse-proxy.conf /etc/nginx/sites-enabled/app-reverse-proxy.conf
```

Zu guter letzt muss die Konfiguration nur noch auf Fehler geprüft und bei Erfolg neu geladen werden, damit wir über die öffentliche IP-Adresse nun unseren Node Server erreichen.

```
$ sudo nginx -t
$ sudo nginx -s reload
```

## TODO: SSL mit Let's Encrypt

### Certbot Installieren

```
sudo add-apt-repository ppa:certbot/certbot
sudo apt install python-certbot-nginx
sudo nano /etc/nginx/sites-available/app-reverse-proxy.conf
```

Zugewiesene Domain in Nginx konfigurieren

```
...
server_name example.com www.example.com;
...
```

### Firewall einstellen

```
sudo ufw allow 'Nginx Full'
sudo ufw delete allow 'Nginx HTTP'
```

### Zertifikat beantragen

SSL Zertifikat beantragen. Für jede Domain -d

```
sudo certbot --nginx -d example.com
```
