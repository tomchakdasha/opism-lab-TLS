<!--
Бланк звіту до лабораторної роботи «TLS та сертифікати».
Скопіювати цей файл у власний публічний репозиторій звіту як README.md і заповнити.
Вивід команд вставляти текстом у блоки коду, а не знімками екрана.
Порожні поля не допускаються: якщо щось не вдалося, записати, що саме і чому.
-->

# Лабораторна робота 2. TLS та сертифікати

| Поле | Значення |
|---|---|
| ПІБ | Томчак Дар'я Ігорівна |
| Група | ІПЗ-2.01 |
| Ідентифікатор | se-201-31 |
| Дата виконання | 15.11.2026 |
| Середовище | GitHub Codespaces |
| Версія OpenSSL | OpenSSL 3.0.13 30 Jan 2024 (Library: OpenSSL 3.0.13 30 Jan 2024) |

## Параметри варіанта

<!-- Переписати з картки stand/cards/<id>.md. -->

| Параметр | Значення |
|---|---|
| Віддалений сервер | front.se-201-31.lab |
| Імена для сертифіката | www.se-201-31.lab, api.se-201-31.lab |
| Ваш сервер (IP) | 10.53.1.1 |

<!-- Якщо стенд перезапускався під час роботи, вказати, після якого завдання. -->

## Завдання 1. HTTP і HTTPS у дампі трафіку

Вивід кроку 1.1:

```text
root@se-201-31:/lab/work# cd /lab/work
timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 80' > t1-http.txt &
sleep 1; curl -s http://front.se-201-31.lab/; wait
grep -a -E 'GET /|Host:|server: web' t1-http.txt
[1] 42
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
server: web
Host: front.se-201-31.lab
SNI: 
protocol: HTTP/1.1 
client: 10.53.1.1
10 packets captured
10 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 80' > t1-http.txt
09:14:56.267651 IP 10.53.1.1.42160 > 10.53.0.80.80: Flags [P.], seq 1:83, ack 1, win 502, options [nop,nop,TS val 488068454 ecr 906755244], length 82: HTTP: GET / HTTP/1.1
..Uf6...GET / HTTP/1.1
Host: front.se-201-31.lab
server: web
Host: front.se-201-31.lab

```

Вивід кроку 1.2:

```text
root@se-201-31:/lab/work# timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t1-https.txt &
sleep 1; curl -s --cacert $STAND_CA https://front.se-201-31.lab/; wait
grep -a -c -E 'GET /|server: web' t1-https.txt
grep -a -o 'front.se-201-31.lab' t1-https.txt | wc -l
[1] 48
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
server: web
Host: front.se-201-31.lab
SNI: front.se-201-31.lab
protocol: HTTP/2.0 TLSv1.3
client: 10.53.1.1
22 packets captured
22 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t1-https.txt
0
1

```

| Що шукали в дампі | HTTP | HTTPS |
|---|---|---|
| рядок запиту `GET /` | GET / HTTP/1.1 | 0 |
| заголовок `Host` | Host: front.se-201-31.lab | нема |
| тіло відповіді (`server: web`) | server: web | 0 |
| ім'я віддаленого сервера (кількість входжень) | front.se-201-31.lab; 2 | front.se-201-31.lab; 1 |

1. У якому повідомленні і в якому полі передається ім'я сервера, яке видно в дампі HTTPS?

> У повідомленні Client Hello в SNI(Server Name Indication)

2. Чому рядок `SNI:` у відповіді сервера порожній для HTTP і заповнений для HTTPS?

> HTTP не виконує TLS-рукостискання, тому не передає SNI, а HTTPS встановлює TLS-з'єднання, клієнт передає домен у ClientHello, тому сервер повернув SNI: front.se-201-31.lab

3. Рядок `protocol:` показує HTTP/1.1 для запиту по HTTP і HTTP/2.0 для запиту по HTTPS, хоча обидва запити виконано однією командою `curl` без додаткових ключів. Що визначило версію HTTP у другому випадку?

> Протокол було узгоджено між клієнтом і сервером через ALPN під час TLS-рукостискання, під час запиту по HTTP цього не відбувається

## Завдання 2. Рукостискання по записах

Вивід кроку 2.1 (TLS 1.3):

