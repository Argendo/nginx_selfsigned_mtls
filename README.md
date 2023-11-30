# nginx_selfsigned_mtls
Simple cheat sheet how to make Nginx MTLS authorization with self-signed certificates.


# CA

Для начала создаем центр сертификации Certificate Authority (CA) следующей командой:

```bash
openssl req \  
  -new \  
  -x509 \  
  -nodes \  
  -days 365 \  
  -subj '/CN=EXAMPLE-CA' \  
  -keyout ca.key \  
  -out ca.crt
```

На выходе получим два файла: _ca.key_ и  _ca.crt_ (файл ключа и сертификат соответственно).


# SERVER CERT AND KEY

Далее создадим сертификаты сервера.
Генерируем ключ командой: 
```bash
openssl genrsa \  
  -out server.key 2048
```

Теперь нам необходимо создать запрос на подпись или Certificate Signing Request (CSR). Допустим, мы создаем сертификат для домена _your-domain-he.re_, в таком случае необходимо указать это доменное имя в поле Common Name (CN).
```bash
openssl req \  
  -new \  
  -key server.key \  
  -subj '/CN=your-domain-he.re' \  
  -out server.csr
```

Поскольку мы создаем самоподписанный сертификат, необходимо генерировать наш сертификат при помощи CSR и нашего CA.
Сделать это можно следующей командой:

```bash
openssl x509 \  
  -req \  
  -in server.csr \  
  -CA ca.crt \  
  -CAkey ca.key \  
  -CAcreateserial \  
  -days 365 \  
  -out server.crt
```

Сертификат и ключ для сервера готовы.

# CLIENT CERT AND KEY

Все действия их секции для сервера необходимо повторить и для клиента:

## Ключ
```bash
openssl genrsa \  
  -out client.key 2048
```
## Запрос на подпись
```bash
openssl req \  
-new \  
-key client.key \  
-subj '/CN=my-client' \  
-out client.csr
```
## Сертификат
```bash
openssl x509 \  
-req \  
-in client.csr \  
-CA ca.crt \  
-CAkey ca.key \  
-CAcreateserial \  
-days 365 \  
-out client.crt
```

При необходимости клиентский сертификат можно упаковать в один файл pcks12 формата:
```bash
openssl pkcs12 -export -out client.p12 -inkey client.key -in client.crt -certfile /ca.crt 
```


# NGINX CONF

Примерная конфигурация nginx для mTLS авторизации выглядит так: 

```nginx
server {
    listen                  443 ssl;
    server_name your_domain_he.re www.your_domain_he.re;

    ssl_certificate /etc/nginx/certs/server/server.crt;
    ssl_certificate_key /etc/nginx/certs/server/server.key;
    ssl_protocols           TLSv1.2 TLSv1.3;
    ssl_ciphers             HIGH:!aNULL:!MD5;
    ssl_client_certificate  /etc/nginx/certs/ca/ca.crt;
    ssl_verify_client       on;

    location / {
        if ($ssl_client_verify != SUCCESS) { return 403; }

        proxy_pass http://localhost:8080;

    }
}
```
