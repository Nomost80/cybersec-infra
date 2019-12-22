https://docs.iredmail.org/install.iredmail.on.rhel.html
https://docs.iredmail.org/active.directory.html

- Roundcube webmail: https://mail.corp.alphapar.fr/mail/
* - SOGo groupware: https://mail.corp.alphapar.fr/SOGo/
* - netdata (monitor): https://mail.corp.alphapar.fr/netdata/
*
* - Web admin panel (iRedAdmin): https://mail.corp.alphapar.fr/iredadmin/

* - Username: postmaster@corp.alphapar.fr
* - Password: j5pQk78i

Dec 18 00:20:36 mail roundcube: <rv4nma1j> IMAP Error: Login failed for gsoufflet@alphapar.fr against 127.0.0.1 from 10.0.10.11. AUTHENTICATE LOGIN: A0001 NO [UNAVAILABLE] Temporary authentication failure. [mail.corp.alphapar.fr:2019-12-17 23:20:36] in /opt/www/roundcubemail-1.4.1/program/lib/Roundcube/rcube_imap.php on line 200 (POST /mail/?_task=login&_action=login)

ldapsearch -H ldap://DC.corp.alphapar.fr -x -W -D "vault@corp.alphapar.fr"  -b "ou=Users,ou=IT,ou=alphapar,dc=corp,dc=alphapar,dc=fr"

ldapsearch -H ldap://DC.corp.alphapar.fr -x -W -D "vault@corp.alphapar.fr"  -b "ou=groups,ou=alphapar,dc=corp,dc=alphapar,dc=fr" -s sub "(objectclass=group)"

postmap -q fbobin@alphapar.fr ldap:/etc/postfix/ad_virtual_mailbox_maps.cf

postmap -q fbobin@alphapar.fr ldap:/etc/postfix/ad_sender_login_maps.cf

postmap -q it@corp.alphapar.fr ldap:/etc/postfix/ad_virtual_group_maps.cf

telnet localhost 143
a login "gsoufflet" "Nis@NGtR35"

filter'  => '(&(objectclass=group)(accountStatus=active)(enabledService=display)',