```text
root@se-201-31:/lab/work# openssl s_client -connect front.se-201-31.lab:443 -servername front.se-201-31.lab -alpn h2,http/1.1 \
  -CAfile $STAND_CA -trace </dev/null > t2-13.txt 2>&1
grep -a -E '^(Sent|Received) Record|Content Type|^    [A-Z][A-Za-z]+, Length' t2-13.txt
grep -a -E '^(ALPN|New,)|Verify return' t2-13.txt
Sent Record
  Content Type = Handshake (22)
    ClientHello, Length=330
Received Record
  Content Type = Handshake (22)
    ServerHello, Length=118
Received Record
  Content Type = ChangeCipherSpec (20)
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    EncryptedExtensions, Length=15
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    Certificate, Length=989
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    CertificateVerify, Length=74
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    Finished, Length=48
Sent Record
  Content Type = ChangeCipherSpec (20)
Sent Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    Finished, Length=48
Sent Record
  Content Type = ApplicationData (23)
  Inner Content Type = Alert (21)
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
ALPN protocol: h2
Verify return code: 0 (ok)

```

Вивід кроку 2.2 (TLS 1.2):

```text
root@se-201-31:/lab/work# openssl s_client -connect front.se-201-31.lab:443 -servername front.se-201-31.lab -alpn h2,http/1.1 \
  -CAfile $STAND_CA -tls1_2 -trace </dev/null > t2-12.txt 2>&1
grep -a -E '^(Sent|Received) Record|Content Type|^    [A-Z][A-Za-z]+, Length' t2-12.txt
grep -a -E '^(ALPN|New,)|Verify return' t2-12.txt
Sent Record
  Content Type = Handshake (22)
    ClientHello, Length=225
Received Record
  Content Type = Handshake (22)
    ServerHello, Length=74
Received Record
  Content Type = Handshake (22)
    Certificate, Length=984
Received Record
  Content Type = Handshake (22)
    ServerKeyExchange, Length=111
Received Record
  Content Type = Handshake (22)
    ServerHelloDone, Length=0
Sent Record
  Content Type = Handshake (22)
    ClientKeyExchange, Length=33
Sent Record
  Content Type = ChangeCipherSpec (20)
Sent Record
  Content Type = Handshake (22)
    Finished, Length=12
Received Record
  Content Type = Handshake (22)
    NewSessionTicket, Length=214
Received Record
  Content Type = ChangeCipherSpec (20)
Received Record
  Content Type = Handshake (22)
    Finished, Length=12
Received Record
  Content Type = ApplicationData (23)
Sent Record
  Content Type = Alert (21)
New, TLSv1.2, Cipher is ECDHE-ECDSA-AES256-GCM-SHA384
ALPN protocol: h2
    Verify return code: 0 (ok)

```

<!-- Один рядок на кожен запис з виводу 2.1, у тому самому порядку. Напрям: К→С або С→К. -->

| № | Напрям | `Content Type` | `Inner Content Type` | Повідомлення | `Length` |
|---|---|---|---|---|---|
| 1 | К→С | Handshake (22) | - | ClientHello | 330 |
| 2 | С→К | Handshake (22) | - | ServerHello | 118 |
| 3 | С→К | ChangeCipherSpec (20) | - | - | - |
| 4 | С→К | ApplicationData (23) | Handshake (22) | EncryptedExtensions | 15 |
| 5 | С→К | ApplicationData (23) | Handshake (22) | Certificate | 989 |
| 6 | С→К | ApplicationData (23) | Handshake (22) | CertificateVerify | 74 |
| 7 | С→К | ApplicationData (23) | Handshake (22) | Finished | 48 |
| 8 | К→С | ChangeCipherSpec (20) | - | - | - |
| 9 | К→С | ApplicationData (23) | Handshake (22) | Finished | 48 |
| 10 | К→С | ApplicationData (23) | Alert (21) | - | - |

Вивід кроків 2.3 і 2.4:

```text
root@se-201-31:/lab/work# grep -a -A3 'extension_type=server_name' t2-13.txt | head -4
        extension_type=server_name(0), length=24
          0000 - 00 16 00 00 13 66 72 6f-6e 74 2e 73 65 2d 32   .....front.se-2
          000f - 30 31 2d 33 31 2e 6c 61-62                     01-31.lab
        extension_type=ec_point_formats(11), length=4
root@se-201-31:/lab/work# for v in 1.3 1.2; do
  timeout 6 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t2-dump-$v.txt &
  sleep 1; curl -s -o /dev/null --tlsv1.2 --tls-max $v --cacert $STAND_CA https://front.se-201-31.lab/; wait
  echo "TLS $v: $(grep -a -c 'OPISM Stand Intermediate' t2-dump-$v.txt)"
done
[1] 64
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
23 packets captured
23 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 6 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t2-dump-$v.txt
TLS 1.3: 0
[1] 70
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
21 packets captured
21 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 6 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t2-dump-$v.txt
TLS 1.2: 2

```

