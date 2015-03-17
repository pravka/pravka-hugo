+++
date = "2015-03-14T15:40:01-07:00"
draft = false
title = "Mutual Authentication with Client Certificates in Nginx"
slug = "nginx-mutual-auth"
tags = ["nginx", "ssl"]
+++


Create the CA Key and Certificate for signing Client Certs:

```
openssl genrsa -des3 -out ca.key 4096 openssl req -new -x509 -days 365 -key ca.key -out ca.crt
openssl genrsa -des3 -out ca.key 4096
openssl req -new -x509 -days 365 -key ca.key -out ca.crt
```

Self-sign the certificate — for personal/testing purposes only:

```
openssl x509 -req -days 365 -in server.csr -CA ca.crt -CAkey ca.key -set_serial 01 -out server.crt
```

Create the Client Key and CSR:

```
openssl genrsa -des3 -out client.key 1024 openssl req -new -key client.key -out client.csr
openssl genrsa -des3 -out client.key 1024
openssl req -new -key client.key -out client.csr
```

Finally, sign the client certificate with our CA cert:

```    
openssl x509 -req -days 365 -in client.csr -CA ca.crt -CAkey ca.key -set_serial 01 -out client.crt
```

And move the certs into place:
```
mkdir -p /etc/nginx/certs
mv server.{key,crt} /etc/nginx/certs
mv ca.crt /etc/nginx/certs
```

In /etc/nginx/conf.d/default:

```    
upstream backend {
    server ip:port;
}
server {
    listen        443;
    ssl on;
    server_name ;
    ssl_certificate      /etc/nginx/certs/server.crt;
    ssl_certificate_key  /etc/nginx/certs/server.key;
    ssl_client_certificate /etc/nginx/certs/ca.crt;
    ssl_verify_client on;
    ssl_verify_depth 2;
    ssl_protocols  SSLv2 SSLv3 TLSv1;
    ssl_ciphers  ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP;
    ssl_prefer_server_ciphers   on;
    location / {
      proxy_pass  http://backend;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header Host $http_host;
    } 
}
```

To test connecting with a client certificate:
```    
curl -v -s -k --key client.key --cert client.crt https://your_hostname.com
```

Per-location mutual authentication

In some cases, you may want to have a location that’s accessible without presenting a client certificate. You can accomplish this with an `if` statement in Nginx.
```    
location /unprotected {
    proxy_pass  http://backend;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
location / {
  if ($ssl_client_verify != SUCCESS) { return 403; }
    proxy_pass  http://backend;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header ssl.client_cert $ssl_client_raw_cert;
    proxy_set_header ssl.client_cert.subjectdn $ssl_client_s_dn;
}
```

Multiple CAs

You can only specify `ssl_client_certificate` once. What if you have multiple CAs (i.e. subordinate CAs) in a chain that you all want to trust? Simple.

```    
cat ca1.crt ca2.crt ca3.crt >> ca.pem
```

Then use the resulting pem as the value for `ssl_client_certificate` in Nginx.