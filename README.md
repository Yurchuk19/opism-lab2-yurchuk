<!--
Бланк звіту до лабораторної роботи «TLS та сертифікати».
Скопіювати цей файл у власний публічний репозиторій звіту як README.md і заповнити.
Вивід команд вставляти текстом у блоки коду, а не знімками екрана.
Порожні поля не допускаються: якщо щось не вдалося, записати, що саме і чому.
-->

# Лабораторна робота 2. TLS та сертифікати

| Поле | Значення |
|---|---|
| ПІБ | Юрчук Дмитро Володимирович |
| Група | ЗКІ-21 |
| Ідентифікатор | `se-201-01` |
| Дата виконання | 25.11.2026 |
| Середовище | GitHub Codespaces |
| Версія OpenSSL | OpenSSL 3.0.13 |

## Параметри варіанта

<!-- Переписати з картки stand/cards/<id>.md. -->

| Параметр | Значення |
|---|---|
| Віддалений сервер | `origin.se-201-01.lab` |
| Імена для сертифіката | `www.se-201-01.lab`, `news.se-201-01.lab` |
| Ваш сервер (IP) | `10.53.1.1` |

<!-- Якщо стенд перезапускався під час роботи, вказати, після якого завдання. -->

## Завдання 1. HTTP і HTTPS у дампі трафіку

Вивід кроку 1.1:

```text
root@se-201-01:/# cd /lab/work
timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 80' > t1-http.txt &
sleep 1; curl -s http://origin.se-201-01.lab/; wait
grep -a -E 'GET /|Host:|server: web' t1-http.txt
[1] 48
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
server: web
Host: origin.se-201-01.lab
SNI:
protocol: HTTP/1.1
client: 10.53.1.1
10 packets captured
10 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 80' > t1-http.txt
00:23:56.471465 IP 10.53.1.1.44090 > 10.53.0.80.80: Flags [P.], seq 1:84, ack 1, win 502, options [nop,nop,TS val 3730225759 ecr 3688418552], length 83: HTTP: GET / HTTP/1.1
.V._....GET / HTTP/1.1
Host: origin.se-201-01.lab
server: web
Host: origin.se-201-01.lab
```

Вивід кроку 1.2:

```text
root@se-201-01:/lab/work# timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t1-https.txt &
sleep 1; curl -s --cacert $STAND_CA https://origin.se-201-01.lab/; wait
grep -a -c -E 'GET /|server: web' t1-https.txt
grep -a -o 'origin.se-201-01.lab' t1-https.txt | wc -l
[1] 54
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
server: web
Host: origin.se-201-01.lab
SNI: origin.se-201-01.lab
protocol: HTTP/2.0 TLSv1.3
client: 10.53.1.1
22 packets captured
22 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t1-https.txt
0
1
root@se-201-01:/lab/work#
```

| Що шукали в дампі | HTTP | HTTPS |
|---|---|---|
| рядок запиту `GET /` | є (`GET / HTTP/1.1`) | немає (лічильник `0` для `GET /` і `server: web`) |
| заголовок `Host` | є (`Host: origin.se-201-01.lab`) | немає |
| тіло відповіді (`server: web`) | є (`server: web`) | немає (лічильник `0`) |
| ім'я віддаленого сервера (кількість входжень) | є відкритим текстом (у `Host` запиту й у тілі відповіді; окремо не рахувалось) | 1 |

1. У якому повідомленні і в якому полі передається ім'я сервера, яке видно в дампі HTTPS?

   Ім'я сервера передається в першому повідомленні TLS, ClientHello (`Sent Record ... ClientHello, Length=331` у `t2-13.txt`), у розширенні `server_name` (SNI). У дампі `extension_type=server_name(0), length=25` поруч у шістнадцятковому вигляді видно текст `origin.se-201-01.lab`. Воно відкрите, бо ClientHello іде ще до шифрування.

2. Чому рядок `SNI:` у відповіді сервера порожній для HTTP і заповнений для HTTPS?

   SNI це частина протоколу TLS, а в звичайному HTTP шару TLS немає, і ClientHello взагалі не надсилається. Тому в HTTP (`http://origin.se-201-01.lab/`) рядок `SNI:` порожній, а в HTTPS сервер отримав розширення з ClientHello і показав `SNI: origin.se-201-01.lab`. В HTTP ім'я передається тільки в заголовку `Host`.

3. Рядок `protocol:` показує HTTP/1.1 для запиту по HTTP і HTTP/2.0 для запиту по HTTPS, хоча обидва запити виконано однією командою `curl` без додаткових ключів. Що визначило версію HTTP у другому випадку?

   Версію HTTP обирають під час TLS-рукостискання через розширення ALPN: клієнт пропонує `h2,http/1.1`, а сервер вибирає, і в `t2-13.txt` вийшло `ALPN protocol: h2`, тому HTTPS працює по HTTP/2.0 (у відповіді `protocol: HTTP/2.0 TLSv1.3`). У звичайному HTTP без TLS рукостискання немає, тож немає й місця для такого вибору, і запит іде як HTTP/1.1 (`GET / HTTP/1.1` у дампі).