| Показник | TLS 1.3 | TLS 1.2 |
|---|---|---|
| Довжина ClientHello, байт | 330 | 225 |
| Номер першого запису, тип якого спостерігач не бачить | 4 | 8 |
| Кількість RTT від ClientHello до першого HTTP-запиту | 1 | 2 |
| Чи видно сертифікат у дампі (результат 2.4) | Ні, TLS 1.3: 0 | Так, TLS 1.2: 2 |
| Обраний протокол ALPN | h2 | h2 |

1. За якими рядками виводу 2.1 визначено, з якого запису починається шифрування? Що бачить спостерігач замість типу кожного наступного запису?

> Шифрування в TLS 1.3 починається з четвертого запису, що видно за рядками Content Type = ApplicationData (23) та Inner Content Type = Handshake (22). Спостерігач бачить лише зовнішній тип ApplicationData (23), але не бачить справжній тип і вміст зашифрованого повідомлення

2. Чому ClientHello у TLS 1.3 довший, ніж у TLS 1.2? Назвіть розширення, яке за рахунок цього скорочує рукостискання на один RTT.

> ClientHello у TLS 1.3 довший, бо містить більше інформації для встановлення захищеного з'єднання, розширення key_share дозволяє одразу передати дані для обміну ключами, а рукостискання скорочується з 2 RTT до 1 RTT

3. Довжина розширення `server_name` у виводі 2.3 становить 17 байт для імені з 12 символів. З яких частин складається це значення і яким воно буде для вашого імені?

> Довжина server_name складається з 2 байтів для довжини списку, 1 байта для типу імені, 2 байтів для довжини імені та самого домену. Для мого домену front.se-201-31.lab, який містить 19 символів, довжина становить 2 + 1 + 2 + 19 = 24 байти, що підтверджує рядок length=24

4. Навіщо в TLS 1.3 передається запис ChangeCipherSpec, якщо в рукостисканні він нічого не змінює?

> ChangeCipherSpec у TLS 1.3 потрібен для сумісності зі старими системами та мережевими пристроями, він не змінює ключі шифрування, а лише допомагає забезпечити правильну роботу з'єднання

## Завдання 3. Сертифікат для власного сервера

Вивід кроків 3.1 і 3.2:

```text
root@se-201-31:/lab/work# acme-root
openssl x509 -in /lab/tls/acme-root.crt -noout -subject -dates
збережено /lab/tls/acme-root.crt
subject=CN = Pebble Root CA 053ade
notBefore=Nov 13 09:01:32 2026 GMT
notAfter=Nov 13 09:01:32 2056 GMT
root@se-201-31:/lab/work# curl -sS --cacert /lab/tls/acme-root.crt https://www.se-201-31.lab/
curl: (60) SSL certificate problem: self-signed certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.

```

Вивід кроку 3.3 (останні рядки `lego` і рядки журналу nginx):

