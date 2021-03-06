# The Key Master - A Pragmatic Approach to SSL, SSH, and YubiKeys

## 1. Set Up
 - macOS: ```OpenSSH_8.4p1, OpenSSL 1.1.1h  22 Sep 2020```
    - installed in macOS 10.15.7 with the command: ```brew update; brew upgrade; brew install openssh```
 - raspberryPi: ```Ubuntu Server 20.04.1 LTS w/ OpenSSH_8.2p1 Ubuntu-4ubuntu0.1, OpenSSL 1.1.1f 31 Mar 2020```
    - installed with Raspberry Pi Imager and selecting Ubuntu Server 20.04.1
 - You need a YubiKey with FIDO2 support. Then, using YubiKey Manager, set the FIDO2 PIN
    - Install [YubiKey Manager](https://developers.yubico.com/yubikey-manager/). On macOS ```brew install ykman```
    - ```ykman fido set-pin```
- You need a YubiKey with PIV support. Using YubiKey Manager Reset the PIV support.
  Default PIN, PUK, and Management Key will be the following:
   - default PIN: 123456
   - default PUK: 12345678
   - default Management key: 010203040506070801020304050607080102030405060708

- Then, using YubiKey PIV Tool, reset the Card Holder Unique Identifier.
   - ```yubico-piv-tool -a set-ccc```
   - ```yubico-piv-tool -a set-chuid```


## 2. Certificate Authority
We start by creating the private/public key pair for the Certficate Authority. For RSA keys, we can use OpenSSL toolset or the OpenSSH toolset, as the Private Keys are interchangable in PEM format.

### 2.1 Create a Certficate Authority using OpenSSL
Use the following commands to generate an RSA private/public key pair using the OpenSSL toolset. It will generate private and public key in PEM format. Then, use ```ssh-keygen``` convert the public key into SSH format.
```
openssl genrsa -out ca_key 2048
chmod 400 ca_key
openssl req -new -x509 -key ca_key -out ca.crt
openssl rsa -in ca_key -RSAPublicKey_out -out ca_key_pem.pub
ssh-keygen -i -m PEM -f ca_key_pem.pub > ca_key.pub
```

### 2.2 (Alternative Deprecated) Create a Certficate Authority using OpenSSH
Use the following commands to generate an RSA private/public key pair using the OpenSSH toolset. It will generate private key in PEM format and public key in native SSH format. Then, use ```ssh-keygen``` convert the SSH public key into interchangeable PEM format.
```
ssh-keygen -t rsa -b 2048 -C "CA" -m PEM -f ca_key
ssh-keygen -e -m PEM -f ca_key.pub > ca_key_pem.pub
```

## 3. Configure SSH for User's Certificates
An SSH sever can be be configured to accept User's Certificate during the login process. Such configuration can be performed at Linux user level or at system level for all Linux users. The following configurations will allow for certificate-based auithentication first, then falling back to password authentication should the first one fail. Please refer to earlier sections in the Security Manual for a  hardened configuration to disable password authentication.

### 3.1 Configure SSH at System Level
To configure the SSH server to allow for user's certificate logins, copy the ```ca_key.pub``` on the path ```/etc/ssh/ca_key.pub```. Then, edit ```/etc/ssh/sshd_config``` and add the following lines at the end:
```
PubkeyAuthentication yes
TrustedUserCAKeys /etc/ssh/ca_key.pub
```

### 3.2 (Alternative) Configure SSH at User Level
Create an SSH ```authorized_keys``` file and indicate that the key should be trusted as a certificate authority to validate proprietary OpenSSH certificates for authenticating as that user. This is achieved by adding the ````cert-authority``` prefix in the ```authorized_keys``` file. In this way, all identities signed by the CA will be allowed to login.

Finally, copy the ```authorized_keys``` file to the target server in the home directory of the target Linux account under ```~/.ssh/authorized_keys``` path.
```
sed 's/^/cert-authority /' ca_key.pub > authorized_keys
scp authorized_keys <login>@<server>:/home/<login>/.ssh/authorized_keys
```


## 4. User's Identity, User's Certificate and SSH Logins
User's identiy can be created on the host computer file system or, using OpenSSH 8.2 or above,it can be an created and stored as a resident key within YubiKey using the FIDO2 standard.

### 4.1 User's Identity, User's Certificate using PIV standard on OpenSSH
The following commands create a private RSA key in the YubiKey 5C in slot 9a.
To create the private key you will be asked to enter the YubiKey Management Key and then to touch the key.
After the first command, the private key will be stored in the YubiKey slot 9a and the public key will be in the file system in PKCS8 format.
The next command will create a Certificate Signing Request (CSR) to have the pubic key signed by the Certificate Authority.
The create the CSR, you will be asked to enter the YubiKey PIN and then to touch the key.
In order to use the User's Certificate to perform a login on ESF prototype Web UI, it is necessary to set the Common Name appropriately. See section 5.2 before continuing with the step below.
```
yubico-piv-tool -a generate -s 9a -k --pin-policy=once --touch-policy=always --algorithm=RSA2048 -o user_key_pkcs8.pub
yubico-piv-tool -a verify-pin -a request-certificate -s 9a -S '/CN=john.doe/OU=everyware_iot/O=eurotech.com/' -i user_key_pkcs8.pub -o user.csr
```

The following commands will sign the user's CSR with CA and we will then import the resulting X.509 Certificate in the YubiKey 9a slot.
```
openssl x509 -req -in user.csr -CA ca.crt -CAkey ca_key -CAcreateserial -out user.crt
yubico-piv-tool -k -a import-certificate -s 9a -i user.crt
yubico-piv-tool -a status
```

Then, the following commands will create an SSH user's certificate by signing the user's public key with the ```ssh-keygen``` tool.
The commands below create a certificate that allows to login as the `pi` user. If you are not using a Raspberry PI you might want to change the value of the `-n` argument in the second line.
```
ssh-keygen -i -m PKCS8 -f user_key_pkcs8.pub > user_key.pub
ssh-keygen -s ca_key -I john.doe -n pi user_key.pub
ssh-keygen -Lf user_key-cert.pub
```

Finally, create a local SSH config file and then start the SSH connection like the following.
The following ssh_config option is supported starting from OpenSSH 7.1p2.
You will be asked to enter the YubiKey PIN and touch the key to proceed.
```
echo "Host *\n CertificateFile user_key-cert.pub\n" > ssh_config
ssh -v -I /usr/local/lib/libykcs11.dylib -F ssh_config  pi@raspberrypi.local
```
It is also possible to specify the certificate to be used without creating the `ssh_config` file:
```
ssh -v -I /usr/local/lib/libykcs11.dylib -i user_key-cert.pub pi@raspberrypi.local
```

The log entry similar to the following will be created the server auth.log.
```
Oct 27 12:53:35 raspberrypi sshd[14878]: Accepted publickey for pi from fe80::c4d:be64:65a6:4e36%eth0 port 50783 ssh2: RSA-CERT SHA256:H7ixGmsL00ca0B4SnMXDJKIqQH1dhPcmSYgRcbSTlus ID john.doe (serial 0) CA RSA SHA256:tPm6k0CATYBYUkvwSMklwIn+meOT/6DzOTr8EXWrBCk
Oct 27 12:53:35 raspberrypi sshd[14878]: pam_unix(sshd:session): session opened for user pi by (uid=0)
Oct 27 12:53:35 raspberrypi systemd-logind[367]: New session c12 of user pi.
```

### 4.2 (Alternative Deprecated) User's Identity, User's Certificate using FIDO2 standard on OpenSSH 8.2
The following commands create a resident private key in the YubiKey 5C and have it signed by the Certificate Authority.
To create the resident private key you will be asked to enter the YubiKey FIDO2 PIN and then to touch the key.
After the commands, the private key will be stored in the YubiKey, which a private key handle and the public key will be in the file system.
```
ssh-keygen -t ecdsa-sk -C "john.doe" -O resident -f "user_ecdsa_resident_sk"
ssh-keygen -s ca_key -I john.doe -n ubuntu user_ecdsa_resident_sk.pub
```

Finally, login via SSH to the remote server using the resident key.
You will be asked to confirm user's presence by touching the key during the login process.
```
ssh -v -i user_ecdsa_resident_sk <login>@<server>
```

The log entry similar to the following will be created the server auth.log.
```
Oct 20 10:38:14 ubuntu sshd[17633]: Postponed publickey for ubuntu from 192.168.1.158 port 63803 ssh2 [preauth]
Oct 20 10:38:16 ubuntu sshd[17633]: Accepted publickey for ubuntu from 192.168.1.158 port 63803 ssh2: ECDSA-SK-CERT SHA256:o0uxCRlZwoAdIZMykjJYdDW6U2pGdDAddN00+D3xtSk ID john.doe (serial 0) CA RSA SHA256:H+aAknRDZX6DAk5NCAaQ3d+34aRoZ5Hk6wL3viAH5zo
Oct 20 10:38:16 ubuntu sshd[17633]: pam_unix(sshd:session): session opened for user ubuntu by (uid=0)
Oct 20 10:38:16 ubuntu systemd-logind[1172]: New session 95 of user ubuntu.
```

### 4.3 (Alternative Deprecated) User's Identity, User's Certificate using OpenSSH
Use the following commands to create a user's key and have it signed by the Certificate Authority.
```
ssh-keygen -t rsa -b 4096 -C “john.doe” -m PEM -f user_key
ssh-keygen -s ca_key -I john.doe -n <login> user_key.pub
```

Finally, login via SSH to the remote server.
```
ssh -v -i user_key <login>@<server>
```

The log entry similar to the following will be created the server auth.log.
```
Oct 20 10:31:47 ubuntu sshd[17542]: Accepted publickey for ubuntu from 192.168.1.158 port 63614 ssh2: RSA-CERT SHA256:Fxs8xBUDp7yGthmTAxLVpSlQm+VbXLeeduByz87cjfw ID john.doe (serial 0) CA RSA SHA256:H+aAknRDZX6DAk5NCAaQ3d+34aRoZ5Hk6wL3viAH5zo
Oct 20 10:31:47 ubuntu sshd[17542]: pam_unix(sshd:session): session opened for user ubuntu by (uid=0)
Oct 20 10:31:47 ubuntu systemd-logind[1172]: New session 94 of user ubuntu.
```


## 5. Web Application Logins

### 5.1 Jetty SSL Mutual Anthentication
First, make sure to have configured Jetty for HTTPS access - follow the steps in Jetty
To configure Jetty for SSL Mutual Authentication, create a new Truststore with the CA.
```
keytool -import -alias ca -file ca.crt -storetype JKS -keystore truststore.jks
```

Then configure Jetty ```ssl.ini``` as the following and restart it:
```
jetty.sslContext.trustStorePath=etc/truststore.jks
jetty.sslContext.trustStorePassword=password
jetty.sslContext.trustStoreType=JKS
jetty.sslContext.needClientAuth=true
```

On the Jetty server side, the certificate used by the client to authenticate is available through the following technique:
https://stackoverflow.com/questions/20056304/in-the-jetty-server-how-can-i-obtain-the-client-certificate-used-when-client-aut

### 5.2 ESF Web UI SSL Clinet Authentication Prototype Integration

A prototype that integrates support for SSL client autentication in Kura/ESF Web UI can be obtained by building branches [1] (Kura) and [2] (ESF).

This prototype allows to define multiple Web UI roles, each role is identified by a name and a set of permissions that allows to restrict the set of allowed operations.

At the time of writing the UI sections that allow to create roles and assign permissions is incomplete. A role named "admin" is available out of the box, this role has all permissions.

The prototype allows to configure zero or more X.509 CA certificates used for SSL client side authentication. The prototype contains a working web ui section that allows to manage CAs.

In order to perform a successful login, the client must present a certificate chain that has been signed by one of the configured CAs. The Common Name of the leaf certificate of the chain must be set to the role name (e.g "admin").

In order to add a CA perform the following steps:

1. Access to the web ui using password authentication and navigate to the **Security** -> **Certificates List** section.
2. Press the **Add** button, select **Https Client Certificate** from the list
3. Paste the CA certificate in PEM format in the **Certificate** field (the content of the `ca_key_pem.pub`).

[1] https://github.com/nicolatimeus/kura/tree/feature_web-ext
[2] https://github.com/eurotech/kura_eth/tree/feature_web-ext

### 5.3 Mozilla Firefox
Use the follow instructions to add YKCS11 as a security device to Mozilla Firefox.
https://developers.yubico.com/yubico-piv-tool/YKCS11/Supported_applications/firefox.html

Finally, visit the Jetty website where SSL mutual authentication is configured using FireFox.
You will be prompted to enter the YubiKey PIN and to select an Identify from the YubiKey
Afer selecting the Identity in the 9a slot and touching the YubiKey will be allowed to login.


### 5.4 Safari or Chrome on OSX
Safari and Chrome are using the macOS Keychain to select the SSL Identity.
The macOS can be extended using the Smart Card support in macOS.
To achieve this, we need to pair the YubiKey to macOS.
For Smart Cards to be paired with macOS, the CHUID must be set and the slots 9a and 9c should have a key and a certificate.
To do so, let's create a key in the 9c slot and load its certificate in the YubiKey.
```
yubico-piv-tool -a generate -s 9d -k --pin-policy=once --touch-policy=always --algorithm=RSA2048 -o manager_key_pkcs8.pub
yubico-piv-tool -a verify-pin -a request-certificate -s 9d -S '/CN=manager/OU=everyware_iot/O=eurotech.com/' -i manager_key_pkcs8.pub -o manager.csr
openssl x509 -req -in manager.csr -CA ca.crt -CAkey ca_key -CAcreateserial -out manager.crt
yubico-piv-tool -k -a import-certificate -s 9d -i manager.crt
yubico-piv-tool -a status
```

Set a new CHUID

```
yubico-piv-tool -aset-chuid
```

Then, unplug and replug the YubiKey on a Mac running macOS High Sierra or later.
You will be prompted to pair the Smart Card.
Enter the YubiKey PIN, touch the card, and the macOS account password.
At that point the YubiKey paired and can be used by Safari.

Finally, visit the Jetty website where SSL mutual authentication is configured using Safari.
You will be prompted to enter the YubiKey PIN and to select an Identify from the YubiKey
Afer selecting the Identity in the 9a slot and touching the YubiKey will be allowed to login.


## 6. References

## 6.1 SSH References
 - https://ruimarinho.gitbooks.io/yubikey-handbook/content/ssh/authenticating-ssh-with-piv-and-pkcs11-client/
 - https://ruimarinho.gitbooks.io/yubikey-handbook/content/ssh/authenticating-ssh-via-user-certificates-server/
 - https://www.freebsd.org/cgi/man.cgi?ssh_config(5)
 - https://www.stavros.io/posts/u2f-fido2-with-ssh/
 - https://technotes.seastrom.com/2020/03/27/openssh-82-yubikey.html
 - https://www.yubico.com/authentication-standards/fido2/
 - https://www.openssh.com/releasman ssh_configenotes.html

## 6.2 Jetty References
 - https://www.eclipse.org/jetty/documentation/current/jetty-ssl-distribution.html
 - https://stackoverflow.com/questions/20056304/in-the-jetty-server-how-can-i-obtain-the-client-certificate-used-when-client-aut

## 6.3 macOS Smart Card
 - https://support.yubico.com/hc/en-us/articles/360016649059-Using-Your-YubiKey-as-a-Smart-Card-in-macOS
 - https://support.apple.com/en-us/HT210541
 - https://ludovicrousseau.blogspot.com/2019/10/macos-catalina-and-smart-cards-status.html
 - ```man SmartCardServices```
 - https://ludovicrousseau.blogspot.com/2018/09/smart-card-integration-in-macos-sierra.html
 - https://github.com/OpenSC/OpenSC/issues/1767




