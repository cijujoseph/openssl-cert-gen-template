# openssl-cert-gen-template

Clone this repo and follow the instructions for the following:

* Create a root CA pair
* Intermediate CA pair
* Generate & sign client & server certs
* Import server certs in JKS format 
* Import CAs into a truststore

This repo structure is created based on [this documentation](https://jamielinux.com/docs/openssl-certificate-authority/index.html). For detailed explanation on the following steps, please refer the above mentioned docs!  

#### Create root CA pair
> When prompted for CN, use "Alfresco Software Inc Root CA" as CN. Root key will be generated at private/ca.key.pem. Root certificate generated can be found at certs/ca.cert.pem
> You may also want to provide a valid directory in the openssl.cnf for "dir"

```
cd openssl-cert-gen-template
openssl genrsa -aes256 -out private/ca.key.pem 4096
openssl req -config openssl.cnf \
      -key private/ca.key.pem \
      -new -x509 -days 7300 -sha256 -extensions v3_ca \
      -out certs/ca.cert.pem
openssl x509 -noout -text -in certs/ca.cert.pem
```

#### Create intermediate CA pair

> When prompted for CN, use "Alfresco Software Inc Intermediate CA" as CN. Intermediate CA key will be generated at intermediate/private/intermediate.key.pem. Intermediate certificate can be found at intermediate/certs/intermediate.cert.pem
> You may also want to provide a valid directory in the openssl.cnf for "dir"

```
cd intermediate
openssl genrsa -aes256 \
      -out private/intermediate.key.pem 4096
openssl req -config openssl.cnf -new -sha256 \
      -key private/intermediate.key.pem \
      -out csr/intermediate.csr.pem
      
openssl ca -config ../openssl.cnf -extensions v3_intermediate_ca \
      -days 3650 -notext -md sha256 \
      -in csr/intermediate.csr.pem \
      -out certs/intermediate.cert.pem

openssl x509 -noout -text \
      -in certs/intermediate.cert.pem
      
openssl verify -CAfile ../certs/ca.cert.pem \
      certs/intermediate.cert.pem
      
cat certs/intermediate.cert.pem \
      ../certs/ca.cert.pem > certs/ca-chain.cert.pem
```

#### Create & sign a client certificate

> When prompted for CN, enter CN as admin@app.activiti.com (Alfresco Process Services default userid)

```
cd intermediate
openssl genrsa -aes256 \
      -out private/admin.key.pem 2048
      
openssl req -config openssl.cnf \
      -key private/admin.key.pem \
      -new -sha256 -out csr/admin.csr.pem
      
openssl ca -config openssl.cnf \
      -extensions usr_cert -days 375 -notext -md sha256 \
      -in csr/admin.csr.pem \
      -out certs/admin.cert.pem
openssl x509 -noout -text \
      -in certs/admin.cert.pem
      
openssl verify -CAfile certs/ca-chain.cert.pem \
      certs/admin.cert.pem
```

#### Create & sign a server certificate

> When prompted for CN, enter CN as localhost (for localhost apps)

```
cd intermediate
openssl genrsa -aes256 \
      -out private/localhost.key.pem 2048
      
openssl req -config openssl.cnf \
      -key private/localhost.key.pem \
      -new -sha256 -out csr/localhost.csr.pem
      
openssl ca -config openssl.cnf \
      -extensions server_cert -days 375 -notext -md sha256 \
      -in csr/localhost.csr.pem \
      -out certs/localhost.cert.pem
openssl x509 -noout -text \
      -in certs/localhost.cert.pem
      
openssl verify -CAfile certs/ca-chain.cert.pem \
      certs/localhost.cert.pem
```

#### Create a keystore for app server configuration

```
openssl pkcs12 -export -chain \
	-in intermediate/certs/localhost.cert.pem \
	-inkey intermediate/private/localhost.key.pem \
	-out keystore/keystore.p12 \
	-CAfile intermediate/certs/ca-chain.cert.pem
	
keytool -importkeystore \
	-destkeystore keystore/keystorex.jks \
	-srckeystore keystore/keystore.p12 \
	-srcstoretype PKCS12
```
#### Create a truststore with both root and intermediate CAs

```
keytool -import -alias rootca \
	-file certs/ca.cert.pem \
	-keystore truststore/truststore.jks
	
keytool -import -alias intermediateca \
	-file intermediate/certs/intermediate.cert.pem \
	-keystore truststore/truststore.jks
```