```text
root@se-201-31:/lab/work# lego --server $ACME_DIR --email admin@se-201-31.lab --accept-tos --path /lab/work/lego \
  --http --http.webroot /var/www/acme \
  --domains www.se-201-31.lab --domains api.se-201-31.lab run --no-bundle
grep acme-challenge /var/log/nginx/access.log
2026/11/13 09:25:40 No key found for account admin@se-201-31.lab. Generating a P256 key.
2026/11/13 09:25:40 Saved key to /lab/work/lego/accounts/ca.lab_14000/admin@se-201-31.lab/keys/admin@se-201-31.lab.key
2026/11/13 09:25:40 [INFO] acme: Registering account for admin@se-201-31.lab
!!!! HEADS UP !!!!

Your account credentials have been saved in your Let's Encrypt
configuration directory at "/lab/work/lego/accounts".

You should make a secure backup of this folder now. This
configuration directory will also contain certificates and
private keys obtained from Let's Encrypt so making regular
backups of this folder is ideal.
2026/11/13 09:25:40 [INFO] [www.se-201-31.lab, api.se-201-31.lab] acme: Obtaining SAN certificate
2026/11/13 09:25:41 [INFO] [www.se-201-31.lab] AuthURL: https://ca.lab:14000/authZ/Qy8wgRZ8dlj8Qw-fqg6wlDCdeH4A_kO2_t0JVlsZ1rA
2026/11/13 09:25:41 [INFO] [api.se-201-31.lab] AuthURL: https://ca.lab:14000/authZ/m9n-L18MTOKz7OcNLzYQ84V3ETSn1UrTpl2Q2xaZTgE
2026/11/13 09:25:41 [INFO] [www.se-201-31.lab] acme: Could not find solver for: tls-alpn-01
2026/11/13 09:25:41 [INFO] [www.se-201-31.lab] acme: use http-01 solver
2026/11/13 09:25:41 [INFO] [api.se-201-31.lab] acme: Could not find solver for: tls-alpn-01
2026/11/13 09:25:41 [INFO] [api.se-201-31.lab] acme: use http-01 solver
2026/11/13 09:25:41 [INFO] [www.se-201-31.lab] acme: Trying to solve HTTP-01
2026/11/13 09:25:44 [INFO] [www.se-201-31.lab] The server validated our request
2026/11/13 09:25:44 [INFO] [api.se-201-31.lab] acme: Trying to solve HTTP-01
2026/11/13 09:25:48 [INFO] [api.se-201-31.lab] The server validated our request
2026/11/13 09:25:48 [INFO] [www.se-201-31.lab, api.se-201-31.lab] acme: Validations succeeded; requesting certificates
2026/11/13 09:25:48 [INFO] Wait for certificate [timeout: 30s, interval: 500ms]
2026/11/13 09:25:49 [INFO] [www.se-201-31.lab] Server responded with a certificate.
10.53.0.10 - - [13/Nov/2026:09:25:41 +0000] "GET /.well-known/acme-challenge/GS45Kmo4B2YNdjFQYap3waXwOa9UZWk_GPEGc_pKLl4 HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [13/Nov/2026:09:25:41 +0000] "GET /.well-known/acme-challenge/GS45Kmo4B2YNdjFQYap3waXwOa9UZWk_GPEGc_pKLl4 HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [13/Nov/2026:09:25:41 +0000] "GET /.well-known/acme-challenge/GS45Kmo4B2YNdjFQYap3waXwOa9UZWk_GPEGc_pKLl4 HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [13/Nov/2026:09:25:44 +0000] "GET /.well-known/acme-challenge/iws1vSi4A5uIoaoO4vJkJeUaUsAdOiD2i0w9DSLeIDg HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [13/Nov/2026:09:25:44 +0000] "GET /.well-known/acme-challenge/iws1vSi4A5uIoaoO4vJkJeUaUsAdOiD2i0w9DSLeIDg HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [13/Nov/2026:09:25:44 +0000] "GET /.well-known/acme-challenge/iws1vSi4A5uIoaoO4vJkJeUaUsAdOiD2i0w9DSLeIDg HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"

```

Вивід кроку 3.4:

```text
root@se-201-31:/lab/work# cd /lab/work/lego/certificates
openssl x509 -in www.se-201-31.lab.crt -noout -subject -issuer -dates -serial -ext subjectAltName
openssl x509 -in www.se-201-31.lab.issuer.crt -noout -subject -issuer
subject=CN = www.se-201-31.lab
issuer=CN = Pebble Intermediate CA 17f86f
notBefore=Nov 13 09:25:48 2026 GMT
notAfter=Nov 13 09:25:47 2031 GMT
serial=76A3D255F7DCC522
X509v3 Subject Alternative Name: 
    DNS:www.se-201-31.lab, DNS:api.se-201-31.lab
subject=CN = Pebble Intermediate CA 17f86f
issuer=CN = Pebble Root CA 053ade

```

Вивід кроку 3.5:

```text
root@se-201-31:/lab/work/lego/certificates# cp www.se-201-31.lab.crt /lab/tls/server.crt
cp www.se-201-31.lab.key /lab/tls/server.key
web-reload
curl -sS --cacert /lab/tls/acme-root.crt https://www.se-201-31.lab/
openssl s_client -connect www.se-201-31.lab:443 -servername www.se-201-31.lab -CAfile /lab/tls/acme-root.crt \
  </dev/null 2>/dev/null | sed -n '/^Certificate chain/,/^---/p;/^Verify return code/p'
nginx: the configuration file /lab/nginx.conf syntax is ok
nginx: configuration file /lab/nginx.conf test is successful
2026/11/13 09:27:27 [notice] 93#93: signal process started
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
Certificate chain
 0 s:CN = www.se-201-31.lab
   i:CN = Pebble Intermediate CA 17f86f
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 13 09:25:48 2026 GMT; NotAfter: Nov 13 09:25:47 2031 GMT
---
Verify return code: 21 (unable to verify the first certificate)

```

Вивід кроку 3.6:

