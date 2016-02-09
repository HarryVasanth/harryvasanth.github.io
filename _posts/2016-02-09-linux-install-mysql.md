---
layout: post
title: "Linux - Install MySQL"
date: 2016-02-09 14:12:44 +01
categories: linux hosting
tags: hosting mysql
---

## Installation

Use `apt` to install MySQL server:

```shell
sudo apt install mysql-server
```

```shell
sudo mysql
```

```sql
mysql> ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'new_password';
```

Start the interactive script by running:

```shell
sudo mysql_secure_installation
```

Answer `Y` for yes, or anything else to continue without enabling.

```shell
VALIDATE PASSWORD PLUGIN can be used to test passwords
and improve security. It checks the strength of password
and allows the users to set only those passwords which are
secure enough. Would you like to setup VALIDATE PASSWORD plugin?

Press y|Y for Yes, any other key for No:
```

Choose the option `1` MEDIUM:

```shell
There are three levels of password validation policy:

LOW    Length >= 8
MEDIUM Length >= 8, numeric, mixed case, and special characters
STRONG Length >= 8, numeric, mixed case, special characters and dictionary                  file

Please enter 0 = LOW, 1 = MEDIUM and 2 = STRONG: 1
```

```shell
Using existing password for root.

Estimated strength of the password: 100

Change the password for root ? ((Press y|Y for Yes, any other key for No) : n
```

For the rest of the questions, press `Y` and hit the `ENTER` key at each prompt. This will remove some anonymous users and the test database, disable remote root logins, and load these new rules so that MySQL immediately respects the changes you have made.

## Configuration

Let’s change the authentication method from `auth_socket` to `mysql_native_password`. To do this, open up the MySQL prompt from your terminal and check which authentication method each of your MySQL user accounts uses:

```shell
sudo mysql
```

```sql
mysql> SELECT user,authentication_string,plugin,host FROM mysql.user;
```

The output should look like this:

```shell
+------------------+-------------------------------------------+-----------------------+-----------+
| user             | authentication_string                     | plugin                | host      |
+------------------+-------------------------------------------+-----------------------+-----------+
| root             |                                           | auth_socket           | localhost |
| mysql.session    | *THISISNOTAVALIDPASSWORDTHATCANBEUSEDHERE | mysql_native_password | localhost |
| mysql.sys        | *THISISNOTAVALIDPASSWORDTHATCANBEUSEDHERE | mysql_native_password | localhost |
| debian-sys-maint | *CC744277A401A7D25BE1CA89AFF17BF607F876FF | mysql_native_password | localhost |
+------------------+-------------------------------------------+-----------------------+-----------+
4 rows in set (0.00 sec)
```

To configure the `root` account to authenticate with a password, run the following `ALTER USER` command. Be sure to change password to a strong password of your choosing and then run `FLUSH PRIVILEGES` which tells the server to reload the grant tables and put your new changes into effect:

```sql
mysql> ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'password';
mysql> FLUSH PRIVILEGES;
```

## Verification

Now when you check which authentication method each of your MySQL user accounts uses the output should be something like this:

```sql
mysql> SELECT user,authentication_string,plugin,host FROM mysql.user;
```

```shell
+------------------+-------------------------------------------+-----------------------+-----------+
| user             | authentication_string                     | plugin                | host      |
+------------------+-------------------------------------------+-----------------------+-----------+
| root             | *3636DACC8616D997782ADD0839F92C1571D6D78F | mysql_native_password | localhost |
| mysql.session    | *THISISNOTAVALIDPASSWORDTHATCANBEUSEDHERE | mysql_native_password | localhost |
| mysql.sys        | *THISISNOTAVALIDPASSWORDTHATCANBEUSEDHERE | mysql_native_password | localhost |
| debian-sys-maint | *CC744277A401A7D25BE1CA89AFF17BF607F876FF | mysql_native_password | localhost |
+------------------+-------------------------------------------+-----------------------+-----------+
4 rows in set (0.00 sec)
```

```shell
mysql> exit
```