## Завдання 2. Рукостискання по записах

Вивід кроку 2.1 (TLS 1.3):

```text
root@se-201-01:/lab/work# openssl s_client -connect origin.se-201-01.lab:443 -servername origin.se-201-01.lab -alpn h2,http/1.1 -CAfile $STAND_CA -trace </dev/null > t2-13.txt 2>&1
grep -a -E '^(Sent|Received) Record|Content Type|^    [A-Z][A-Za-z]+, Length' t2-13.txt
grep -a -E '^(ALPN|New,)|Verify return' t2-13.txt
Sent Record
  Content Type = Handshake (22)
    ClientHello, Length=331
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
    CertificateVerify, Length=75
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
root@se-201-01:/lab/work# openssl s_client -connect origin.se-201-01.lab:443 -servername origin.se-201-01.lab -alpn h2,http/1.1 -CAfile $STAND_CA -trace -tls1_2 </dev/null > t2-12.txt 2>&1
grep -a -E '^(Sent|Received) Record|Content Type|^    [A-Z][A-Za-z]+, Length' t2-12.txt
grep -a -E '^(ALPN|New,)|Verify return' t2-12.txt
Sent Record
  Content Type = Handshake (22)
    ClientHello, Length=226
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
    NewSessionTicket, Length=230
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
| 1 | К→С | Handshake (22) | — | ClientHello | 331 |
| 2 | С→К | Handshake (22) | — | ServerHello | 118 |
| 3 | С→К | ChangeCipherSpec (20) | — | — | — |
| 4 | С→К | ApplicationData (23) | Handshake (22) | EncryptedExtensions | 15 |
| 5 | С→К | ApplicationData (23) | Handshake (22) | Certificate | 989 |
| 6 | С→К | ApplicationData (23) | Handshake (22) | CertificateVerify | 75 |
| 7 | С→К | ApplicationData (23) | Handshake (22) | Finished | 48 |
| 8 | К→С | ChangeCipherSpec (20) | — | — | — |
| 9 | К→С | ApplicationData (23) | Handshake (22) | Finished | 48 |
| 10 | К→С | ApplicationData (23) | Alert (21) | — | — |

Вивід кроків 2.3 і 2.4:

```text
root@se-201-01:/lab/work# grep -a -A3 'extension_type=server_name' t2-13.txt | head -4
        extension_type=server_name(0), length=25
          0000 - 00 17 00 00 14 6f 72 69-67 69 6e 2e 73 65 2d   .....origin.se-
          000f - 32 30 31 2d 30 31 2e 6c-61 62                  201-01.lab
        extension_type=ec_point_formats(11), length=4
