Learn some basic commands to install and run LAMP (Apache 2.x, PHP 8.2, MariaDB) on Ubuntu.

LAMP is an acronym for a popular open-source software stack used for web development. It stands for Linux, Apache, MySQL, and PHP/Perl/Python.

Linux is the operating system, Apache is the web server, MySQL is the relational database management system, and PHP/Perl/Python are the server-side scripting languages used to create dynamic web pages and web applications.

The LAMP stack is widely used because all of its components are open-source and freely available, making it easy for developers to create and deploy web applications. It is also known for its stability, security, and flexibility.

LAMP is often compared to other web stacks, such as WAMP (Windows, Apache, MySQL, and PHP) and MAMP (Mac, Apache, MySQL, and PHP), which are variations of the original LAMP stack, designed to work on different operating systems.

"Apache 2" (or "Apache HTTP Server version 2") is a widely used open-source web server software developed and maintained by the Apache Software Foundation. It is the successor to the original Apache web server, and it is currently one of the most popular web servers in use on the internet.

Apache 2 is cross-platform and runs on a variety of operating systems including Linux, Unix, Windows, and macOS. It supports various programming languages such as PHP, Perl, Python, and Ruby, and can serve as a platform for running web applications built using these languages.

Some of the key features of Apache 2 include support for multiple virtual hosts, SSL/TLS encryption, URL rewriting, and load balancing. It also includes a flexible module architecture that allows developers to extend its functionality with custom modules. We can install this server by doing the following steps. Start by updating the package manager cache. If this is the first time you’re using sudo within this session, you’ll be prompted to provide your user’s password to confirm you have the right privileges to manage system packages with apt:

```
sudo apt update
```

Then, install Apache with:

```
sudo apt install apache2 -y
```

Once the installation is finished, you’ll need to adjust your firewall settings to allow HTTP traffic. Ubuntu’s default firewall configuration tool is called Uncomplicated Firewall (UFW). It has different application profiles that you can leverage. To list all currently available UFW application profiles, execute this command:

```php
sudo ufw app list
```

Here’s what each of these profiles mean:

Apache: This profile opens only port 80 (normal, unencrypted web traffic). Apache Full: This profile opens both port 80 (normal, unencrypted web traffic) and port 443 (TLS/SSL encrypted traffic). Apache Secure: This profile opens only port 443 (TLS/SSL encrypted traffic). For now, it’s best to allow only connections on port 80, since this is a fresh Apache installation and you don’t yet have a TLS/SSL certificate configured to allow for HTTPS traffic on your server.

To only allow traffic on port 80, use the Apache profile:

```javascript
sudo ufw allow in "Apache"
```

Verify the change with:

```
sudo ufw status
```

