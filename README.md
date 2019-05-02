# Cisco-CSPC-Collector

## User Accounts
There are 3 user accounts as far as RHEL is concerned.

+ admin
+ collectorlogin
+ root

Admin is used to log into the Web UI. It can also be used to SSH to the appliance.

collectorlogin is used for administering the appliance itself.

root is prevented from SSH'ing or accessing anything directly, but can be used for granting sudo access.

Root and collectorlogin are set to expire every 90 days. Reset the passwords using the admin account as seen below.
```
===========================================================================
                Cisco Network Appliance Administration
===========================================================================
To see the list of all the commands press '?'
 
admin# pwdreset collectorlogin 90
Password for 'collectorlogin' reset to - <removed> successfully
Password expires in 90 days
Shell is enabled
passwd: all authentication tokens updated successfully
*** Please memorize the new password ***
Lost passwords cannot be recovered. The only alternative to recover is to reinstall the server.
 
 
admin# pwdreset root 90
Password for 'root' reset to - <removed> successfully
Password expires in 90 days
Shell is enabled
passwd: all authentication tokens updated successfully
*** Please memorize the new password ***
Lost passwords cannot be recovered. The only alternative to recover is to reinstall the server.
```

## Web UI SSL Certificate
The main web UI is accessible at https://<collectorname:8001/. By default, Tomcat uses a self-signed certificate. Let's fix that with a certificate signed by our local certificate authority. This guide assumes a Microsoft CA.

### Obtaining the certificate
+ Using your local computer, generate an encrypted 2048 bit RSA key with OpenSSL. Set the passphrase as 1234 for ease of use.
+ Using your local computer, generate a CSR (certificate signing request) using that same key
+ Submit the CSR to the CA (certificate authority). Add in these SANs: san:dns=<collectorname>.domain.tld&dns=<collectorname>
+ Download the certificate as base-64
+ Open the cert and key in a text editor
  
### Loading the certificate on the collector
+ SSH to the collector on port 22 as the `collectorlogin` account
+ Create a new file for the certificate and paste the data in
```
nano <collectorname>.cer
```
+ Create a new file for the private key and paste the data in
```
nano <collectorname>.pem
```
+ Combine those files into a PKCS12 file. You'll need that same passphrase you created for the RSA key (1234)
```
openssl pkcs12 -export -in <collectorname>.cer -inkey <collectorname>.pem > <collectorname>.p12
```
+ Add the PKCS12 file to the existing JKS (Java keystore) used by Tomcat. Use this, and only this as the JKS passphrase: `cspcgxt`
```
/opt/cisco/ss/adminshell/applications/CSPC/jreinstall/bin/keytool -importkeystore -srckeystore <collectorname>.p12 -srcstoretype pkcs12 -destkeystore $CSPCHOME/webui/tomcat/conf/cspcgxt -deststoretype jks
```
+ List the certs in the keystore to ensure it imported successfully. There will be 2 entries, or aliases, "tomcat" and "1"
```
/opt/cisco/ss/adminshell/applications/CSPC/jreinstall/bin/keytool -list -v -keystore $CSPCHOME/webui/tomcat/conf/cspcgxt
```
+ Delete the existing self-signed cert by removing the "tomcat" alias
```
/opt/cisco/ss/adminshell/applications/CSPC/jreinstall/bin/keytool -delete -alias tomcat -keystore $CSPCHOME/webui/tomcat/conf/cspcgxt
```
+ Change the "1" alias to tomcat so that the server will pick it up and use it
```
/opt/cisco/ss/adminshell/applications/CSPC/jreinstall/bin/keytool -changealias -alias 1 -destalias tomcat -keystore $CSPCHOME/webui/tomcat/conf/cspcgxt
```
+ Again, list the entries in the keystore and ensure "tomcat" is present and is the newly-minted cert.
```
/opt/cisco/ss/adminshell/applications/CSPC/jreinstall/bin/keytool -list -v -keystore $CSPCHOME/webui/tomcat/conf/cspcgxt
```
+ Restart the collector
```
service cspc restart
```
