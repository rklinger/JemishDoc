<a name="phd-krb-setup"></a>
#Securing PHD with MIT Kerberos 5
This section describes how to configure Kerberos authentication for your PHD cluster when using EMC Isilon for your HDFS layer. It is recommended that you install PHD and integrate it with EMC Isilon without any security configuration prior to enabling any security so that you can ensure that all services are installed and running.

Below are all the steps to take (in order) to configure Kerberos authentication in your PHD environment when using EMC Isilon as the HDFS layer. 

* [If using stand-alone KDC](#phd-krb-kdc)
	* [Install KDC](#install-kdc)
	* [Configure Cluster Hosts](#kdc-clients)
	* [Prepare EMC Isilon Environment for stand-alone KDC](#isi-krb-KDC) 
	* [Ambari Enable Security](#ambari-enable-security)

**Note:** Make sure you have configured NTP and there is no time skew amongst any cluster hosts or EMC Isilon nodes.

**Note:** Make sure forward and reverse DNS lookups are configured correctly and all cluster and EMC Isilon hosts are resolvable correctly.


<a name="phd-krb-kdc"></a>
##Securing PHD with stand-alone MIT Kerberos 5 KDC

<a name="install-kdc"></a>
**Install KDC**

This section outlines a simple stand-alone krb5 KDC setup.

These instructions were largely derived from Kerberos: The Definitive Guide by James Garman, O'Reilly, pages 53-62.

1. Install the Kerberos packages (krb5-libs, krb5-workstation, and krb5-server) on the KDC host.
1. Define your REALM in /etc/krb5.conf as shown below:

* Set the kdc and admin_server variables to the resolvable hostname of the KDC host.
* Set the default_domain to your REALM.

In the following example, REALM was changed to VLAN172.FE.GOPIVOTAL.COM and the admin server and KDC host were changed to eng-dca-mdw.vlan172.fe.gopivotal.com:

```
[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log

[libdefaults]
 default_realm = VLAN172.FE.GOPIVOTAL.COM
 dns_lookup_realm = false
 dns_lookup_kdc = false
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true

[realms]
 VLAN172.FE.GOPIVOTAL.COM = {
  kdc = eng-dca-mdw.vlan172.fe.gopivotal.com
  admin_server = eng-dca-mdw.vlan172.fe.gopivotal.com
 }

[domain_realm]
 .vlan172.fe.gopivotal.com = VLAN172.FE.GOPIVOTAL.COM
 vlan172.fe.gopivotal.com = VLAN172.FE.GOPIVOTAL.COM
```

1.  Set up /var/kerberos/krb5kdc/kdc.conf:
 * If you want to use AES-256, uncomment the master_key_type line.
 * If you do not want to use AES-256, remove it from the supported_enctypes line.
 * Add a key_stash_file entry: /var/kerberos/krb5kdc/.k5.REALM.
 * Set the maximum ticket lifetime and renew lifetime to your desired values (24 hours and 7 days are typical).
 * Add the kadmind_port entry: kadmind_port = 749.

**Important:** The stash file lets the KDC server start up for root without a password being entered. The result (**NOT using AES-256**) for the above REALM is:

```
[kdcdefaults]
 kdc_ports = 88
 kdc_tcp_ports = 88

[realms]
 VLAN172.FE.GOPIVOTAL.COM = {
  #master_key_type = des3-hmac-sha1
  acl_file = /var/kerberos/krb5kdc/kadm5.acl
  dict_file = /usr/share/dict/words
  admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
  key_stash_file = /var/kerberos/krb5kdc/.k5.VLAN172.FE.GOPIVOTAL.COM
  max_life = 24h 0m 0s
  max_renewable_life = 7d 0h 0m 0s
  kadmind_port = 749
  supported_enctypes = des3-hmac-sha1:normal arcfour-hmac:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal
 }
```
1. Create the KDC master password by running:

```
kdb5_util create -s
```
**Do NOT forget your password**, as this is the root KDC password.This typically runs quickly, but can take 5-10 minutes if the code has trouble getting the random bytes it needs.

1. Add an administrator account as username/admin@REALM. Run the kadmin.local application from the command line:

```
kadmin.local: addprinc root/admin@VLAN172.FE.GOPIVOTAL.COM
```
Type quit to exit kadmin.local.

**Important:** The KDC does not need to be running to add a principal.


1. Start the KDC by running:

```
/etc/init.d/krb5kdc start
```
You should get an [OK] indication if it started without error. If there are errors, please correct the condition after examining the log file: ``/var/log/krb5kdc.log``

1. Edit ``/var/kerberos/krb5kdc/kadm5.acl`` and change the admin permissions username from * to your admin. You can add other admins with specific permissions if you want. This is a sample ACL file:

```
root/admin@VLAN172.FE.GOPIVOTAL.COM     *
```
1. Use kadmin.local on the KDC to enable the administrator(s) remote access:

```
kadmin.local: ktadd -k /var/kerberos/krb5kdc/kadm5.keytab kadmin/admin kadmin/changepw
```

**Important:** kadmin.local is a KDC host-only version of kadmin that can do things remote kadmin cannot (such as use the -norandkey option in ktadd).

1. Start kadmind:

```
/etc/init.d/kadmin start
```
The KDC configuration is now done and the KDC is ready to use. 

<a name="kdc-clients"></a>
**Prepare KDC Clients (all cluster hosts)**

1. Install krb5-libs , krb5-auth-dialog and krb5-workstation on all cluster hosts, including any client/gateway hosts.

```
yum install krb5-libs krb5-workstation krb5-auth-dialog
```
1. Use scp to copy ``/etc/krb5.conf`` from the KDC host to all cluster hosts.
1. Validate with a simple kinit test as below from any cluster host:

```
kinit root/admin
``` 
You should be able to login as ``root/admin`` from any cluster host. If you have errors, please make sure that your KDC is runnign and reachable on the network from any host. Also make sure to referrence the ``/var/log/krb5kdc.log`` on hte KDC host for any error conditions being reported. 
 
<a name="isi-krb-KDC"></a>
###Prepare EMC Isilon Environment for stand-alone KDC

This section describes the steps you need to take to configure EMC Isilon to use a stand-alone MIT Kerberos 5 KDC. 

1. From the OneFS Web UI, navigate to Access, Authentication Providers and click on get Started on the Kerberos Provider tab as shown below:

	![](images/image23.png)

1. Fill out the form with all the information as it pertains to your environment as shown below:

**Note:** Use all upper case when defning a REALM and lower case when defining domains either in FQDN of KDC hosts or domain mappings as seen in the example below.

![](images/image24.png)

1. Upon successful creation of a Kerberos REALM and provider you should see below on the Kerberos Provider tab:

	![](images/image25.png)	

1. Click on the Kerberos Settings Tab and enable the settings as shown below: 
	
	![](images/image26.png)
	
**Note:** Remember to click Save Changes after you have modified the settings above. 	

1. Using the OneFS CLI on any one of the Isilon nodes, configure your HDFS zone to use kerberos as its authentication mechanism. Replace your Isilon Zone Name in the command below: 

```
isi zone zones modify --hdfs-authentication=kerberos_only <zone_name>
```

**Note:** If you do not recall your HDFS zone name, you can use the command ``isi zone zones list `` to list all your zones and use the `<zone_name>` that is serving up your HDFS layer. 


<a name="phd-krb-AD"></a>
##Securing PHD with Kerberos and Active Driectory

This section describes how to configure Kerberos authentication with Active Directory for your PHD cluster when using EMC Isilon as the HDFS layer. It is recommended that you install PHD and integrate it with EMC Isilon without any security configuration prior to enabling any security so that you can ensure that all services are installed and running successfully.

Below are all the steps to take (in order) to configure Kerberos authentication with Active Directory in your PHD environment when using EMC Isilon as the HDFS layer. 

* [If using stand-alone KDC](#phd-krb-kdc)
	* [Install KDC](#install-kdc)
	* [Configure Cluster Hosts](#kdc-clients)
	* [Prepare EMC Isilon Environment for AD](#isi-krb-AD) 
	* [Ambari Enable Security](#ambari-enable-security)

**Note:** Make sure you have configured NTP and there is no time skew amongst any cluster hosts or EMC Isilon nodes.

**Note:** Make sure forward and reverse DNS lookups are configured correctly and all cluster and EMC Isilon hosts are resolvable correctly.


<a name="isi-krb-AD"></a>
###Prepare EMC Isilon Environment for AD



<a name="ambari-enable-security"></a>
##Ambari Enable Security


1. In the Ambari UI, click on Admin and then Security
1. Click on Enable Security to start the wizard and follow on-screen instructions to enable kerberos Security. 
1. Upon completion of the wizard, if a service fails to start up, review the service log file and correct the error condition. 

<a name="hawq-krb-setup"></a>
#Securing HAWQ and PXF with MIT Kerberos 5
This Section describes how to configure Kerberos authentication for HAWQ and PXF when using EMC Isilon as your HDFS layer. 

**Note:** Make sure to have completed all the steps in [Securing PHD with MIT Kerberos 5](#phd-krb-setup) prior to performing any steps in this section.

<a name="hawq-krb-kdc"></a>
##Securing HAWQ and PXF with stand-alone MIT Kerberos 5 KDC

<a name="hawq-krb-AD"></a>
##Securing HAWQ and PXF with Kerberos and Active Driectory