root@se-201-01:/lab/work# for v in 1.3 1.2; do timeout 6 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t2-dump-$v.txt & sleep 1; curl -s -o /dev/null --tlsv1.2 --tls-max $v --cacert $STAND_CA https://origin.se-201-01.lab/; wait; echo "TLS $v: $(grep -a -c 'OPISM Stand Intermediate' t2-dump-$v.txt)"; done
[1] 70
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
21 packets captured
21 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 6 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t2-dump-$v.txt
TLS 1.3: 0
[1] 76
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
21 packets captured
21 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 6 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t2-dump-$v.txt
TLS 1.2: 2
root@se-201-01:/lab/work#
```

| Показник | TLS 1.3 | TLS 1.2 |
|---|---|---|
| Довжина ClientHello, байт | 331 | 226 |
| Номер першого запису, тип якого спостерігач не бачить | запис №4 (`EncryptedExtensions`): з нього запис іде як `ApplicationData (23)`, хоча всередині `Inner Content Type = Handshake (22)` | такого запису немає: у виводі 2.2 немає жодного `Inner Content Type`, тип кожного запису видно відкрито |
| Кількість RTT від ClientHello до першого HTTP-запиту | 1 RTT (клієнт чекає один літ сервера від `ServerHello` до `Finished`, потім надсилає свій `Finished` разом із запитом) | 2 RTT (спочатку чекає `ServerHello ... ServerHelloDone`, потім `CCS` і `Finished` сервера) |
| Чи видно сертифікат у дампі (результат 2.4) | ні (`TLS 1.3: 0`) | так (`TLS 1.2: 2`) |
| Обраний протокол ALPN | `h2` | `h2` |

1. За якими рядками виводу 2.1 визначено, з якого запису починається шифрування? Що бачить спостерігач замість типу кожного наступного запису?

   У `t2-13.txt` це видно з рядків запису 4: `Content Type = ApplicationData (23)` і одразу під ним `Inner Content Type = Handshake (22)` (`EncryptedExtensions`), так само в записах 5–7 (`Certificate`, `CertificateVerify`, `Finished`). Тобто шифрування починається одразу після `ServerHello`. Спостерігач у мережі бачить тільки зовнішній тип `ApplicationData (23)` і не знає, що всередині, а `Inner` показує лише openssl, бо він сам розшифровує. Це підтверджує дамп: у TLS 1.3 назву `OPISM Stand Intermediate` у сертифікаті знайдено 0 разів, а в TLS 1.2 2 рази.

2. Чому ClientHello у TLS 1.3 довший, ніж у TLS 1.2? Назвіть розширення, яке за рахунок цього скорочує рукостискання на один RTT.

   У TLS 1.3 `ClientHello` довший (331 проти 226 байт), бо клієнт уже в ньому надсилає свою частину для обчислення ключа, це розширення `key_share`. Через нього сервер може порахувати ключі одразу після `ClientHello` і відповісти всім одним літом, тому економиться один RTT, а в TLS 1.2 для обміну ключами потрібен окремий `ClientKeyExchange` після відповіді сервера.

3. Довжина розширення `server_name` у виводі 2.3 становить 17 байт для імені з 12 символів. З яких частин складається це значення і яким воно буде для вашого імені?

   `length=25` складається з 2 байтів довжини списку імен (`00 17`, це 23), 1 байта типу імені (`00`, host_name), 2 байтів довжини імені (`00 14`, це 20) і самого імені `origin.se-201-01.lab` (20 символів): 2 + 1 + 2 + 20 = 25. Службових байтів завжди 5, тому для мого імені виходить 25, а для іншого імені буде 5 плюс його довжина.

4. Навіщо в TLS 1.3 передається запис ChangeCipherSpec, якщо в рукостисканні він нічого не змінює?

   У TLS 1.3 `ChangeCipherSpec` нічого не змінює (ключі змінюються без нього), він залишений для сумісності. Старе мережеве обладнання (фаєрволи, проксі) очікує послідовність записів як у TLS 1.2 і могло б розірвати з'єднання, якби `CCS` зник, тому в `t2-13.txt` він є і від сервера (запис 3), і від клієнта (запис 8).

## Завдання 3. Сертифікат для власного сервера

Вивід кроків 3.1 і 3.2:

```text
root@se-201-01:/lab/work# acme-root
openssl x509 -in /lab/tls/acme-root.crt -noout -subject -dates
збережено /lab/tls/acme-root.crt
subject=CN = Pebble Root CA 1286d9
notBefore=Nov 25 00:21:26 2026 GMT
notAfter=Nov 25 00:21:26 2056 GMT
root@se-201-01:/lab/work# curl -sS --cacert /lab/tls/acme-root.crt https://www.se-201-01.lab/
curl: (60) SSL certificate problem: self-signed certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```

Вивід кроку 3.3 (останні рядки `lego` і рядки журналу nginx):

```text
root@se-201-01:/lab/work# lego --server $ACME_DIR --email admin@se-201-01.lab --accept-tos --path /lab/work/lego --http --http.webroot /var/www/acme --domains www.se-201-01.lab --domains news.se-201-01.lab run --no-bundle
grep acme-challenge /var/log/nginx/access.log
2026/11/25 00:27:18 No key found for account admin@se-201-01.lab. Generating a P256 key.
2026/11/25 00:27:18 Saved key to /lab/work/lego/accounts/ca.lab_14000/admin@se-201-01.lab/keys/admin@se-201-01.lab.key
2026/11/25 00:27:18 [INFO] acme: Registering account for admin@se-201-01.lab
!!!! HEADS UP !!!!

Your account credentials have been saved in your Let's Encrypt
configuration directory at "/lab/work/lego/accounts".