```text
root@se-201-31:/lab/work/lego/certificates# cat www.se-201-31.lab.crt www.se-201-31.lab.issuer.crt > /lab/tls/server.crt
web-reload
nginx: the configuration file /lab/nginx.conf syntax is ok
nginx: configuration file /lab/nginx.conf test is successful
2026/11/13 09:28:08 [notice] 101#101: signal process started
root@se-201-31:/lab/work/lego/certificates# curl -sS --cacert /lab/tls/acme-root.crt https://www.se-201-31.lab/
openssl s_client -connect www.se-201-31.lab:443 -servername www.se-201-31.lab -CAfile /lab/tls/acme-root.crt \
  </dev/null 2>/dev/null | sed -n '/^Certificate chain/,/^---/p;/^Verify return code/p'
curl -sS --cacert /lab/tls/acme-root.crt https://api.se-201-31.lab/
server: se-201-31 (https)
Host: www.se-201-31.lab
SNI: www.se-201-31.lab
protocol: TLSv1.3
Certificate chain
 0 s:CN = www.se-201-31.lab
   i:CN = Pebble Intermediate CA 17f86f
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 13 09:25:48 2026 GMT; NotAfter: Nov 13 09:25:47 2031 GMT
 1 s:CN = Pebble Intermediate CA 17f86f
   i:CN = Pebble Root CA 053ade
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 13 09:01:32 2026 GMT; NotAfter: Nov 13 09:01:32 2056 GMT
---
Verify return code: 0 (ok)
server: se-201-31 (https)
Host: api.se-201-31.lab
SNI: api.se-201-31.lab
protocol: TLSv1.3

```

| Параметр | Значення |
|---|---|
| Видавець сертифіката сервера | CN = Pebble Intermediate CA 17f86f |
| Видавець сертифіката видавця | CN = Pebble Root CA 053ade |
| Імена в SAN | DNS:www.se-201-31.lab, DNS:api.se-201-31.lab |
| Строк дії (від – до) | notBefore=Nov 13 09:25:48 2026 GMT <br> notAfter=Nov 13 09:25:47 2031 GMT |
| Тип ключа сервера і алгоритм підпису | a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256 |
| Кількість сертифікатів, які надсилає сервер, у кроці 3.5 і в кроці 3.6 | 1; 2|

1. Хто і як перевірив, що ви контролюєте обидва імена? Вкажіть рядки виводу, які це підтверджують.

> Обидва домени перевірив центр сертифікації ACME за допомогою HTTP-01, він звернувся до адрес www.se-201-31.lab і api.se-201-31.lab та перевірив, чи сервер повертає правильні дані для підтвердження, обидві перевірки пройшли успішно, що підтверджують рядки:

```
2026/11/13 09:25:44 [INFO] [www.se-201-31.lab] The server validated our request
2026/11/13 09:25:44 [INFO] [api.se-201-31.lab] acme: Trying to solve HTTP-01
2026/11/13 09:25:48 [INFO] [api.se-201-31.lab] The server validated our request
2026/11/13 09:25:48 [INFO] [www.se-201-31.lab, api.se-201-31.lab] acme: Validations succeeded; requesting certificates
```

2. Чому в кроці 3.5 перевірка не пройшла, хоча корінь центру сертифікації вказано в `--cacert`, а сертифікат виданий правильно?

> Сервер надсилав лише свій сертифікат, без сертифіката проміжного центру, тому клієнт не зміг перевірити повний ланцюжок довіри, навіть маючи кореневий сертифікат, що підтверджує рядок:
 
 ```
Verify return code: 21 (unable to verify the first certificate)
 ```

3. У виводі ланцюжка для сертифіката сервера вказано `id-ecPublicKey` і `RSA-SHA256`. Що означає кожне з цих значень і чому вони не суперечать одне одному?

> id-ecPublicKey вказує на тип відкритого ключа власника сертифіката, а RSA-SHA256 позначає алгоритм цифрового підпису, яким CA підписав цей сертифікат, у створенні та використанні сертифіката беруть участь два різні ключі, які належать різним сторонам, тому вони не суперечать

4. Що станеться з перевіркою вашого сервера після перезапуску стенда і чому? Що в реальній інфраструктурі відповідає цій ситуації?

> Після перезапуску стенда кореневий сертифікат ACME зміниться, тому раніше виданий сертифікат сервера більше не проходитиме перевірку з новим коренем, раніше виданий сертифікат сервера залишиться підписаним старим проміжним і кореневим сертифікатами, через що клієнт, який довіряє лише новому кореневому сертифікату, не зможе перевірити старий сертифікат сервера, і перевірка TLS завершиться помилкою. У реальній інфраструктурі така ситуація відповідає зміні кореневого центру сертифікації

## Завдання 4. Діагностика несправностей

Вивід кроку 4.1:

