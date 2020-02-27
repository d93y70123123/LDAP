# LDAP replication  
軟體使用的是:openldap  

## 首先先開啟ldap的log  
**兩台都要  
```bash
[root@idserver ldap]# vim /etc/rsyslog.conf
[root@idserver ldap]# systemctl restart rsyslog.service
[root@idserver ldap]# systemctl status rsyslog.service
● rsyslog.service - System Logging Service
   Loaded: loaded (/usr/lib/systemd/system/rsyslog.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2020-02-27 20:46:47 CST; 7s ago
     Docs: man:rsyslogd(8)
           http://www.rsyslog.com/doc/
 Main PID: 4877 (rsyslogd)
    Tasks: 3 (limit: 26213)
   Memory: 3.0M
   CGroup: /system.slice/rsyslog.service
           └─4877 /usr/sbin/rsyslogd -n

 2月 27 20:46:47 idserver systemd[1]: Starting System Logging Service...
 2月 27 20:46:47 idserver rsyslogd[4877]: environment variable TZ is not set, auto correcting this to TZ=/etc/localtime>
 2月 27 20:46:47 idserver systemd[1]: Started System Logging Service.
 2月 27 20:46:47 idserver rsyslogd[4877]:  [origin software="rsyslogd" swVersion="8.37.0-13.el8" x-pid="4877" x-info="h>
[root@idserver ldap]#
```

## 匯入模組、設定
接下來要新增兩個ldif檔  
1. 新增同步的模組  
```bash
[root@idserver ldap]# vim syncprov_mod.ldif

dn: cn=module,cn=config
objectClass: olcModuleList
cn: module
olcModulePath: /usr/lib64/openldap
olcModuleLoad: syncprov.la

[root@localhost ldap]# ldapadd -Y EXTERNAL -H ldapi:/// -f syncprov_mod.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "cn=module,cn=config"
```

2. 將模組加入並設定  
```bash
[root@idserver ldap]# vim syncprov.ldif

dn: olcOverlay=syncprov,olcDatabase={2}[輸入自己使用的資料庫],cn=config
objectClass: olcOverlayConfig
objectClass: olcSyncProvConfig
olcOverlay: syncprov
olcSpSessionLog: 100

[root@localhost ldap]# ldapadd -Y EXTERNAL -H ldapi:/// -f syncprov.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "olcOverlay=syncprov,olcDatabase={2}bdb,cn=config"
```

3. 設定replication
一樣透過ldif更改  
ServerID跟rid必須是獨立不重複的  
* 記得把 # 跟 + 刪除  

```diff
# create new
dn: cn=config
changetype: modify
replace: olcServerID
# specify uniq ID number on each server
+ olcServerID: 1

dn: olcDatabase={2}hdb,cn=config
changetype: modify
add: olcSyncRepl
+ olcSyncRepl: rid=001
+ provider=ldap://192.168.1.1:389/  # [輸入另外一台ip]
  bindmethod=simple
+ binddn="[用來管理的帳號，通常是rootdn]"
+ credentials=[你的密碼]
+ searchbase="dc=dic,dc=ksu"  # 你得baseDN
  scope=sub
  schemachecking=on
  type=refreshAndPersist
  retry="30 5 300 3"
  interval=00:00:01:00
-
add: olcMirrorMode
olcMirrorMode: TRUE
```

**匯入檔案**  
```bash
[root@localhost ldap]# ldapmodify -Y EXTERNAL -H ldapi:/// -f replication.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "cn=config"

modifying entry "olcDatabase={2}bdb,cn=config"

```

## 測試

1. 新增使用者ldif  
```bash
[root@ha1 ldap]# vim user.ldif

#kai Information
dn: cn=kai,ou=login,dc=dic,dc=ksu
cn: kai
sn: kai
uid: kai
objectClass: inetOrgPerson
objectclass: person
objectclass: organizationalPerson
objectclass: top
objectclass: posixAccount
objectclass: shadowAccount
userpassword: {SSHA}4Z+obCEcutHmqdFiG3b2ycf4s/cx
shadowLastChange: 15854
shadowMax: 99999
shadowWarning: 7
uidNumber: 3300
gidNumber: 2002
title: student
loginShell: /bin/bash
homeDirectory: /common/kai/Linux
displayName: 1
```

2. 新增使用者  
```bash
[root@localhost ldap]# ldapadd -x -D "[rootDN]" -w [密碼] -f user.ldif
adding new entry "cn=kaii,ou=login,dc=dic,dc=ksu"
```

3. 在兩台機器搜尋剛剛新增的使用者kaii  

**第一台機器**  
```bash
[root@localhost ldap]# hostname
localhost.localdomain
[root@localhost ldap]# ldapsearch -x -H ldap://127.0.0.1 -b "dc=dic,dc=ksu" uid=kaii
# extended LDIF
#
# LDAPv3
# base <dc=dic,dc=ksu> with scope subtree
# filter: uid=kaii
# requesting: ALL
#

# kaii, login, dic.ksu
dn: cn=kaii,ou=login,dc=dic,dc=ksu
cn: kaii
sn: kaii
uid: kaii
objectClass: inetOrgPerson
objectClass: person
objectClass: organizationalPerson
objectClass: top
objectClass: posixAccount
objectClass: shadowAccount
shadowMax: 99999
shadowWarning: 7
uidNumber: 3300
gidNumber: 2002
title: student
loginShell: /bin/bash
homeDirectory: /common/kaii/Linux
displayName: 1

# search result
search: 2
result: 0 Success

# numResponses: 2
# numEntries: 1
```

**第二台機器**  
```bash
[root@idserver ldap]# hostname
kai-replication
[root@idserver ldap]# ldapsearch -x -H ldap://127.0.0.1 -b "dc=dic,dc=ksu" uid=kaii
# extended LDIF
#
# LDAPv3
# base <dc=dic,dc=ksu> with scope subtree
# filter: uid=kaii
# requesting: ALL
#

# kaii, login, dic.ksu
dn: cn=kaii,ou=login,dc=dic,dc=ksu
cn: kaii
sn: kaii
uid: kaii
objectClass: inetOrgPerson
objectClass: person
objectClass: organizationalPerson
objectClass: top
objectClass: posixAccount
objectClass: shadowAccount
userPassword:: e1NTSEF9NForb2JDRWN1dEhtcWRGaUczYjJ5Y2Y0cy9jeA==
shadowLastChange: 15854
shadowMax: 99999
shadowWarning: 7
uidNumber: 3300
gidNumber: 2002
title: student
loginShell: /bin/bash
homeDirectory: /common/kaii/Linux
displayName: 1

# search result
search: 2
result: 0 Success

# numResponses: 2
# numEntries: 1
```