You should make a secure backup of this folder now. This
configuration directory will also contain certificates and
private keys obtained from Let's Encrypt so making regular
backups of this folder is ideal.
2026/11/25 00:27:18 [INFO] [www.se-201-01.lab, news.se-201-01.lab] acme: Obtaining SAN certificate
2026/11/25 00:27:19 [INFO] [www.se-201-01.lab] AuthURL: https://ca.lab:14000/authZ/blPa3Mhj-2U37ax-i5-uYIXUaMdwM5246aXfqmOvqkw
2026/11/25 00:27:19 [INFO] [news.se-201-01.lab] AuthURL: https://ca.lab:14000/authZ/lF7AoKkKndSCpfrhQ-fdJysMnb0c33flbCuDDNB7TC0
2026/11/25 00:27:19 [INFO] [www.se-201-01.lab] acme: Could not find solver for: tls-alpn-01
2026/11/25 00:27:19 [INFO] [www.se-201-01.lab] acme: use http-01 solver
2026/11/25 00:27:19 [INFO] [news.se-201-01.lab] acme: Could not find solver for: tls-alpn-01
2026/11/25 00:27:19 [INFO] [news.se-201-01.lab] acme: use http-01 solver
2026/11/25 00:27:19 [INFO] [www.se-201-01.lab] acme: Trying to solve HTTP-01
2026/11/25 00:27:22 [INFO] [www.se-201-01.lab] The server validated our request
2026/11/25 00:27:22 [INFO] [news.se-201-01.lab] acme: Trying to solve HTTP-01
2026/11/25 00:27:28 [INFO] [news.se-201-01.lab] The server validated our request
2026/11/25 00:27:28 [INFO] [www.se-201-01.lab, news.se-201-01.lab] acme: Validations succeeded; requesting certificates
2026/11/25 00:27:28 [INFO] Wait for certificate [timeout: 30s, interval: 500ms]
2026/11/25 00:27:29 [INFO] [www.se-201-01.lab] Server responded with a certificate.
10.53.0.10 - - [25/Nov/2026:00:27:19 +0000] "GET /.well-known/acme-challenge/uMODPDMjd162INqrqX9ixlSOpwmE2DR4wEWLePLd1k4 HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [25/Nov/2026:00:27:19 +0000] "GET /.well-known/acme-challenge/uMODPDMjd162INqrqX9ixlSOpwmE2DR4wEWLePLd1k4 HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [25/Nov/2026:00:27:19 +0000] "GET /.well-known/acme-challenge/uMODPDMjd162INqrqX9ixlSOpwmE2DR4wEWLePLd1k4 HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [25/Nov/2026:00:27:22 +0000] "GET /.well-known/acme-challenge/i9Wn5M2mXw5YQ2ReA4C6Qd5Gv0Q6nv5Jftv0Pr3ksfA HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [25/Nov/2026:00:27:22 +0000] "GET /.well-known/acme-challenge/i9Wn5M2mXw5YQ2ReA4C6Qd5Gv0Q6nv5Jftv0Pr3ksfA HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [25/Nov/2026:00:27:22 +0000] "GET /.well-known/acme-challenge/i9Wn5M2mXw5YQ2ReA4C6Qd5Gv0Q6nv5Jftv0Pr3ksfA HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
```

Вивід кроку 3.4:

```text
root@se-201-01:/lab/work# cd /lab/work/lego/certificates
openssl x509 -in www.se-201-01.lab.crt -noout -subject -issuer -dates -serial -ext subjectAltName
openssl x509 -in www.se-201-01.lab.issuer.crt -noout -subject -issuer
subject=CN = www.se-201-01.lab
issuer=CN = Pebble Intermediate CA 288376
notBefore=Nov 25 00:27:28 2026 GMT
notAfter=Nov 25 00:27:27 2031 GMT
serial=758D4B377B20977B
X509v3 Subject Alternative Name:
    DNS:www.se-201-01.lab, DNS:news.se-201-01.lab
