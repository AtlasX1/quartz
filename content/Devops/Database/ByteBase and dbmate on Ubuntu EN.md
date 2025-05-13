Learn some basic using MySQL (MariaDB) with ByteBase shema migration tools on Ubuntu.

MariaDB Community Server is the open source relational database loved by developers all over the world. MariaDB Community Server is released under the GNU Public License v2. Throughout its history, MariaDB has shown its commitment to open source and the open source community.

Bytebase is an open-source database DevOps tool, it's the GitLab for managing databases throughout the application development lifecycle. It offers a web-based workspace for DBAs and Developers to collaborate and manage the database change safely and efficiently.

As DevOps enters the mainstream, teams are adopting tools like GitLab/GitHub for managing code, and Terraform for managing Infrastructure. Similarly, Bytebase is the tool for managing databases during application development.

Bytebase complements the existing cloud provider's database platforms or the company's internal database operation platforms. While those platforms take care of the database instance level operations (e.g. provisioning a database instance), Bytebase helps teams to use the provisioned database to build their application.

Let's start.

# Setup environment

You have already installed the MariaDB database management system and created 2 databases "staging" and "production". In "staging" create a table "tasks" and in "production" create a table "completed". Default data is inserted into both tables. Check status a database:

```
sudo systemctl status mariadb
```

#### Deploy Bytebase via Docker.

Make sure your Docker is running, and start the Bytebase Docker container with following command:

```javascript
docker run --init \
  --name bytebase \
  --platform linux/amd64 \
  --restart always \
  --publish 5678:8080 \
  --health-cmd "curl --fail http://localhost:5678/healthz || exit 1" \
  --health-interval 5m \
  --health-timeout 60s \
  --volume ~/.bytebase/data:/var/opt/bytebase \
  bytebase/bytebase:2.13.2 \
  --data /var/opt/bytebase \
  --port 8080
```

Bytebase is now running via Docker, and you can access it via localhost:5678

```javascript
docker run --rm --init \
  --name bytebase \
  --publish 8080:8080 --pull always \
  --volume ~/.bytebase/data:/var/opt/bytebase \
  bytebase/bytebase:3.5.2
```


## dbmate

Dbmate is a database migration tool that will keep your database schema in sync across multiple developers and your production servers.

It is a standalone command line tool that can be used with Go, Node.js, Python, Ruby, PHP, or any other language or framework you are using to write database-backed applications. This is especially helpful if you are writing multiple services in different languages, and want to maintain some sanity with consistent development tools. Dbmate using for RDBMS.

Details: [Overview dbmate](https://github.com/amacneil/dbmate)

#### Install dbmate

```javascript
sudo curl -fsSL -o /usr/local/bin/dbmate https://github.com/amacneil/dbmate/releases/latest/download/dbmate-linux-amd64
```

```bash
sudo chmod +x /usr/local/bin/dbmate
```

Create a directory for works

```bash
mkdir softserve
cd softserve
```

Create an .env file. This file contains database configuration data.

```php
echo "DATABASE_URL=\"mysql://staging_user:password1@127.0.0.1:3306/staging\" " > .env
```

Create a new migration for the database. This will create a directory structure. It includes the "db" folder and the "migrations" folder in it (which corresponds to the standard project directory structure). Or create this in manual.

```
dbmate n new_users_table
```

Check status

```
dbmate status
```

It`s good. Next add field in to database structure. Open created in early step the migration file in editor, for example

```
nano db/migrations/20230502053951_new_users_table.sql
```

Add next data to file:

```yaml
-- migrate:up
create table users (
  id integer,
  name varchar(100),
  email varchar(50) not null
);

-- migrate:down
drop table users;
```

Save and exit by use Ctrl-o, Enter, Ctrl-x.

Run migration

```
dbmate up
```

Check result(password mypass):

```
mariadb -u root -p
```

```php
use staging
show tables;
```

Return to CLI.

```php
exit
```