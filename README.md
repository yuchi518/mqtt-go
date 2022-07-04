

### 啟動 mosquitto server

mosquitto 與 mosquitto-no-cert 分別為使用與不使用 SSL 連線所需的設定目錄。

```shell
# 使用 SSL 的啟動方法
docker run -it --name mosquitto -p 8883:8883 -v ~/Dropbox/RogersAI/Develop/mqtt-go/mosquitto:/mosquitto/ eclipse-mosquitto:2.0.14-openssl
# 不使用 SSL 的啟動方法
docker run -it --name mosquitto-no-cert -p 1883:1883 -v ~/Dropbox/RogersAI/Develop/mqtt-go/mosquitto-no-cert:/mosquitto/ eclipse-mosquitto:2.0.14
```
注意要將 ~/Dropbox/RogersAI/Develop/mqtt-go 需要修改到正確的位置，而依據需求(使用或不使用ssl)將專案內的/mosquitto或者/mosquitto-no-cert對應到容器內的/mosquitto位置。


### 產生 cert 相關檔案
當 mosquitto 使用加密連線時，需要建立三個部分的簽章：CA, server, client。
mosquitto-eclipse 搭配 paho go cmd裡的sample code "ssl"

* [mosquitto config 設定](https://mosquitto.org/man/mosquitto-conf-5.html)
* [mosquitto tls 產生方法](https://mosquitto.org/man/mosquitto-tls-7.html)

###### 注意事項

在 mosquitto tls 文件一開始有指出，產生三個部分的簽章過程中所使用的參數一定要有所差異。
> It is important to use different certificate subject parameters for your CA, server and clients.
> If the certificates appear identical, even though generated separately,
> the broker/client will not be able to distinguish between them and you will experience difficult to diagnose errors.

1. 確定使用 openssl 1.1 而不是其他版本或者LibreSSL
    ```shell
    brew install openssl@1.1
    # mac m1 機型的位置, 設定alias
    /opt/homebrew/Cellar/openssl@1.1/1.1.1p/bin/openssl version
    alias openssl11='/opt/homebrew/Cellar/openssl@1.1/1.1.1p/bin/openssl' 
    # mac intel 機型的位置, 設定alias
    /usr/local/Cellar/openssl@1.1/1.1.1p/bin/openssl version
    alias openssl11='/usr/local/Cellar/openssl@1.1/1.1.1p/bin/openssl'
    ```
2. 產生 CA (我方自己行建立與保管)
    ```shell
    # openssl req -new -x509 -days <duration> -extensions v3_ca -keyout ca.key -out ca.crt
    openssl11 req -new -x509 -days 3650 -extensions v3_ca -keyout ca.key -out ca.crt
    ```
3. 產生 server 憑證 (我方自行建立與保管)
    ```shell
    # == server
    # openssl genrsa -des3 -out server.key 2048
    # openssl genrsa -out server.key 2048
    openssl11 genrsa -des3 -out server.key 2048
    # openssl req -out server.csr -key server.key -new
    openssl11 req -out server.csr -key server.key -new
    # openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days <duration>
    openssl11 x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 3650
    ```

4. 產生 client 憑證 (原則上針對不同client，我方要協助產生不同的憑證提供給對方)
    ```shell
    # openssl genrsa -des3 -out client.key 2048
    openssl11 genrsa -des3 -out client.key 2048
    # openssl req -out client.csr -key client.key -new
    openssl11 req -out client.csr -key client.key -new
    # openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt -days <duration>
    openssl11 x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt -days 3650
    ```

5. 將憑證轉為 pem (*.key, *.crt to *.pem)
    ```shell
    # openssl x509 -in cert.crt -out cert.pem
    # openssl x509 -in cert.cer -out cert.pem
    # openssl x509 -in cert.der -out cert.pem
    openssl11 x509 -in ca.crt -out ca-cert.pem
    openssl11 x509 -in client.crt -out client-cert.pem
    openssl11 rsa -in client.key -text > client-key.pem
    openssl11 x509 -in server.crt -out server-cert.pem
    openssl11 rsa -in server.key -text > server-key.pem
    ```
