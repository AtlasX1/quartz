# Installing MongoDB

Now you need to install the database system to be able to store and manage data. MongoDB is a popular NoSQL database management system.

More details [MongoDB](https://www.mongodb.com/)

Consider the installation option under Ubuntu Linux.

Again, use apt to acquire and install this software:

```
sudo apt update
```

First, if not installed, download and install SSL lib(for example to ubuntu 22.04):

```bash
apt install libssl1.1
sudo apt update
sudo apt-get install gnupg
```

Note: The version matters!

Doing so would make it possible for a package update to download and install repository key on MongoDB (Check for which version you are downloading : focal - 20.04; jammy - 22.04)

```php
wget -qO - https://www.mongodb.org/static/pgp/server-4.4.asc | sudo apt-key add -
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu $(lsb_release -c --short)/mongodb-org/4.4 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.4.list
sudo apt-get update
```

After update install MongoDB

```dockerfile
sudo apt-get install -y mongodb-org=4.4.11 mongodb-org-server=4.4.11 mongodb-org-shell=4.4.11 mongodb-org-mongos=4.4.11 mongodb-org-tools=4.4.11
```

Run MongoDB

```
sudo systemctl start mongod
```

Check installation

```
sudo systemctl status mongod
```

If it shows "enabled", it's good, please proceed to the next step.


## Install Mongo Shell

Download and setup key:

```javascript
wget -qO - https://www.mongodb.org/static/pgp/server-6.0.asc | sudo apt-key add –
apt-get install gnupg
```

```dockerfile
wget -qO - https://www.mongodb.org/static/pgp/server-6.0.asc | sudo apt-key add -
```

```php
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/6.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list
```

Update repository :

```
sudo apt update
```

Mongosh supports OpenSSL. You can also configure mongosh to use your system's OpenSSL installation.

To install the latest stable version of mongosh with the included OpenSSL libraries:

```
apt install -y mongodb-mongosh
```

or

To install mongosh with your OpenSSL 1.1 libraries:

```javascript
sudo apt-get install -y mongodb-mongosh-shared-openssl11
```

or

To install mongosh with your OpenSSL 3.0 libraries:

```javascript
sudo apt-get install -y mongodb-mongosh-shared-openssl3
```

Run Mongo Shell:

```
mongosh
```

or

```javascript
mongosh "mongodb://localhost:27017"
```

To exit from mongosh type in CLI "exit"

```php
exit
```

If you will to remote connect to database, please make next steps:

- Connection refused means you probably do not have a firewall problem. Connection timeout indicates a firewall issue.
- Since you can connect locally via localhost, the error indicates that the mongo process is only listening on localhost.
- Edit the file /etc/mongod.conf. The interesting line is bindIp.
- It should look like this for IPv4 only: bindIp: 0.0.0.0
- If you have IPv6 enabled bindIp: ::,0.0.0.0 *Warning: enable authentication first. You might be hacked faster than you might expect.
## This is your stage for experimentation

1 Run mongo shell.

```
mongosh
```

2 You can create a database with your name.

Solutions

### Create database

```php
use john_doe
```

3 Insert one document into the "books" collection in the "john_doe" database.

Solutions

### Create single documents

```javascript
db.books.insertOne({title: 'MongoDB DBA Guru Notebook', description: 'A funny customized lined notebook journal for a busy MongoDB DBA employee and team member.', author: 'Glen S. Howoff', publisher: 'Independently published', pages: '102', year: '2021', tags: ['mongodb', 'database', 'NoSQL'], copies: 10 })
```

4 Insert multiple documents into the "books" collection in the "john_doe" database

Solutions

### Create multiple documents

```javascript
db.books.insertMany([
  {title: 'Mastering MongoDB 6.x', description: 'Expert techniques to run high-volume and fault-tolerant database solutions using MongoDB 6.x, 3rd Edition 3rd ed. Edition', author: 'Alex Giamas', publisher: 'Packt Publishing', pages: '460', year: '2022', tags: ['mongodb', 'database', 'NoSQL'], copies: 8 },
   {title: 'MongoDB Fundamentals', description: 'Learn how to deploy and monitor databases in the cloud, manipulate documents, visualize data, and build applications running on MongoDB using Node.js', author: 'Amit Phaltankar,Juned Ahsan,Michael Harrison ,Liviu Nedov', publisher: 'Packt Publishing', pages: '748', year: '2021', tags: ['mongodb', 'database', 'NoSQL'], copies: 8 }
])
```

5 Changing the number of copies per 1 for the first book

Solutions

### Update database

```yaml
db.books.updateOne({ title: 'MongoDB DBA Guru Notebook' },{ $set: { copies: 1 } })
```

6 Show result

Solutions

#### Show result

```
db.books.find()
```