subject=CN = Pebble Intermediate CA 288376
issuer=CN = Pebble Root CA 1286d9
root@se-201-01:/lab/work/lego/certificates#
```

Вивід кроку 3.5:

```text
root@se-201-01:/lab/work/lego/certificates# cp www.se-201-01.lab.crt /lab/tls/server.crt
cp www.se-201-01.lab.key /lab/tls/server.key
web-reload
curl -sS --cacert /lab/tls/acme-root.crt https://www.se-201-01.lab/
openssl s_client -connect www.se-201-01.lab:443 -servername www.se-201-01.lab -CAfile /lab/tls/acme-root.crt </dev/null 2>/dev/null | sed -n '/^Certificate chain/,/^---/p;/^Verify return code/p'
nginx: the configuration file /lab/nginx.conf syntax is ok
nginx: configuration file /lab/nginx.conf test is successful
2026/11/25 00:28:29 [notice] 98#98: signal process started
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
Certificate chain
 0 s:CN = www.se-201-01.lab
   i:CN = Pebble Intermediate CA 288376
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 25 00:27:28 2026 GMT; NotAfter: Nov 25 00:27:27 2031 GMT
---
Verify return code: 21 (unable to verify the first certificate)
```

Вивід кроку 3.6:

```text
root@se-201-01:/lab/work/lego/certificates# cat www.se-201-01.lab.crt www.se-201-01.lab.issuer.crt > /lab/tls/server.crt
web-reload
curl -sS --cacert /lab/tls/acme-root.crt https://www.se-201-01.lab/
openssl s_client -connect www.se-201-01.lab:443 -servername www.se-201-01.lab -CAfile /lab/tls/acme-root.crt </dev/null 2>/dev/null | sed -n '/^Certificate chain/,/^---/p;/^Verify return code/p'
curl -sS --cacert /lab/tls/acme-root.crt https://news.se-201-01.lab/
nginx: the configuration file /lab/nginx.conf syntax is ok
nginx: configuration file /lab/nginx.conf test is successful
2026/11/25 00:28:36 [notice] 106#106: signal process started
server: se-201-01 (https)
Host: www.se-201-01.lab
SNI: www.se-201-01.lab
protocol: TLSv1.3
Certificate chain
 0 s:CN = www.se-201-01.lab
   i:CN = Pebble Intermediate CA 288376
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 25 00:27:28 2026 GMT; NotAfter: Nov 25 00:27:27 2031 GMT
 1 s:CN = Pebble Intermediate CA 288376
   i:CN = Pebble Root CA 1286d9
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 25 00:21:26 2026 GMT; NotAfter: Nov 25 00:21:26 2056 GMT
---
Verify return code: 0 (ok)
server: se-201-01 (https)
Host: news.se-201-01.lab
SNI: news.se-201-01.lab
protocol: TLSv1.3
root@se-201-01:/lab/work/lego/certificates#
```

| Параметр | Значення |
|---|---|
| Видавець сертифіката сервера | `CN = Pebble Intermediate CA 288376` |
| Видавець сертифіката видавця | `CN = Pebble Root CA 1286d9` |
| Імена в SAN | `DNS:www.se-201-01.lab`, `DNS:news.se-201-01.lab` |
| Строк дії (від – до) | Nov 25 00:27:28 2026 GMT – Nov 25 00:27:27 2031 GMT |
| Тип ключа сервера і алгоритм підпису | `id-ecPublicKey, 256 (bit)`; `sigalg: RSA-SHA256` |
| Кількість сертифікатів, які надсилає сервер, у кроці 3.5 і в кроці 3.6 | 1 (лише `0 s:CN = www.se-201-01.lab`) і 2 (додано `1 s:CN = Pebble Intermediate CA 288376`) |

1. Хто і як перевірив, що ви контролюєте обидва імена? Вкажіть рядки виводу, які це підтверджують.

   Перевірку зробив ACME-сервер `ca.lab` (`10.53.0.10`, це видно в привітанні стенда). Lego обрав метод `acme: use http-01 solver`, поклав файл-токен у вебкаталог мого nginx, а ACME-сервер сам зайшов на мій сервер і забрав його. У журналі nginx це рядки `10.53.0.10 ... "GET /.well-known/acme-challenge/... HTTP/1.1" 200 ... "LetsEncrypt-Pebble-VA"`, а в lego `The server validated our request` окремо для `www` і для `news`. Тобто контроль над іменем доведено тим, що я зміг віддати потрібний токен за цим іменем.

2. Чому в кроці 3.5 перевірка не пройшла, хоча корінь центру сертифікації вказано в `--cacert`, а сертифікат виданий правильно?

   У 3.5 я скопіював на сервер тільки сертифікат `www`, тому в ланцюжку був лише `0 s:CN = www.se-201-01.lab`. Корінь клієнт знає (`acme-root.crt`), але сертифікат `www` підписав проміжний `Pebble Intermediate CA 288376`, а його сервер не надіслав, тому клієнт не зміг дійти від сертифіката до кореня і видав `unable to get local issuer certificate`, код 21. У 3.6 я склеїв сертифікат і `issuer.crt` в один файл, тому в ланцюжку стало два сертифікати і вийшло `Verify return code: 0 (ok)`.

3. У виводі ланцюжка для сертифіката сервера вказано `id-ecPublicKey` і `RSA-SHA256`. Що означає кожне з цих значень і чому вони не суперечать одне одному?

   `id-ecPublicKey, 256 (bit)` це тип ключа самого сервера (еліптична крива), саме цей ключ сервер використовує в рукостисканні. `RSA-SHA256` це те, чим видавець підписав сертифікат сервера. Суперечності немає, бо це ключі двох різних учасників: у проміжного центру ключ `rsaEncryption, 2048 (bit)`, тому він і підписує по RSA, а ключ сервера в його сертифікаті може бути іншого типу.

4. Що станеться з перевіркою вашого сервера після перезапуску стенда і чому? Що в реальній інфраструктурі відповідає цій ситуації?

   Після перезапуску стенда корінь і проміжний Pebble створяться заново, з новими ключами. Мій старий сертифікат підписаний старим проміжним, тому новий корінь його не визнає, і перевірка не пройде, а збережений `acme-root.crt` теж буде старий. Треба заново зберегти корінь (`acme-root`) і перевипустити сертифікат через lego. У реальному житті це схоже на ситуацію, коли центр сертифікації змінив корінь або корінь відкликали чи перестали йому довіряти: всі видані сертифікати треба перевипускати, а клієнти мають оновити список довірених коренів.

## Завдання 4. Діагностика несправностей

Вивід кроку 4.1:

```text
root@se-201-01:/lab/work/lego/certificates# for i in 1 2 3 4 5; do n=f$i.se-201-01.lab; echo "=== $n"; curl -sS --cacert $STAND_CA https://$n/ 2>&1 | head -1; openssl s_client -connect $n:443 -servername $n -CAfile $STAND_CA </dev/null 2>/dev/null | sed -n '/^Certificate chain/,/^---/p;/^Verify return code/p' | grep -v PKEY; done
=== f1.se-201-01.lab
curl: (60) SSL: no alternative certificate subject name matches target host name 'f1.se-201-01.lab'
Certificate chain
 0 s:CN = old.se-201-01.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 25 00:21:23 2026 GMT; NotAfter: Feb 23 00:21:23 2027 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 25 00:21:23 2026 GMT; NotAfter: Nov 24 00:21:23 2031 GMT
