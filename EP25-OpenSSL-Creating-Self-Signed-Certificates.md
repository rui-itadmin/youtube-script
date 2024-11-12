EP25: OpenSSL – Creating Self-Signed Certificates
===
https://youtu.be/cOG5nO-ZMQE

In this episode:
---
- Creating self-signed certificates (including those with SAN)
- Creating a self-signed wildcard certificate
- Trusting self-signed certificates
- Verifying certificates with Nginx and other tools
- Creating a self-signed CA certificate
- Using a self-signed CA to sign other self-signed certificates



Add some hostname mappings in /etc/hosts.
---
```shell
# example.local
echo "192.168.122.115 www.example.local" | sudo tee -a /etc/hosts
echo "192.168.122.115 docs.example.local" | sudo tee -a /etc/hosts

# example.corp
echo "192.168.122.115 www.example.corp" | sudo tee -a /etc/hosts
echo "192.168.122.115 docs.example.corp" | sudo tee -a /etc/hosts
```

Generate a basic self-signed certificate.
---
```shell
sudo mkdir /etc/nginx/ssl ; cd /etc/nginx/ssl

var_fqdn=www.example.local

sudo openssl req -noenc -x509 -days 365 \
    -keyout ${var_fqdn}.key \
    -out ${var_fqdn}.crt \
    -subj "/CN=${var_fqdn}"
    #-newkey rsa:4096 # with a longer key

ls -al
```

