### Notes to create a certificate authority with openssl. Assumes the directories will be created under /root

### Create the directories

mkdir -p /root/mtls/{certs,private}


### Create index.txt and serial files to keep track of signed certificates
cd /root/mtls/
echo 01 > serial
touch index.txt


### Copy the openssl.cnf file from /etc/pki/tls/ and make modifications to it
cp /etc/pki/tls/openssl.cnf .


###  Within openssl.cnf, update the following attributes ***
dir = /root/mtls
new_certs_dir = $dir/certs
certificate = $dir/certs/cacert.pem
countryName_default
stateOrProvinceName_default
localityName_default
0.organizationName_default
organizationalUnitName_default


### Generate the private key for the CA certificate
openssl genrsa -out private/cakey.pem 4096


### Create the CA certificate
openssl req -new -x509 -days 3650 -config /root/mtls/openssl.cnf -key private/cakey.pem -out certs/cacert.pem

### Convert the certificate to PEM format

openssl x509 -in certs/cacert.pem -out certs/cacert.pem -outform PEM

### Create a directory for server certificates
mkdir /root/server_certs
cd /root/server_certs/


### Generate the private key for the server
openssl genrsa -out server.key.pem 4096


### Generating a Certificate Signing Request (CSR) for the Server.
### Remember to revise the Common Name (CN) to match the server's hostname, e.g., "server.yourdomain.com."
openssl req -new -key server.key.pem -out server.csr


### Creating the Server Certificate
openssl ca -config /root/mtls/openssl.cnf -days 1650 -notext -batch -in server.csr -out server.cert.pem


### The certificate information in the database will be updated with this command. 
### You can check the serial number with
cat /root/mtls/index.txt
openssl x509 -in server.cert.pem -noout -serial

### Create the directory for client certificates.
mkdir /root/client_certs
cd /root/client_certs/


### Generate the private key for the client.
openssl genrsa -out client.key.pem 4096


### Generating a Certificate Signing Request (CSR) for the Client
### Remember to revise the Common Name (CN) to match the client's hostname, e.g., "client.yourdomain.com."
openssl req -new -key client.key.pem -out client.csr


### Creating the Client Certificate
openssl ca -config /root/mtls/openssl.cnf -days 1650 -notext -batch -in client.csr -out client.cert.pem


### The certificate information in the database will be updated with this command. 
### You can check the serial number with
cat /root/mtls/index.txt
openssl x509 -in client.cert.pem -noout -serial

### Validating with openssl

openssl s_server -accept 3000 -CAfile /root/mtls/certs/cacert.pem -cert /root/server_certs/server.cert.pem -key /root/server_certs/server.key.pem -state

openssl s_client -connect 127.0.0.1:3000 -key /root/client_certs/client.key.pem -cert /root/client_certs/client.cert.pem -CAfile /root/mtls/certs/cacert.pem -stat

### curl as a test client

curl -vvv  --proxy-cacert cacert.pem -k --proxy "https://proxy.cloudsecurity.ninja:3128" "https://www.google.com"

curl -vvv --proxy-cert app2.cert.pem --proxy-key app2.key.pem --proxy-cacert cacert.pem -k --proxy "https://proxy.cloudsecurity.ninja:3128" "https://www.f5.com"
