 Learn some basic commands to install and run MySQL (MariaDB) on Ubuntu.

MariaDB Community Server is the open source relational database loved by developers all over the world. Created by MySQL’s original developers, MariaDB is compatible with MySQL and guaranteed to stay open source forever. MariaDB powers some of the world’s most popular websites such as Wikipedia and WordPress.com. It is also the core engine behind banking, social media, mobile and e-commerce sites worldwide. MariaDB Community Server is released under the GNU Public License v2. Throughout its history, MariaDB has shown its commitment to open source and the open source community. MariaDB Community Server is guaranteed open source, forever and free. In addition, commercially developed components such as MariaDB’s MaxScale are released under the Business Software License. The source code is always available, after some time the license converts to GPL assuring the software is forever available to the community.
# Installing MariaDB

Now you need to install the database system to be able to store and manage data. MariaDB is a popular database management system, a clone to MySQL and absolutely free. The command system is identical to MySQL. Note: Do not install MySQL and MariaDB together, as this may cause a conflict. More details [MariaDB](https://mariadb.org/)

Consider the installation option under Ubuntu Linux.

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
CREATE DATABASE softserve;
```

Show created databases:

```
SHOW DATABASES;
```

Use the created database:

```php
USE softserve;
```

Create a new table in to current database:

```yaml
CREATE TABLE products ( id INT(5) NOT NULL AUTO_INCREMENT, name VARCHAR(191) NULL DEFAULT NULL, price DECIMAL (18, 2) NULL DEFAULT NULL ,level TINYINT(2) NULL DEFAULT NULL, PRIMARY KEY (id)) ENGINE=InnoDB  DEFAULT CHARSET=utf8mb4;

```

Insert new data in to created table:

```javascript
INSERT INTO products (name, price, level)  
VALUES ('Salt', '19', '2'),  
('Sugar', '32', '3'),  
('Bread', '17', '0'),
('Butter', '55', '1'),  
('Milk', '32', '2');
```

Test correct inserting data:

```dockerfile
SELECT * FROM products ORDER BY name ASC;
```

Get out of MariaDB:

```php
exit
```
## This is your stage for experimentation

You can create a table "users" with next fields: ID, full name, email.
### Create a table

```sql
CREATE TABLE users ( id INT(5) NOT NULL AUTO_INCREMENT, full_name VARCHAR(191) NULL DEFAULT NULL, email VARCHAR (100) NULL DEFAULT NULL, PRIMARY KEY (id)) ENGINE=InnoDB  DEFAULT CHARSET=utf8mb4;

```

Add a 2 new users into table.
### Insert data in to table

```sql
INSERT INTO users (full_name, email)  
VALUES ('John Doe', 'john.doe@john.doe'),  
('Jane Doe', 'jane.doe@john.doe')  ;
```

Show all data from table "users" sorted by full name by descending.
### Select

```sql
 SELECT * FROM users ORDER BY full_name DESC;
```

Update Jane Doe on Judy Doe and show results
### Select

```sql
 UPDATE users 
 SET full_name = 'Judy Doe' 
 WHERE full_name = 'Jane Doe';
 SELECT * FROM users;
```

Create an INNER JOIN query that will show all products with full user records that are joined by the level and user ID fields.
### Select

```sql
SELECT * FROM products INNER JOIN users ON products.level = users.id  ORDER BY products.name;
```