```text
root@se-201-31:/lab/work/lego/certificates# for i in 1 2 3 4 5; do
  n=f$i.se-201-31.lab; echo "=== $n"
  curl -sS --cacert $STAND_CA https://$n/ 2>&1 | head -1
  openssl s_client -connect $n:443 -servername $n -CAfile $STAND_CA </dev/null 2>/dev/null \
    | sed -n '/^Certificate chain/,/^---/p;/^Verify return code/p' | grep -v PKEY
done
=== f1.se-201-31.lab
server: web, f1
Certificate chain
 0 s:CN = f1.se-201-31.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 13 09:01:31 2026 GMT; NotAfter: Feb 11 09:01:31 2027 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 13 09:01:31 2026 GMT; NotAfter: Nov 12 09:01:31 2031 GMT
---
Verify return code: 0 (ok)
=== f2.se-201-31.lab
curl: (60) SSL certificate problem: unable to get local issuer certificate
Certificate chain
 0 s:CN = f2.se-201-31.lab
   i:O = Foreign Org, CN = Foreign Intermediate CA
   v:NotBefore: Nov 13 09:01:31 2026 GMT; NotAfter: Feb 11 09:01:31 2027 GMT
 1 s:O = Foreign Org, CN = Foreign Intermediate CA
   i:O = Foreign Org, CN = Foreign Root CA
   v:NotBefore: Nov 13 09:01:31 2026 GMT; NotAfter: Nov 12 09:01:31 2031 GMT
---
Verify return code: 20 (unable to get local issuer certificate)
=== f3.se-201-31.lab
curl: (60) SSL certificate problem: certificate has expired
Certificate chain
 0 s:CN = f3.se-201-31.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Apr 27 09:01:31 2026 GMT; NotAfter: Oct 24 09:01:31 2026 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 13 09:01:31 2026 GMT; NotAfter: Nov 12 09:01:31 2031 GMT
---
Verify return code: 10 (certificate has expired)
=== f4.se-201-31.lab
curl: (60) SSL certificate problem: unable to get local issuer certificate
Certificate chain
 0 s:CN = f4.se-201-31.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 13 09:01:31 2026 GMT; NotAfter: Feb 11 09:01:31 2027 GMT
---
Verify return code: 21 (unable to verify the first certificate)
=== f5.se-201-31.lab
curl: (60) SSL certificate problem: self-signed certificate
Certificate chain
 0 s:CN = f5.se-201-31.lab
   i:CN = f5.se-201-31.lab
   v:NotBefore: Nov 13 09:01:31 2026 GMT; NotAfter: Feb 11 09:01:31 2027 GMT
---
Verify return code: 18 (self-signed certificate)

```

| Сервер | Помилка `curl` | `Verify return code` | Яка перевірка не пройшла | Рядок виводу, що це доводить | Що має змінити власник сервера |
|---|---|---|---|---|---|
| f1 | - | 0 (ok) | Усі перевірки успішні | server: web, f1 <br> Verify return code: 0 (ok) | Нічого |
| f2 | unable to get local issuer certificate | 20 (unable to get local issuer certificate) | Перевірка довіри до центру сертифікації, не вдалося отримати сертифікат локального видавця | i:O = Foreign Org, CN = Foreign Root CA <br> Verify return code: 20 (unable to get local issuer certificate) | Отримати сертифікат від довіреного центру |
| f3 | certificate has expired | 10 (certificate has expired) | Перевірка терміну дії сертефікату, прострочений сертефікат | v:NotBefore: Apr 27 09:01:31 2026 GMT; NotAfter: Oct 24 09:01:31 2026 GMT <br> Verify return code: 10 (certificate has expired) | Отримати новий діючий сертифікат |
| f4 | unable to get local issuer certificate | 21 (unable to verify the first certificate) | Перевірка ланцюжка сертифікатів, не вдалося перевірити перший сертефікат | У виводі відсутній сертефікат під номером 1: <br> Certificate chain <br> 0 s:CN = f4.se-201-31.lab <br> i:O = OPISM Lab, CN = OPISM Stand Intermediate CA <br> v:NotBefore: Nov 13 09:01:31 2026 GMT; NotAfter: Feb 11 09:01:31 2027 GMT <br> --- <br> Verify return code: 21 (unable to verify the first certificate) | Отримати та встановити новий чинний сертифікат |
| f5 | self-signed certificate | 18 (self-signed certificate) | Перевірка довіри до сертифіката, самопідписаний сертефікат | 0 s:CN = f5.se-201-31.lab <br> i:CN = f5.se-201-31.lab <br> Verify return code: 18 (self-signed certificate) | Замінити самопідписаний сертифікат на сертифікат від довіреного центру |