Traffic on port 80 is now allowed through the firewall. [ACCESS WEB APP](https://9d7873658005-10-244-5-115-80.spch.r.killercoda.com/)

As a result, the following page will be displayed ![apache.png](https://storage.googleapis.com/killercoda-prod-europe1/repositories/online-marathon/DevOps_dev/Setup_LAMP/assets/apache.png)

## PHP

PHP (Hypertext Preprocessor) is a popular open-source server-side scripting language used for creating dynamic web pages and web applications. It was originally designed for web development but is now also used as a general-purpose programming language.

PHP code is executed on the server and generates HTML, which is sent to the client's browser for rendering. It is used in combination with HTML, CSS, JavaScript, and other web technologies to create web applications.

PHP has many features that make it a popular choice for web development, including its simplicity, ease of use, flexibility, and ability to work with different web servers and databases. It is compatible with many popular database management systems, including MySQL, Oracle, and PostgreSQL.

PHP is also well-supported with a large community of developers, making it easy to find help, tutorials, and resources online. It is used by many popular websites such as Facebook, Wikipedia, and WordPress.

Web site - [https://php.net](https://php.net/)

Actualy version: 5.7, 7.4.x, 8.2

### Setup PHP

To get PHP 8.2 packages installed on Ubuntu we’ll use Ondrej PHP PPA which provides the latest stable versions of PHP for Ubuntu and Debian systems.

Install few dependency packages before adding the repo.

```
sudo apt install -y lsb-release gnupg2 ca-certificates apt-transport-https software-properties-common
```

Run the following commands in your terminal to add PPA to your system.

```dockerfile
sudo add-apt-repository ppa:ondrej/php
```

You can manually confirm if the repository is working by running the apt update command.

```
sudo apt update
```

Once the PPA is added, use the apt command to install PHP 8.2 and any other associated PHP modules on Ubuntu server.

```
sudo apt -y install php8.2 php8.2-mysql php8.2-gd php8.2-ldap php8.2-odbc  php8.2-xml php8.2-xmlrpc php8.2-mbstring php8.2-snmp php8.2-soap curl 
```

Check installation

```
php -v
```

# Installing MariaDB

Now that you have a web server up and running, you need to install the database system to be able to store and manage data for your site. MariaDB is a popular database management system used within PHP environments.

Again, use apt to acquire and install this software:

```
sudo apt install mariadb-server -y
```

```
sudo mysql_secure_installation
```

This will take you through a series of prompts where you can make some changes to your MariaDB installation’s security options. The first prompt will ask you to enter the current database root password. Since you have not set one up yet, press ENTER to indicate “none”. You’ll be asked if you want to switch to unix socket authentication. Since you already have a protected root account, you can skip this step. Type n and then press ENTER. The next prompt asks you whether you’d like to set up a database root password. On Ubuntu, the root account for MariaDB is tied closely to automated system maintenance, so you should not change the configured authentication methods for that account. Doing so would make it possible for a package update to break the database system by removing access to the administrative account. Type n and then press ENTER.

```
sudo systemctl status mariadb
```

First, open up the MariaDB prompt:

```
sudo mariadb
```

Create a user:

```javascript
GRANT ALL ON *.* TO 'admin'@'localhost' IDENTIFIED BY 'password' WITH GRANT OPTION;
```

Create a remote user:

```javascript
GRANT ALL ON *.* TO 'ruser'@'%' IDENTIFIED BY 'password' WITH GRANT OPTION;
```

Update database privileges:

```
FLUSH PRIVILEGES;
```

Exit from MariaDB CLI

```php
exit
```

When the firewall is enabled, add rules to open port 3306 and restart the firewall.

```yaml
sudo ufw allow 3306 & ufw reload
```

After setup you have login and create new database in to MariaDB server by use created users for example:

```
mariadb -u ruser -p
```

```
CREATE DATABASE newdb;
```

Show created databases:

```
SHOW DATABASES;
```

Use the created database:

```php
USE newdb;
```

Create a new table in to current database:

```yaml
CREATE TABLE softserve ( id INT(5) NOT NULL AUTO_INCREMENT, name VARCHAR(30) NULL DEFAULT NULL, phone VARCHAR(12) NULL DEFAULT NULL ,level TINYINT(2) NULL DEFAULT NULL, PRIMARY KEY (id)) ENGINE=InnoDB  DEFAULT CHARSET=utf8mb4;

```

Insert new data in to created table:

```javascript
INSERT INTO softserve (name, phone, level)  
VALUES ('Don', '123456789123', '2'),  
('Etta', '568569874521', '3'),  
('Irma', '987654321741', '0'),
('Barbara', '234567891789', '1'),  
('Gladys', '325647897412', '2');
```

Test correct inserting data:

```dockerfile
SELECT * FROM softserve ORDER BY name ASC;
```

Get out of MariaDB:

```php
exit
```

## Test server

Create a index.php file in to Apache root directory:

```javascript
sudo touch /var/www/html/index.php
sudo nano /var/www/html/index.php
```

Insert this example php code:

```xml
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
<html>
<body>
<?php

$hostname = "localhost";
$username = "ruser";
$password = "password";
$db = "newdb";

$dbconnect=mysqli_connect($hostname,$username,$password,$db);

if ($dbconnect->connect_error) {
  die("Database connection failed: " . $dbconnect->connect_error);
}

?>
  <figure>
    <img src="https://upload.wikimedia.org/wikipedia/commons/e/e3/SoftServe.svg" />
    <figcaption>
      <a href="https://upload.wikimedia.org/wikipedia/commons/e/e3/SoftServe.svg">SoftServe</a>, <a href="https://creativecommons.org/licenses/by/2.0">CC BY 2.0</a>, via Wikimedia Commons
    </figcaption>
  </figure>
  </a>
<table border="1" align="center">
<tr>
  <td>Reviewer Name</td>
  <td>Phone</td>
  <td>Level</td>
</tr>

<?php

$query = mysqli_query($dbconnect, "SELECT * FROM softserve")
   or die (mysqli_error($dbconnect));

while ($row = mysqli_fetch_array($query)) {
  echo
   " <tr>
    <td>{$row['name']}</td>
    <td>{$row['phone']}</td>
    <td>{$row['level']}</td>
   </tr>\n";

}

?>
</table>
</body>
</html>
```

and save the file.

Delete the index.html file and change the permission and owner for the index.php file:

```javascript
sudo rm /var/www/html/index.html
sudo chmod 755 /var/www/html/index.php
sudo chown www-data:www-data /var/www/html/index.php
```