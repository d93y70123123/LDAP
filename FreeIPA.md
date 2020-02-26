# FreeIPA  

# install IPA 
**module list**  
```bash
[root@localhost ~]# yum module list idm
上次中介資料過期檢查：0:33:54 以前，時間點為西元2020年02月26日 (週三) 20時36分52秒。
CentOS-8 - AppStream
Name       Stream          Profiles                                      Summary
idm        DL1             common [d], adtrust, client, dns, server      The Red Hat Enterprise Linux Identity Management system module
idm        client [d]      common [d]                                    RHEL IdM long term support client module

提示：預設[d]、已啟用[e]、已停用[x]、已安裝[i]
```

**module enable**  
```bash
[root@localhost ~]# yum module enable idm:DL1
上次中介資料過期檢查：0:35:46 以前，時間點為西元2020年02月26日 (週三) 20時36分52秒。
依賴關係解析完畢。
============================================================================================================================================
 Package                          Architecture                    Version                            Repository                        Size
============================================================================================================================================
Enabling module streams:
 389-ds                                                           1.4
 httpd                                                            2.4
 idm                                                              DL1
 pki-core                                                         10.6
 pki-deps                                                         10.6

處理事項摘要
============================================================================================================================================

這樣可以嗎 [y/N]： y
完成！
```

**全系統更新**  
跟yum update
```bash
[root@localhost ~]# dnf distro-sync
上次中介資料過期檢查：0:49:38 以前，時間點為西元2020年02月26日 (週三) 20時36分52秒。
依賴關係解析完畢。
無事可做。
完成！
```

**安裝IPA**  
```bash
[root@localhost ~]# yum install ipa-server
上次中介資料過期檢查：0:56:32 以前，時間點為西元2020年02月26日 (週三) 20時36分52秒。
依賴關係解析完畢。
============================================================================================================================================
 Package                                  Architecture    Version                                                  Repository          Size
============================================================================================================================================
安裝:
 bind-dyndb-ldap                          x86_64          11.1-14.module_el8.1.0+265+e1e65be4                      AppStream          130 k
 ipa-server                               x86_64          4.8.0-13.module_el8.1.0+265+e1e65be4                     AppStream          509 k
 ipa-server-dns                           noarch          4.8.0-13.module_el8.1.0+265+e1e65be4                     AppStream          181 k
將安裝依賴項目:
...
 maven                                                    3.5
處理事項摘要
============================================================================================================================================
安裝  154 軟體包

總下載大小：108 M
安裝的大小：340 M
這樣可以嗎 [y/N]：
```

**若沒有DNS則將電腦資訊加入hosts**  
```bash
[root@localhost ~]# echo "192.168.19.8 ipa1.example.com" | sudo tee -a /etc/hosts
192.168.19.8 ipa1.example.com

更改hostname
[root@localhost ~]# hostnamectl set-hostname ipa1.example.com --static
```

## IPA 初始化設定  
```diff
[root@ipa1 ~]# ipa-server-install

The log file for this installation can be found in /var/log/ipaserver-install.log
==============================================================================
This program will set up the IPA Server.
Version 4.8.0

This includes:
  * Configure a stand-alone CA (dogtag) for certificate management
  * Configure the NTP client (chronyd)
  * Create and configure an instance of Directory Server
  * Create and configure a Kerberos Key Distribution Center (KDC)
  * Configure Apache (httpd)
  * Configure the KDC to enable PKINIT

To accept the default shown in brackets, press the Enter key.

+ Do you want to configure integrated DNS (BIND)? [no]:

Enter the fully qualified domain name of the computer
on which you're setting up server software. Using the form
<hostname>.<domainname>
Example: master.example.com.


+ Server host name [ipa1.example.com]:

The domain name has been determined based on the host name.

+ Please confirm the domain name [example.com]:

The kerberos protocol requires a Realm name to be defined.
This is typically the domain name converted to uppercase.

+ Please provide a realm name [EXAMPLE.COM]:
Certain directory server operations require an administrative user.
This user is referred to as the Directory Manager and has full access
to the Directory for system management tasks and will be added to the
instance of directory server created for IPA.
The password must be at least 8 characters long.

+ Directory Manager password:
+ Password (confirm):

The IPA server requires an administrative user, named 'admin'.
This user is a regular system account used for IPA server administration.

+ IPA admin password:
+ Password (confirm):

+ Do you want to configure chrony with NTP server or pool address? [no]:

The IPA Master Server will be configured with:
Hostname:       ipa1.example.com
IP address(es): 192.168.19.8
Domain name:    example.com
Realm name:     EXAMPLE.COM

The CA will be configured with:
Subject DN:   CN=Certificate Authority,O=EXAMPLE.COM
Subject base: O=EXAMPLE.COM
Chaining:     self-signed

+ Continue to configure the system with these values? [no]: yes

The following operations may take some minutes to complete.
Please wait until the prompt is returned.
....
```

**安裝完後**
```bash
...
==============================================================================
Setup complete

Next steps:
        1. You must make sure these network ports are open:
                TCP Ports:
                  * 80, 443: HTTP/HTTPS
                  * 389, 636: LDAP/LDAPS
                  * 88, 464: kerberos
                UDP Ports:
                  * 88, 464: kerberos
                  * 123: ntp

        2. You can now obtain a kerberos ticket using the command: 'kinit admin'
           This ticket will allow you to use the IPA tools (e.g., ipa user-add)
           and the web user interface.

Be sure to back up the CA certificates stored in /root/cacert.p12
These files are required to create replicas. The password for these
files is the Directory Manager password
The ipa-server-install command was successful
```