1. Для сервера з помилкою `unable to get local issuer certificate` визначте, яка це несправність: сервер не надсилає проміжний сертифікат чи сертифікат виданий центром, якому клієнт не довіряє. Чим ці дві несправності відрізняються у виводі `openssl s_client`?

> Для f2 сервер надає сертифікат сервера 0 і проміжного CA 1, але ланцюжок веде до Foreign Root CA, якого немає серед довірених коренів Verify return code: 20. Для f4 сервер надає лише сертифікат 0, хоча його видав OPISM Stand Intermediate CA; сертефікату проміжного центру немає Verify return code: 21. Тому у f2 несправність довіри до CA, а у f4 неповний ланцюжок

2. Якщо серед ваших серверів є сервер з невідповідним ім'ям, чому `openssl s_client` повертає для нього `Verify return code: 0 (ok)`, а `curl` повідомляє про помилку? Якщо такого сервера немає, дайте відповідь для загального випадку.

> openssl s_client у використаній команді перевіряє ланцюжок довіри сертифіката, але не перевіряє відповідність доменного імені без додаткового параметра -verify_hostname, тому він може показати 0 (ok), а curl додатково перевіряє, чи є ім'я сайту в полі SAN сертифіката, якщо такого імені немає, curl повідомить про помилку

3. Які з п'яти серверів відповіли б на запит `curl -k`? Що при цьому втрачається порівняно з `--cacert`?

> На запит curl -k відповіли б усі п'ять серверів, параметр -k вимикає перевірку сертифіката; --cacert дозволяє перевіряти сертифікат за допомогою довіреного кореневого центру, а -k пропускає перевірку довіри, терміну дії та відповідності імені, через що з'єднання залишається зашифрованим, але клієнт не може бути впевненим, що підключився саме до потрібного сервера

## Завдання 5. Ціна нового з'єднання

**Прогноз** (записати до вимірювання). Скільки RTT мине до завершення кожного етапу:

| Етап | TLS 1.3, нове з'єднання | TLS 1.2, нове з'єднання | Повторне використання з'єднання |
|---|---|---|---|
| `time_connect` | 2 RTT | 2 RTT | 0 RTT |
| `time_appconnect` | 3 RTT | 4 RTT | 0 RTT |
| `time_starttransfer` | 4 RTT | 5 RTT | 1 RTT |

Вивід кроків 5.2 і 5.3:

```text
root@se-201-31:/lab/work# F='%{time_namelookup} %{time_connect} %{time_appconnect} %{time_starttransfer} %{time_total} %{http_version} %{num_connects}\n'
U=https://front.se-201-31.lab/
for i in 1 2 3; do curl -s -o /dev/null --cacert $STAND_CA -w "$F" $U; done
for i in 1 2 3; do curl -s -o /dev/null --tls-max 1.2 --cacert $STAND_CA -w "$F" $U; done
0.050979 0.101252 0.130848 0.156690 0.156740 2 1
0.025741 0.050875 0.079457 0.104808 0.104876 2 1
0.025446 0.050631 0.079294 0.104676 0.104738 2 1
0.025648 0.050780 0.104640 0.130014 0.130125 2 1
0.025491 0.050684 0.107123 0.132527 0.132582 2 1
0.025498 0.050658 0.104712 0.130034 0.130119 2 1
root@se-201-31:/lab/work# curl -s --cacert $STAND_CA -w "$F" -o /dev/null $U -o /dev/null $U -o /dev/null $U
0.025533 0.050703 0.079650 0.105021 0.105106 2 1
0.000014 0.000000 0.000000 0.025337 0.025388 2 0
0.000034 0.000000 0.000000 0.025628 0.025706 2 0

```

<!-- Значення в мілісекундах. Для нових з'єднань — медіана трьох вимірювань. -->

| Запит | `namelookup` | `connect` | `appconnect` | `starttransfer` | `total` | `num_connects` |
|---|---|---|---|---|---|---|
| TLS 1.3, нове | 25.741 | 50.875 | 79.457 | 104.808 | 104.876 | 1 |
| TLS 1.2, нове | 25.498 | 50.684 | 104.712 | 130.034 | 130.125 | 1 |
| 5.3, перший запит | 25.533 | 50.703 | 79.650 | 105.021 | 105.106 | 1 |
| 5.3, другий запит | 0.014 | 0 | 0 | 25.337 | 25.388 | 0 |
| 5.3, третій запит | 0.034 | 0 | 0 | 25.628 | 25.706 | 0 |

RTT, визначений вимірюванням, і з якого значення його визначено:

