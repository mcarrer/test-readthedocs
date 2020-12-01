

## Setup

### EJBCA Setup
```
cd dev/ejbca
docker run -it --rm -p 80:8080 -p 443:8443 -h localhost \
    -v /Users/mcarrer/dev/ejbca/persistent/:/mnt/persistent/ \
    -e TLS_SETUP_ENABLED="simple" \
    -e "DATABASE_JDBC_URL=jdbc:h2:/mnt/persistent/ejbcadb;DB_CLOSE_DELAY=-1" \
    --name ejbca \
    primekey/ejbca-ce
```

```
https://localhost/ejbca/
https://localhost/ejbca/ra/
https://localhost/ejbca/adminweb/
```

```
docker exec -it `docker ps -q -f "name=ejbca"` /bin/bash
docker stop `docker ps -q -f "name=ejbca"`
```

```
docker exec -it `docker ps -q -f "name=ejbca"` /bin/bash
cd ejbca
openssl rand -base64 16
/opt/primekey/ejbca/bin/ejbca.sh ra addendentity --eeprofile "DEVICE" --username "01020304" --password "0sqMZ1vL15yquku7L7J4QQ==" --dn="C=IT,ST=Udine,L=Amaro,O=Eurotech SpA,OU=Everyware IoT,CN=F8369BE8BD2B,serialNumber=Y119E9A0030" --caname=DeviceCA --type=1 --token PEM -u ejbca --clipassword=ejbca
```

```
cd ../jscep-cli-jdk6
openssl genrsa -out device_01020304_key
openssl req -new -sha256 \
    -key device_01020304_key \
    -subj "/C=IT/ST=Udine/L=Amaro/O=Eurotech SpA/OU=Everyware IoT/CN=F8369BE8BD2B/serialNumber=Y119E9A0030" \
    -out device_01020304.csr -outform PEM

openssl s_client -showcerts -connect localhost:443 </dev/null 2>/dev/null | openssl x509 -outform PEM > ejbcacert.pem
openssl x509 -in ejbcacert.pem -noout -text

keytool -importcert -v -alias server-alias -file ejbcacert.pem -keystore cacerts.jks -keypass changeit -storepass changeit -storetype JKS

java -Djavax.net.ssl.trustStore=/Users/mcarrer/dev/ejbca/device/cacerts.jks \
    -jar $JSCEP_LIB/jscep-cli-1.3-SNAPSHOT-jar-with-dependencies.jar \
    --ca-identifier DeviceCA \
    --dn "C=IT,ST=Udine,L=Amaro,O=Eurotech SpA,OU=Everyware IoT,CN=F8369BE8BD2B,serialNumber=Y119E9A0030" \
    --challenge "0sqMZ1vL15yquku7L7J4QQ==" \
    --key-file device_01020304_key \
    --csr-file device_01020304.csr \
    --url https://localhost:443/ejbca/publicweb/apply/scep/pkiclient.exe \
    --url https://localhost:443/ejbca/publicweb/apply/scep/pkiclient.exe \
    --verbose

```

### JSCEP CLI Setup
```
git clone git://github.com/asyd/jscep-cli-jdk6.git
cd jscep-cli-jdk6
mvn package
```

## References

### EJCA References
- https://doc.primekey.com/ejbca/ejbca-operations/ejbca-ca-concept-guide/protocols/scep
- https://doc.primekey.com/ejbca/ejbca-operations/ejbca-ca-concept-guide/end-entities-overview/subject-distinguished-names


### SCEP References
- https://www.cisco.com/c/en/us/support/docs/security-vpn/public-key-infrastructure-pki/116167-technote-scep-00.html
- https://doc.primekey.com/ejbca6152/ejbca-operations/ejbca-concept-guide/protocols/scep#SCEP-jSCEPjSCEP
- https://github.com/jscep/jscep
- https://github.com/asyd/jscep-cli-jdk6

