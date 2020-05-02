# Wie man einen Node.js + MySQL Webserver auf Ubuntu aufsetzt

Outline:

1. mysql
   - installieren
   - datenbank anlegen
   - benutzer anlegen
2. nodej app
   - node installieren
   - environment variables setzen
   - app auf den server kopieren
3. nginx reverse proxy
   - nginx installieren
   - konfigurieren

## MySQL

## Installation

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

## Datenbank Anlegen

Als nächstes wird die anwendungsspezifische Datenbank angelegt. Das tun wir als nächstes, damit wir im Anschluss einen Benutzer erstellen können, der nur Zugriff auf diese Datenbank hat. DATENBANK wird durch den Datenbanknamen ersetzt.

```SQL
mysql> CREATE DATABASE DATENBANK;
```

## Anwendungsbenutzer anlegen

Jetzt wird der Benutzer angelegt. Dazu 'name' und 'password' mit den entsprechenden Werten füllen. Außerdem wird DATENBANK durch den im vorherigen Schritt gewählten Datenbanknamen ausgetauscht. Dadurch erhält der Benutzer nur Zugriff auf diese Datenbank.

```SQL
mysql> CREATE USER 'name'@'localhost' IDENTIFIED BY 'password';
mysql> GRANT ALL PRIVILEGES ON DATENBANK.* TO 'name'@'localhost' WITH GRANT OPTION;
mysql> FLUSH PRIVILEGES;
```