Create an Nginx configuration file that uses the self-signed certificate mentioned above.
---
[Reference documentation.](https://nginx.org/en/docs/http/configuring_https_servers.html)
[Reference documentation.](https://nginx.org/en/docs/varindex.html)
```shell conf for website www.example.local
cd /etc/nginx/conf.d
sudo tee www.example.local.conf << EOF
server {
    listen              443 ssl;
    server_name         www.example.local;
    ssl_certificate     /etc/nginx/ssl/www.example.local.crt;
    ssl_certificate_key /etc/nginx/ssl/www.example.local.key;
    #ssl_protocols       TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on; # added
    location / {
      return 200 "host is \$host";
    }
}
EOF

sudo nginx -t
sudo systemctl reload nginx
```


```shell curl test website https://www.example.local
curl https://www.example.local
curl -k https://www.example.local # allow insecure connection

# Trust this self-signed certificate. (www.example.local.crt)
sudo cp /etc/nginx/ssl/www.example.local.crt /usr/local/share/ca-certificates
sudo update-ca-certificates
# The method to trust the certificate differs in different environments. Here’s an example of how to trust a self-signed certificate in Debian 12.

# The warning about the self-signed certificate in curl has disappeared.
curl https://www.example.local

```
> curl: (60) SSL certificate problem: self-signed certificate
More details here: https://curl.se/docs/sslcerts.html


Create a self-signed certificate with SAN (Subject Alternative Name).
---
```shell
cd /etc/nginx/ssl
ls -al
# Note that when OpenSSL is set to output key and CRT files with the same name, it will overwrite them directly without any warnings.

var_fqdn=www.example.local
var_domain=${var_fqdn#*.}

sudo openssl req -noenc -x509 -days 365 \
    -keyout ${var_fqdn}.key \
    -out ${var_fqdn}.crt \
    -subj "/CN=${var_fqdn}" \
    -addext "subjectAltName = DNS:${var_domain},DNS:${var_fqdn}"
# Use addext subjectAltName to set the SAN.

ls -al

openssl x509 -in ${var_fqdn}.crt -text -noout | grep -E 'Not After|Issuer' ;
openssl x509 -in ${var_fqdn}.crt -text -noout | grep 'Subject Alternative Name' -A1
```

Reload Nginx to apply the new self-signed certificate.
---
```shell reload configuration
sudo nginx -t
sudo systemctl reload nginx
```

```shell curl test
curl https://www.example.local
# Since it's a new certificate, curl will show the warning about using a self-signed certificate again.

# Then set up trust for the new certificate.
sudo cp /etc/nginx/ssl/www.example.local.crt /usr/local/share/ca-certificates
sudo update-ca-certificates
curl https://www.example.local

```


Take a small step forward by creating a wildcard self-signed certificate.
---
```shell
cd /etc/nginx/ssl
ls -al

var_fqdn=*.example.local
var_domain=${var_fqdn#*.} # Remove *. from var_fqdn to get example.local.
var_filename=wildcard.${var_domain}

sudo openssl req -noenc -x509 -days 365 \
    -keyout ${var_filename}.key \
    -out ${var_filename}.crt \
    -subj "/CN=${var_fqdn}" \
    -addext "subjectAltName = DNS:${var_domain},DNS:${var_fqdn}"

ls -al

openssl x509 -in ${var_filename}.crt -text -noout | grep -E 'Not After|Issuer'
openssl x509 -in ${var_filename}.crt -text -noout | grep 'Subject Alternative Name' -A1
```

Create a docs site to test the wildcard certificate.
---

```shell
cd /etc/nginx/conf.d
sudo tee docs.example.local.conf << EOF
server {
    listen              443 ssl;
    server_name         docs.example.local;
    ssl_certificate     /etc/nginx/ssl/wildcard.example.local.crt;
    ssl_certificate_key /etc/nginx/ssl/wildcard.example.local.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on; # added
    location / {
      return 200 "host is \$host";
    }
}
EOF
sudo nginx -t
sudo systemctl reload nginx
```



```shell curl test
curl https://docs.example.local

# As before, we will need to trust this wildcard self-signed certificate to avoid the warning.
sudo cp /etc/nginx/ssl/wildcard.example.local.crt /usr/local/share/ca-certificates
sudo update-ca-certificates
curl https://docs.example.local

```

Use the tool sslscan to inspect the website's certificate.
---
```shell
sudo apt install sslscan -y
sslscan www.example.local
sslscan docs.example.local
```

-------------------------------
Below is an example of creating a self-signed CA and using this CA to sign certificates.
-------------------------------

Generate a self-signed CA certificate.
---

```shell
cd /etc/nginx/ssl
var_domain=example.corp
sudo openssl req -noenc -x509 -days 3650 \
    -keyout ca-${var_domain}.key \
    -out ca-${var_domain}.crt \
    -subj "/CN=${var_domain}" \
    -addext "basicConstraints=critical,CA:TRUE" \
    -addext "keyUsage=critical,keyCertSign,cRLSign"

openssl x509 -in ca-${var_domain}.crt -text -noout | grep -E 'Not After|Issuer'
openssl x509 -in ca-${var_domain}.crt -text -noout | grep -E 'X509v3' -A1

```

Use the self-signed CA certificate to generate a CSR and CRT.
---
```shell
cd /etc/nginx/ssl
var_fqdn=www.example.corp
# Generate a key and CSR (Certificate Signing Request).
sudo openssl req -new -noenc \
   -keyout ${var_fqdn}.key \
   -out ${var_fqdn}.csr \
   -subj "/CN=${var_fqdn}" \
   -addext "subjectAltName = DNS:${var_domain},DNS:${var_fqdn}"

# Generate a CRT (certificate) from the CSR
sudo openssl x509 -req -days 365 \
   -CA ca-${var_domain}.crt \
   -CAkey ca-${var_domain}.key \
   -CAcreateserial \
   -in ${var_fqdn}.csr \
   -out ${var_fqdn}.crt \
   -copy_extensions copy # need this to copy san info

# Create a fullchain certificate by combining the website certificate with the CA certificate.
cat ${var_fqdn}.crt ca-${var_domain}.crt  | sudo tee ${var_fqdn}.fullchain.crt
openssl x509 -in www.example.corp.crt -noout -text

```

Let's test this self-signed certificate with www.example.corp.
---
```shell
cd /etc/nginx/conf.d
sudo tee www.example.corp.conf << EOF
server {
    listen              443 ssl;
    server_name         www.example.corp;
    ssl_certificate     /etc/nginx/ssl/www.example.corp.fullchain.crt;
    ssl_certificate_key /etc/nginx/ssl/www.example.corp.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on; # added
    location / {
      return 200 "host is \$host";
    }
}
EOF
sudo nginx -t
sudo systemctl reload nginx
```

```shell
curl https://www.example.corp

# The self-signed certificate trust issue will still occur. Here, we will add the CA certificate to the trusted list.
sudo cp /etc/nginx/ssl/ca-${var_domain}.crt /usr/local/share/ca-certificates
sudo update-ca-certificates

curl https://www.example.corp
```




Use the CA to generate a self-signed certificate for docs.example.corp.
---
```shell
cd /etc/nginx/ssl
var_fqdn=docs.example.corp
sudo openssl req -new -noenc \
   -keyout ${var_fqdn}.key \
   -out ${var_fqdn}.csr \
   -subj "/CN=${var_fqdn}" \
   -addext "subjectAltName = DNS:${var_fqdn}"  #no need for dns: example.corp


sudo openssl x509 -req -days 365 \
   -CA ca-${var_domain}.crt \
   -CAkey ca-${var_domain}.key \
   -CAcreateserial \
   -in ${var_fqdn}.csr \
   -out ${var_fqdn}.crt \
   -copy_extensions copy # need this to copy san info


cat ${var_fqdn}.crt ca-${var_domain}.crt  | sudo tee ${var_fqdn}.fullchain.crt
```

Create a configuration in Nginx for docs.example.corp.
---
```shell
cd /etc/nginx/conf.d
sudo tee docs.example.corp.conf << EOF
server {
    listen              443 ssl;
    server_name         docs.example.corp;
    ssl_certificate     /etc/nginx/ssl/docs.example.corp.fullchain.crt;
    ssl_certificate_key /etc/nginx/ssl/docs.example.corp.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on; # added
    location / {
      return 200 "host is \$host";
    }
}
EOF
sudo nginx -t
sudo systemctl reload nginx
```
```shell curl test
curl https://docs.example.corp # no need for trush again

```

Use OpenSSL to check if the key and CRT are a matching pair.
---
```shell
cd /etc/nginx/ssl/
ls

#for basic key and crt
sudo openssl rsa -modulus -noout -in www.example.local.key
openssl x509 -modulus -noout -in www.example.local.crt

#for wildcard key and crt
sudo openssl rsa -modulus -noout -in www.example.local.key | openssl sha256
openssl x509 -modulus -noout -in www.example.local.crt | openssl sha256

#for ca key,csr and crt
sudo openssl rsa -modulus -noout -in docs.example.corp.key | openssl sha256
openssl x509 -modulus -noout -in docs.example.corp.fullchain.crt | openssl sha256
openssl req -modulus -noout -in docs.example.corp.csr | openssl sha256
```