> RTT близько дорівнює 25,16 мс; різниця між time_connect і time_namelookup для TLS 1.3 це 25,134 мс, а для TLS 1.2 це 25,186 мс, обидва результати близько дорівнюють 25,16 мс

1. Виразіть кожне виміряне значення через RTT. Де вимірювання відрізняється від цілого числа RTT і чим це пояснюється?

| Запит | `namelookup` | `connect` | `appconnect` | `starttransfer` | `total` | `num_connects` |
|---|---|---|---|---|---|---|
| TLS 1.3, нове | 1,02 | 2,02 | 3,16 | 4,17 | 4,17 | 1 |
| TLS 1.2, нове | 1,01 | 2,01 | 4,16 | 5,17 | 5,17 | 1 |
| 5.3, перший запит | 1,01 | 2,01 | 3,17 | 4,17 | 4,18 | 1 |
| 5.3, другий запит | 0,00056 | 0 | 0 | 1,007 | 1,009 | 0 |
| 5.3, третій запит | 0,00135 | 0 | 0 | 1,018 | 1,022 | 0 |

Виміряні значення близькі до цілих RTT, але мають невеликі відхилення, це пояснюється тим, що curl враховує не лише мережеву затримку, а й час обробки запитів та встановлення захищеного з'єднання. Повторні запити займають приблизно 1 RTT, оскільки використовують уже встановлене з'єднання

2. Чому `time_namelookup` не близький до нуля, хоча резолвер уже має ім'я в кеші? За яких умов він був би приблизно вдвічі більшим?

> time_namelookup не дорівнює нулю, тому що клієнту все одно потрібно звернутися до DNS-сервера, навіть якщо ім'я вже є в його кеші, близько 1 RTT, якщо імені немає в кеші DNS-сервера, йому потрібно звернутися до іншого сервера, тому час може збільшитися приблизно до 2 RTT

3. Сервіс виконує 200 послідовних запитів до цього сервера. Розрахуйте, скільки часу займе лише очікування мережі, якщо кожен запит встановлює нове з'єднання TLS 1.3, нове з'єднання TLS 1.2 і якщо всі запити йдуть через одне з'єднання.

> TLS 1.3: <br> 200 * 4,17 RTT * 25,16 = 20 983,44 мс = 20,983 с <br> TLS 1.2: <br> 200 * 5,17 RTT * 25,16 = 26015,44 мс = 26,015 с <br> Одне з'єднання: <br> (4 + 199 * 1,025) * 25,16 = 5 174,34 мс = 5,17 с

## Висновки

**Що виявилося несподіваним.** Результат, який не збігся з очікуванням, і рядок виводу, де це видно:

> Несподіваним було те, що сервери f2 і f4 показали однакову помилку curl: unable to get local issuer certificate, хоча причини були різними, у f2 проблема полягала в недовіреному центрі сертифікації Verify return code: 20, а у f4 у відсутності проміжного сертифіката Verify return code: 21

**Яке питання залишилося без відповіді.** Що саме не вдалося пояснити; до якого рядка виводу це стосується; що вже перевірено і що з'ясувати не вдалося:

> Чому під час першого вимірювання TLS 1.3 час DNS-пошуку становив 0.050979 с, а під час наступних приблизно 0.025 с, те ж стосується й інших параметрів, при першомі запиті вони близько в 2 рази більші за наступні

## Декларація використання ШІ

Використовувалися інструменти ШІ: так 

<!-- Якщо так, заповнити таблицю: один рядок на кожне звернення, що вплинуло на звіт. Промпт наводити дослівно. -->

| № | Інструмент | Промпт (дослівно) | Що використано у звіті і що змінено самостійно |
|---|---|---|---|
| 1 | ChatGPT | чому в TLS 1.3 сертифікат не видно в мережевому дампі, хоча OpenSSL його показує? | Проаналізовано пояснення та використано для формулювання шифрування в TLS 1.3, висновки перевірено за власними результатами |
| 2 | ChatGPT | чому TLS 1.3 встановлює з'єднання швидше за TLS 1.2, хоча ClientHello у нього більший? | Проаналізовано пояснення та використано для формулювання роботи розширення key_share та різниці в кількості RTT |
| 3 | ChatGPT | чому сертифікат сервера може мати ключ EC, але бути підписаним алгоритмом RSA-SHA256? | Проаналізовано пояснення та використано для формулювання різниці між ключем сервера та підписом сертифіката |
| 4 | ChatGPT | чому повторне використання HTTPS-з'єднання настільки скорочує час запитів? | Проаналізовано пояснення та використано для формулювання впливу TCP і TLS на час з'єднання |