---
Verify return code: 0 (ok)
=== f2.se-201-01.lab
curl: (60) SSL certificate problem: unable to get local issuer certificate
Certificate chain
 0 s:CN = f2.se-201-01.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 25 00:21:23 2026 GMT; NotAfter: Feb 23 00:21:23 2027 GMT
---
Verify return code: 21 (unable to verify the first certificate)
=== f3.se-201-01.lab
server: web, f3
Certificate chain
 0 s:CN = f3.se-201-01.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 25 00:21:23 2026 GMT; NotAfter: Feb 23 00:21:23 2027 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 25 00:21:23 2026 GMT; NotAfter: Nov 24 00:21:23 2031 GMT
---
Verify return code: 0 (ok)
=== f4.se-201-01.lab
curl: (60) SSL certificate problem: self-signed certificate
Certificate chain
 0 s:CN = f4.se-201-01.lab
   i:CN = f4.se-201-01.lab
   v:NotBefore: Nov 25 00:21:23 2026 GMT; NotAfter: Feb 23 00:21:23 2027 GMT
---
Verify return code: 18 (self-signed certificate)
=== f5.se-201-01.lab
curl: (60) SSL certificate problem: certificate has expired
Certificate chain
 0 s:CN = f5.se-201-01.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: May  9 00:21:23 2026 GMT; NotAfter: Nov  5 00:21:23 2026 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 25 00:21:23 2026 GMT; NotAfter: Nov 24 00:21:23 2031 GMT
