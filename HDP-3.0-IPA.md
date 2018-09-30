
## Instuctions for IPA Lab 

### Pre-reqs
- HDP 3.x / Ambari 2.7.x cluster<br>
- Access to an IPA server that has been setup as descibed in [Hortonworks documentation](https://docs.hortonworks.com/HDPDocuments/HDP3/HDP-3.0.1/authentication-with-kerberos/content/kerberos_optional_use_an_existing_ipa.html)

**Lab Topics**<br>

1. [Register cluster nodes as IPA Clients](#section-1)
2. [Secure Ambari via ambari-server setup-security](#section-2)
3. [Enable Kerberos for cluster services](#section-3)
4. [Update Ranger Policies for Groups/Users](#section-4)
5. [Enable LDAP for ambari, knox](#section-5)

-

<a name="section-1"></a>
## 1. Register cluster nodes as IPA clients
- Run below on *all nodes of HDP cluster* (replace $INTERNAL_IP_OF_IPA)
```
echo "$INTERNAL_IP_OF_IPA ipa.hortonworks.com ipa" >> /etc/hosts
```

- Install yum packages
```
sudo yum install -y ipa-client
```

- Update /etc/resolve.conf (replace INTERNAL_IP_OF_IPA)
```
mv /etc/resolv.conf /etc/resolv.conf.bak 
echo "search hortonworks.com" > /etc/resolv.conf
echo "nameserver $INTERNAL_IP_OF_IPA" >> /etc/resolv.conf
```
- Install IPA client

  ```
	service dbus restart
	
	sudo ipa-client-install \
	--server=ipa.hortonworks.com \
	--realm=HORTONWORKS.COM \
	--domain=hortonworks.com \
	--mkhomedir \
	--principal=admin -w BadPass#1 \
	--unattended
  ```

- Make sure you don't see below message from the output of previous command
```
Missing A/AAAA record(s) for host xxxxxxxxx
```

- To uninstall in case of issues:
```
sudo ipa-client-install --uninstall
```

- Note by changing the DNS, its possible the node may not be able to connect to public internet. When you need to do so (e.g. for yum install, you can temporarily revert back the /etc/resolv.conf.bak)


### Verify

- By registering as a client of the IPA server, SSSD is automatically setup. So now the host recognizes users defined in IPA
```
id hadoopadmin
```

- You can also authenticate and get a kerberos ticket (password is BadPass#1)
```
kinit -V hadoopadmin
```

-


<a name="section-2"></a>
# 2. Secure Ambari via ambari-server setup-security

Lets use FreeIPA Generated certificate for Options 1 and 4 in `ambari-server setup-security`
	
  ```
Security setup options...
===========================================================================
Choose one of the following options:
  *[1] Enable HTTPS for Ambari server.
  *[2] Encrypt passwords stored in ambari.properties file.
  [3] Setup Ambari kerberos JAAS configuration.
  *[4] Setup truststore.
  [5] Import certificate to truststore.
===========================================================================
  ```

**Preparation:** Create certificates on all ipa-client hosts (run this on each node)

Ensure SELinux is not enforcing, else requesting a certificate as the root user with admin's kerberos ticket will be denied by the system and certificate will not be created. 

```
getenforce
# If result is "Enforcing", run the following
sudo su
setenforce 0
```

Obtain kerberos ticket as **admin**(or an IPA Privileged User), and request a x509 certificate pair saved as "host.key" and "host.crt" on each host. 

```
echo BadPass#1 | kinit admin 
mkdir /etc/security/certificates/
cd /etc/security/certificates/
ipa-getcert request -v -f /etc/security/certificates/host.crt -k /etc/security/certificates/host.key
```

List the directory to verify certificates are created. 

```
[root@demo certificates]# ls -ltr /etc/security/certificates/
total 8
-rw------- 1 root root 1704 Sep 30 04:56 host.key
-rw------- 1 root root 1724 Sep 30 04:56 host.crt
```


### 2.1 Enable HTTPS for Ambari server
If you are running knox on this host (which is highly not recommended) changing the default port from 8443 will avoid the port conflict. 

```
Security setup options...
===========================================================================
Choose one of the following options:
  [1] Enable HTTPS for Ambari server.
  [2] Encrypt passwords stored in ambari.properties file.
  [3] Setup Ambari kerberos JAAS configuration.
  [4] Setup truststore.
  [5] Import certificate to truststore.
===========================================================================

# Enable SSL
Enter choice, (1-5): 1
Do you want to configure HTTPS [y/n] (y)? y
SSL port [8443] ? 8444
Enter path to Certificate: /etc/security/certificates/host.crt
Enter path to Private Key: /etc/security/certificates/host.key
Please enter password for Private Key: changeit
```

### Verify
Restart ambari-server. Curl ambari on the new https port **without** specifying the "-k" flag.
```
[root@demo ~]$ curl -u admin:"password" https://`hostname -f`:8444/api/v1/clusters
```

### 2.2 Encrypt passwords stored in ambari.properties file.
This step is required for the kerberos wizard to persist the KDC credentials (`hadoopadmin`). It is also required for persisting the `ldapbind` password, without which, enabling ldaps in Ambari 2.7.1 seems to have some challenges.  

```
[root@demo ~]# ambari-server setup-security
Using python  /usr/bin/python
Security setup options...
===========================================================================
Choose one of the following options:
  [1] Enable HTTPS for Ambari server.
  [2] Encrypt passwords stored in ambari.properties file.
  [3] Setup Ambari kerberos JAAS configuration.
  [4] Setup truststore.
  [5] Import certificate to truststore.
===========================================================================
Enter choice, (1-5): 2
Please provide master key for locking the credential store:
Re-enter master key:
Do you want to persist master key. If you choose not to persist, you need to provide the Master Key while starting the ambari server as an env variable named AMBARI_SECURITY_MASTER_KEY or the start will prompt for the master key. Persist [y/n] (y)? y
Adjusting ambari-server permissions and ownership...
Ambari Server 'setup-security' completed successfully.
```


### 2.3 Setup truststore.

Setting up the truststore ahead of time and restarting Ambari seems to make the ldap integration happier. 
Ambari can leverage the `/etc/pki/java/cacerts` truststore managed by IPA Clients on the hosts. This truststore contains the public CAs, along with the IPA CA, which should be the only certificates needed.    

```
# Example for ipa hostname: ipa.hortonworks.com

[root@demo ~]# /usr/java/default/bin/keytool -list \
-keystore /etc/pki/java/cacerts \
-v -storepass changeit | grep ipa

Alias name: hortonworks.comipaca
   accessLocation: URIName: http://ipa-ca.hortonworks.com/ca/ocsp
```


```
[root@demo certificates]# ambari-server setup-security
Using python  /usr/bin/python
Security setup options...
===========================================================================
Choose one of the following options:
  [1] Enable HTTPS for Ambari server.
  [2] Encrypt passwords stored in ambari.properties file.
  [3] Setup Ambari kerberos JAAS configuration.
  [4] Setup truststore.
  [5] Import certificate to truststore.
===========================================================================
Enter choice, (1-5): 4
Do you want to configure a truststore [y/n] (y)? y
TrustStore type [jks/jceks/pkcs12] (jks):
Path to TrustStore file :/etc/pki/java/cacerts
Password for TrustStore: changeit
Re-enter password: changeit
Ambari Server 'setup-security' completed successfully.
```


### 2.4 Restart ambari for changes to take effect

```
ambari-server restart
```

<br> 


<a name="section-3"></a>
# 3. Enable kerberos on the cluster

-Start Ambari 2.7.x security wizard and select IPA option and pass in below:
![Image](https://raw.githubusercontent.com/HortonworksUniversity/Security_Labs/master/screenshots/IPA-SecurityWizard.png)
  

<a name="section-5"></a>
# 5. Enable LDAP For Ambari, Knox

FreeIPA Tips For LDAP Search Properties


