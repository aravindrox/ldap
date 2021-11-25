sudo yum install *openldap* -y
sudo systemctl start slapd
sudo systemctl enable slapd
sudo echo "dn: olcDatabase={0}config,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SSHA}KMaZ8VVRmcH2+098wP2y6VAMGr9QXoNq" >> ldaprootpasswd.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f ldaprootpasswd.ldif
cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
chown -R ldap:ldap /var/lib/ldap/DB_CONFIG
systemctl restart slapd
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
echo "dn: olcDatabase={1}monitor,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth"
  read by dn.base="cn=service,dc=hadoop,dc=com" read by * none

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=hadoop,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=service,dc=hadoop,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SSHA}KMaZ8VVRmcH2+098wP2y6VAMGr9QXoNq

dn: olcDatabase={2}hdb,cn=config
changetype: modify
add: olcAccess
olcAccess: {0}to attrs=userPassword,shadowLastChange by
  dn="cn=service,dc=hadoop,dc=com" write by anonymous auth by self write by * none
olcAccess: {1}to dn.base="" by * read
olcAccess: {2}to * by dn="cn=service,dc=hadoop,dc=com" write by * read
" >> ldapdomain.ldif
ldapmodify -Y EXTERNAL -H ldapi:/// -f ldapdomain.ldif
echo "dn: dc=hadoop,dc=com
objectClass: top
objectClass: dcObject
objectclass: organization
o: hadoop com
dc: hadoop

dn: cn=service,dc=hadoop,dc=com
objectClass: organizationalRole
cn: service
description: Service Account

dn: ou=service1,dc=hadoop,dc=com
objectClass: organizationalUnit
ou: Account

dn: ou=servicegroup,dc=hadoop,dc=com
objectClass: organizationalUnit
ou: Group

dn: ou=servicegroup1,dc=hadoop,dc=com
objectClass: organizationalUnit
ou: Group" >> baseldapdomain.ldif
ldapadd -x -D cn=service,dc=hadoop,dc=com -w admin -f baseldapdomain.ldif
yum install phpldapadmin -y
sudo sed -i "s/Require local/Require all granted/g" /etc/httpd/conf.d/phpldapadmin.conf
sudo systemctl restart httpd.service
sudo yum install nss-pam-ldapd -y
sudo yum install authconfig* -y
authconfig --disablecache --enableldap --enableldapauth --ldapserver=ldap://$(hostname -i):389 --ldapbasedn="dc=,dc=com" --enablemkhomedir --updateall
