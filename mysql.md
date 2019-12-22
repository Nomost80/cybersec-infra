https://www.tecmint.com/install-mariadb-in-centos-7/

```m
create user 'vault'@'vault.corp.alphapar.fr' identified by 'xxxxxx';
grant all on *.* to 'vault'@'vault.corp.alphapar.fr' with grant option;
flush privileges;

select host, user, password from mysql.user;
```