---
Verify return code: 10 (certificate has expired)
root@se-201-01:/lab/work/lego/certificates#
```

| Сервер | Помилка `curl` | `Verify return code` | Яка перевірка не пройшла | Рядок виводу, що це доводить | Що має змінити власник сервера |
|---|---|---|---|---|---|
| f1 | `no alternative certificate subject name matches target host name 'f1.se-201-01.lab'` | `0 (ok)` | ім'я | `0 s:CN = old.se-201-01.lab`, curl: `no alternative certificate subject name matches target host name 'f1.se-201-01.lab'` | видати сертифікат на ім'я `f1.se-201-01.lab` (воно має бути в сертифікаті) і поставити його на сервер |
| f2 | `unable to get local issuer certificate` | `21 (unable to verify the first certificate)` | ланцюжок (проміжний) | у ланцюжку тільки `0 s:CN = f2.se-201-01.lab`, рядка `1` немає, `Verify return code: 21` | додати проміжний сертифікат у файл сервера разом із сертифікатом сервера |
| f3 | помилки немає (`server: web, f3`) | `0 (ok)` | жодна (справний) | `server: web, f3`, два сертифікати в ланцюжку, `Verify return code: 0 (ok)` | нічого не треба змінювати |
| f4 | `self-signed certificate` | `18 (self-signed certificate)` | самопідписаний | `s:CN = f4.se-201-01.lab` і `i:CN = f4.se-201-01.lab`, `Verify return code: 18` | отримати сертифікат від довіреного центру (наприклад, через ACME), а не підписувати самому |
| f5 | `certificate has expired` | `10 (certificate has expired)` | строк дії | `NotAfter: Nov  5 00:21:23 2026 GMT`, а зараз 25 листопада, `Verify return code: 10` | перевипустити сертифікат з новим строком дії |

1. Для сервера з помилкою `unable to get local issuer certificate` визначте, яка це несправність: сервер не надсилає проміжний сертифікат чи сертифікат виданий центром, якому клієнт не довіряє. Чим ці дві несправності відрізняються у виводі `openssl s_client`?

   У f2 видавець `i:` це `OPISM Stand Intermediate CA`, тобто центр стенда, якому клієнт довіряє через `$STAND_CA`, але проміжного сертифіката в ланцюжку немає, тому клієнт не може дійти від сертифіката сервера до кореня. Тут сервер просто не надіслав проміжний, і це лікується додаванням його у файл сервера. Якби сертифікат видав чужий центр, у `i:` стояла б інша назва, якої клієнт не знає, і навіть з додаванням проміжного ланцюжок не дійшов би до `$STAND_CA`, тобто помилка була б не в тому, що щось не надіслано.

2. Якщо серед ваших серверів є сервер з невідповідним ім'ям, чому `openssl s_client` повертає для нього `Verify return code: 0 (ok)`, а `curl` повідомляє про помилку? Якщо такого сервера немає, дайте відповідь для загального випадку.

   `openssl s_client` за замовчуванням перевіряє тільки ланцюжок сертифікатів, а ім'я в сертифікаті з тим, до якого підключились, не звіряє. У f1 ланцюжок до `$STAND_CA` нормальний, тому вийшло `Verify return code: 0 (ok)`. Curl додатково звіряє ім'я, а в сертифікаті там `old.se-201-01.lab`, тому він видає помилку.

3. Які з п'яти серверів відповіли б на запит `curl -k`? Що при цьому втрачається порівняно з `--cacert`?

   Відповіли б усі п'ять серверів (f1–f5), бо `-k` вимикає перевірку сертифіката повністю, і ім'я, строк, ланцюжок та самопідписаність уже нікого не зупиняють. Шифрування лишається, але зникає впевненість, з ким саме я говорю: підміненому серверу я б теж повірив, тому `-k` не захищає від атаки посередника, на відміну від `--cacert`.

## Завдання 5. Ціна нового з'єднання

**Прогноз** (записати до вимірювання). Скільки RTT мине до завершення кожного етапу:

| Етап | TLS 1.3, нове з'єднання | TLS 1.2, нове з'єднання | Повторне використання з'єднання |
|---|---|---|---|
| `time_connect` | 1 RTT | 1 RTT | 0 |
| `time_appconnect` | 2 RTT | 3 RTT | 0 |
| `time_starttransfer` | 3 RTT | 4 RTT | 1 RTT |

*(Прогноз записано 25.11.2026 о 01:30, до кроку 5.2; час рахується від початку TCP-з'єднання, без DNS; формулювання підготовлено з допомогою ШІ, див. декларацію.)*

Вивід кроків 5.2 і 5.3:

```text
root@se-201-01:/lab/work/lego/certificates# F='%{time_namelookup} %{time_connect} %{time_appconnect} %{time_starttransfer} %{time_total} %{http_version} %{num_connects}\n'
U=https://origin.se-201-01.lab/
for i in 1 2 3; do curl -s -o /dev/null --cacert $STAND_CA -w "$F" $U; done
for i in 1 2 3; do curl -s -o /dev/null --tls-max 1.2 --cacert $STAND_CA -w "$F" $U; done
0.071026 0.106188 0.144517 0.179896 0.179961 2 1
0.035583 0.070710 0.109074 0.144386 0.144441 2 1
0.035491 0.070612 0.108973 0.144289 0.144349 2 1
0.035451 0.070598 0.144146 0.179476 0.179565 2 1
0.035571 0.070808 0.145194 0.180618 0.180711 2 1
0.035723 0.070844 0.144446 0.179790 0.179908 2 1
root@se-201-01:/lab/work/lego/certificates# curl -s --cacert $STAND_CA -w "$F" -o /dev/null $U -o /dev/null $U -o /dev/null $U
0.035493 0.070738 0.111077 0.147737 0.147784 2 1
0.000022 0.000000 0.000000 0.035242 0.035289 2 0
0.000018 0.000000 0.000000 0.035352 0.035395 2 0
root@se-201-01:/lab/work/lego/certificates#
```

<!-- Значення в мілісекундах. Для нових з'єднань — медіана трьох вимірювань. -->

| Запит | `namelookup` | `connect` | `appconnect` | `starttransfer` | `total` | `num_connects` |
|---|---|---|---|---|---|---|
| TLS 1.3, нове | 35.6 | 70.7 | 109.1 | 144.4 | 144.4 | 1 |
| TLS 1.2, нове | 35.6 | 70.8 | 144.4 | 179.8 | 179.9 | 1 |
| 5.3, перший запит | 35.5 | 70.7 | 111.1 | 147.7 | 147.8 | 1 |
| 5.3, другий запит | 0.0 | 0 | 0 | 35.2 | 35.3 | 0 |
| 5.3, третій запит | 0.0 | 0 | 0 | 35.4 | 35.4 | 0 |

RTT, визначений вимірюванням, і з якого значення його визначено: **≈ 35 мс** = `time_connect` − `time_namelookup` = 70.7 − 35.6 = 35.1 мс (медіана TLS 1.3); те саме дає TLS 1.2: 70.8 − 35.6 = 35.2 мс.

1. Виразіть кожне виміряне значення через RTT. Де вимірювання відрізняється від цілого числа RTT і чим це пояснюється?

   Ділю значення на 35 мс:

   | | namelookup | connect | appconnect | starttransfer |
   |---|---|---|---|---|
   | TLS 1.3 | 35,6 / 35 ≈ 1,02 RTT | 70,7 / 35 ≈ 2,02 RTT | 109,1 / 35 ≈ 3,12 RTT | 144,4 / 35 ≈ 4,13 RTT |
   | TLS 1.2 | 35,6 / 35 ≈ 1,02 RTT | 70,8 / 35 ≈ 2,02 RTT | 144,4 / 35 ≈ 4,13 RTT | 179,8 / 35 ≈ 5,14 RTT |

   Виходить, що пошук імені це 1 RTT, TCP ще 1 RTT, TLS 1.3 ще 1 RTT, а TLS 1.2 ще 2 RTT, і на сам запит ще 1 RTT. Не цілими числа стають через зайві 2–4 мс, які з'являються після етапів TLS (наприклад, 109,1 замість 105, а 144,4 замість 140 у TLS 1.2). Це не мережа, а робота самого комп'ютера: обчислення ключів, шифрування і перевірка підпису сертифіката.

2. Чому `time_namelookup` не близький до нуля, хоча резолвер уже має ім'я в кеші? За яких умов він був би приблизно вдвічі більшим?

   Резолвер `10.53.0.53` це окремий вузол, а затримка netem діє на весь вихідний трафік мого контейнера, тому навіть відповідь з кешу резолвера приходить через 1 RTT (запит пішов і відповідь повернулась), тобто ≈ 35 мс. Вдвічі більше (≈ 70) було б, коли імені немає в кеші резолвера і він сам іде за ним до авторитативного сервера, тоді додається ще один RTT. Саме так було в найпершому вимірі TLS 1.3: `namelookup = 0.071026`, бо кеш тоді ще був порожній.

3. Сервіс виконує 200 послідовних запитів до цього сервера. Розрахуйте, скільки часу займе лише очікування мережі, якщо кожен запит встановлює нове з'єднання TLS 1.3, нове з'єднання TLS 1.2 і якщо всі запити йдуть через одне з'єднання.

   Рахую за виміряним часом одного запиту:

   * (а) TLS 1.3, кожного разу нове з'єднання: 200 × 144 мс = 28 800 мс ≈ 28,8 с (це 4 RTT × 200 = 800 RTT);
   * (б) TLS 1.2, кожного разу нове з'єднання: 200 × 180 мс = 36 000 мс = 36 с (це 5 RTT × 200 = 1000 RTT);
   * (в) одне з'єднання для всіх: 148 + 199 × 35 = 148 + 6965 = 7113 мс ≈ 7,1 с (це 4 RTT + 199 × 1 RTT = 203 RTT).

   Отже, одне з'єднання приблизно у 4 рази швидше за нові з'єднання щоразу, а перехід з TLS 1.2 на TLS 1.3 економить ще близько 7 с.

## Висновки

**Що виявилося несподіваним.** Результат, який не збігся з очікуванням, і рядок виводу, де це видно:

Несподіваним для мене було те, що в f1 `openssl s_client` показав `Verify return code: 0 (ok)`, хоча сертифікат виданий на інше ім'я (`0 s:CN = old.se-201-01.lab`), а помилку `no alternative certificate subject name matches target host name 'f1.se-201-01.lab'` побачив тільки `curl`. Я думав, що обидві програми перевіряють сертифікат однаково, а виявилось, що `openssl` перевіряє лише ланцюжок. Ще мене здивувало, що в 3.5 сертифікат `www` був правильний і корінь я вказав, але перевірка не пройшла (`unable to get local issuer certificate`, код 21), бо сервер не надіслав проміжний сертифікат, і в ланцюжку був тільки рядок `0 s:CN = www.se-201-01.lab`.

**Яке питання залишилося без відповіді.** Що саме не вдалося пояснити; до якого рядка виводу це стосується; що вже перевірено і що з'ясувати не вдалося:

Без відповіді лишилося питання, чому ACME-сервер у журналі nginx (3.3) забрав кожен токен тричі: о 00:27:19 є три однакові рядки `GET /.well-known/acme-challenge/uMODP...` від `LetsEncrypt-Pebble-VA`, і так само три рядки для `news` о 00:27:22. Я перевірив журнал nginx, і там усі три запити мають код 200 від одного й того ж `10.53.0.10`, але чому їх три, а не один, я пояснити не можу, бо не бачив налаштувань ACME-сервера.

## Декларація використання ШІ

Використовувалися інструменти ШІ: так

Примітка до таблиці. Промпти наведено дослівно. Повідомлення, які містили лише вставлений вивід терміналу або скріншот, позначено як «[вивід терміналу …]» / «[скріншот …]»: самі виводи повністю наведено у звіті. Повідомлення з текстами моїх відповідей позначено як «[мій текст …]»: ці тексти наведено у звіті дослівно, з виправленнями ШІ, вказаними в останній колонці.


| № | Інструмент | Промпт (дослівно) | Що використано у звіті і що змінено самостійно |
|---|---|---|---|
| 1 | Claude Opus 5.5 | [виводи терміналу завдань 1–5] | ШІ  після кожного завдання пояснював результат і давав команди наступного |
