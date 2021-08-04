---
author: HighPon
title: Nextcloudのhttps化
date: 2021-07-18
description: 自宅のNextcloudに証明書を導入した話です
categories: nginx
---

自宅のNextcloudをhttps化していきます

docker imageはlinuxserver/nextcloudです

Nextcloud内のコンテナに入り、letsencryptの鍵をnginx内に配置をすることでhttps化が可能です

今回は既に作成済みの鍵を配置していきます

# nginx.confを編集

/etc/nginx/nginx.confのhttpディレクティブ内に以下のことを記述します

```
http {
        server {
                listen 443 ssl;
                server_name k3s-cluster.lan.highpon.com;
                ssl_certificate /etc/letsencrypt/live/k3s-cluster.lan.highpon.com/fullchain.pem;
                ssl_certificate_key /etc/letsencrypt/live/k3s-cluster.lan.highpon.com/privkey.pem;
        }
    ....
}
```
これを記述して再起度すれば完成！と思っていたのですが、nginxを再起動する際に以下のエラーが発生しました
```
root@nextcloud-7dd7595774-b6s6j:/# sudo nginx -s reload
nginx: [emerg] SSL_CTX_use_PrivateKey("/etc/letsencrypt/live/k3s-cluster.lan.highpon.com/privkey.pem") failed (SSL: error:0B080074:x509 certificate routines:X509_check_private_key:key values mismatch)
```
どうやら秘密鍵が違うようだ。サーバ証明書と中間証明書の含まれているこのfullchain.pemを指定し、秘密鍵にはprivkey.pemを指定すればよいと考えていたが違うのだろうか

下記のコマンドでmodulusの値を確認する

```
openssl rsa -text -noout -in /key/privkey.pem -modulus
```

```
openssl x509 -text -noout -in fullchain.pem -modulus
```

値を比較してみると異なっている！

値を貼り付けているだけのはずだが、VSCodeで表示したものをコピペすると値が異なるものになっていた

その後にcatコマンドで出力したものを貼り付けると、modulusの値は一致していたため、reloadをした

しかし、/etc/nginx/nginx.confに書き込んだが、何故かうまくいかない

どのようにnginxが起動しているか調べる

```
root@nextcloud-759cd9f5d5-gmtjp:/# ps aux | grep nginx
root         397  0.0  0.0    204     4 pts/0    S+   Aug02   0:00 s6-supervise nginx
root         402  0.0  0.0   7312  5384 ?        Ss   Aug02   0:00 nginx: master process /usr/sbin/nginx -c /config/nginx/nginx.conf
```

このようになっていた

/config/nginxディレクトリに何かありそうと思い調べてみると、/config/nginx/site-confs/default直下にserverディレクティブの記述があり、ここの部分に先述の設定を書き込んで、reloadしてみる

そのようにすると、うまくhttps化されていた

linuxserver/nextcloudを使う際にはここを触ればよいようだ
