# Set Up Certificates to use as an OCSP responder

To generate these certificates, I used [this
link](https://medium.com/@bhashineen/create-your-own-ocsp-server-ffb212df8e63).
To generate the ca certificate, I changed the ca.cnf file as such -
`x509_extensions = v3_CA`. To set up the server certificate, I used `x509_extensions = v3_OCSP_server` and for the client certificate I used `x509_extensions = v3_OCSP_client`.

The specific commands used to generate the certificates are here -

## Generating CA cert

Generate the private key

```openssl genrsa -out ca_ocsp.key 1024```

Generate the CA cert

```openssl req -new -x509 -days 3650 -key ca_ocsp.key -out ca_ocsp.crt -config openssl.cnf```

## Generating the Server cert

Generate the server private key

```openssl genrsa -out server_ocsp.key 1024```

Change the `x509_extensions` field in openssl.cnf as listed above. Then run below to generate a certificate.

```openssl req -new -x509 -days 3650 -key server_ocsp.key -out openssl.crt -config server_ocsp.cnf```

Then run the next command to generate a CSR based on the cert.

```openssl x509 -x509toreq -in server_ocsp.crt -out server_ocsp.csr -signkey server_ocsp.key```

Finally run this last command to have the CA sign the certificate.

```openssl ca -days 600 -keyfile ca_ocsp.key -cert ca_ocsp.crt -policy policy_anything -config openssl.cnf  -notext -out server_ocsp.crt -infiles server_ocsp.csr```

Then join the `server_ocsp.pem` and `server_ocsp.key` file together and clean up the repository. Repeat the process (changing the `x509_extensions` field again) for generating the client certificate.

## Setting up the OCSP responder

To set up an OCSP responder using these certificates, run

```openssl ocsp -index ca_state/index.txt -port 8100 -rsigner ca_ocsp.crt -rkey ca_ocsp.key -CA ca_ocsp.crt -text -out log.txt -ndays 7 &```

To see stapling / mongo shell client OCSP verification, you will need wireshark / tshark running. For tshark, use 

```tshark -i loopback -w loopback.pcap```

To connect, use the mongodb binaries off of evergreen. You will need the `mongod` and `mongo` shell binary. To run the program, use

```/mongod --tlsCAFile .../ca_ocsp.pem --tlsCertificateKeyFile .../server_ocsp.pem --tlsMode requireTLS```

In a separate terminal window, run

```mongo --tlsCAFile .../ca_ocsp.pem --tlsCertificateKeyFile .../client_ocsp.pem --tls --tlsDisabledProtocols TLS1_3```

The status of the certificate is determined by the `ca_state/index.txt` file. On the far left column, there is a letter that is either V, R, or E
standing for Valid, Revoked, or Expired. More information can be found [here](https://pki-tutorial.readthedocs.io/en/latest/cadb.html). Revoking a
certificate through the command line can be done as such

```openssl ca -keyfile ca_ocsp.key -cert ca_ocsp.crt -revoke server_ocsp.pem -config ca.cnf```
