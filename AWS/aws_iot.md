# AWS IoT

[AWS IoT 開発者ドキュメント](https://docs.aws.amazon.com/ja_jp/iot/latest/developerguide/what-is-aws-iot.html)

下でいいんじゃないかなぁ。

[awsiot-handson-basic](https://awsiot-handson-dojo-basic.readthedocs.io/en/latest/index.html)

## クライアント

下記SDK配布サイトから、ターゲットとなる言語のSDKをインストールする。

[AWS IoT SDK](https://docs.aws.amazon.com/ja_jp/iot/latest/developerguide/iot-sdks.html)

pythonが楽そうなので、pythonでやる。

Install from pip

```
pip install AWSIoTPythonSDK
```

Build from source

```
git clone https://github.com/aws/aws-iot-device-sdk-python.git
cd aws-iot-device-sdk-python
python setup.py install
```

## AWS IoTチュートリアル

`学習` -> `AWS IoTに接続する` -> `デバイスの設定 開始方法`

で好みのプラットフォームを選択する。

`connect_device_package.zip`というzipファイルをダウンロードできる。
解凍すると認証に必要なファイルと、アプリケーション(`start.sh`)が入手できる。

```
$ ls
connect_device_package.zip  test_device.cert.pem     test_device.public.key
start.sh                    test_device.private.key
```

`start.sh`の中身は次のようになっている。
Root CAとSDKがなければ、インストールしている。
その後、SDK内のMQTTサンプルアプリケーションを起動して、AWS IoTにデータを送信している。

```sh
$ cat start.sh # stop script on error
set -e

# Check to see if root CA file exists, download if not
if [ ! -f ./root-CA.crt ]; then
  printf "\nDownloading AWS IoT Root CA certificate from AWS...\n"
  curl https://www.amazontrust.com/repository/AmazonRootCA1.pem > root-CA.crt
fi

# install AWS Device SDK for Python if not already installed
if [ ! -d ./aws-iot-device-sdk-python ]; then
  printf "\nInstalling AWS SDK...\n"
  git clone https://github.com/aws/aws-iot-device-sdk-python.git
  pushd aws-iot-device-sdk-python
  python setup.py install
  popd
fi

# run pub/sub sample app using certificates downloaded in package
printf "\nRunning pub/sub sample application...\n"
python aws-iot-device-sdk-python/samples/basicPubSub/basicPubSub.py -e a3ntuhifryqbzg-ats.iot.us-east-1.amazonaws.com -r root-CA.crt -c test_device.cert.pem -k test_device.private.key
```

環境によっては、python SDKをインストールするために、`sudo`が必要になる。

```
$ sudo ./start.sh
```

`aws-iot-device-sdk-python/samples/basicPubSub/basicPubSub.py`を見ると、`sdk/test/Python`というトピックをpub/sub両方するようだ。
サンプルに与えるオプションによって、pub/sub片方のみにすることもできる。

subscribe側。AWS IoTからsubscribeを受けると、`customCallback`が呼ばれる。

```py
# Connect and subscribe to AWS IoT
myAWSIoTMQTTClient.connect()
if args.mode == 'both' or args.mode == 'subscribe':
    myAWSIoTMQTTClient.subscribe(topic, 1, customCallback)
```

publish側。毎秒。`sdk/test/Python`トピックに、`{"message": "Hello World!", "sequence": 2}`というメッセージを送信している。

```py
# Publish to the same topic in a loop forever
loopCount = 0
while True:
    if args.mode == 'both' or args.mode == 'publish':
        message = {}
        message['message'] = args.message
        message['sequence'] = loopCount
        messageJson = json.dumps(message)
        myAWSIoTMQTTClient.publish(topic, messageJson, 1)
        if args.mode == 'publish':
            print('Published topic %s: %s\n' % (topic, messageJson))
        loopCount += 1
    time.sleep(1)
```

`AWS IoT` -> `テスト`

サブスクリプションに、`sdk/test/Python`を追加する。
クライアント側から、`start.sh`を実行すると、AWS IoTコンソールにsubscribeしたメッセージが表示される。

```
sdk/test/Python
2019/02/07 8:08:19
エクスポート
非表示
 {
  "message": "Hello World!",
  "sequence": 3
}
sdk/test/Python
2019/02/07 8:08:18
エクスポート
非表示
 {
  "message": "Hello World!",
  "sequence": 2
}
sdk/test/Python
2019/02/07 8:08:17
エクスポート
非表示
 {
  "message": "Hello World!",
  "sequence": 1
}
```

## AWS IoT Analytics

https://docs.aws.amazon.com/ja_jp/iotanalytics/latest/userguide/welcome.html

## AWS IoT Device Simulator

最近デプロイされたサービスでドキュメントも日本語化されていない。もう少し待つほうが良さそう。

[AWS IoT Device Simulatorを使ってみたら感動した！](https://dev.classmethod.jp/cloud/aws/aws-iot-device-simulator/)

自動車のシミュレーション

[Appendix A: Automotive Module](https://docs.aws.amazon.com/ja_jp/solutions/latest/iot-device-simulator/appendix-a.html)

## AWS Elastic Search

https://www.slideshare.net/ssuser791462/rspberry-pi-aws-iot

AWS IoTのルールを設定すると自動でインスタンスが立ち上がるらしい。

`AWS IoT` -> `ACT` -> `ルールの作成`

アクションの設定で、`Elastic Search`を有効化。

kibanaで可視化できた。OK。