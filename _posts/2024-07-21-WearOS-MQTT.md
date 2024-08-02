---
title: Wear OS 에서 MQTT 통신하기
categories:
- Wear OS
- MQTT
feature_text: "Wear OS 애플리케이션에 MQTT를 적용해보자!"
feature_image: "https://picsum.photos/2560/600?image=872"
author: NurungjiBurger
---
### 들어가며

---

![WearOS](/assets/images/2024-07-21-WearOS-MQTT/WearOS.png)

스마트 물류 자동화 프로젝트 물?류에서 Wear OS기반으로 스마트워치에 MQTT 통신으로 알람 및 작업 목록을 보여주는 앱을 개발하고 있습니다.

Wear OS에 MQTT를 적용하는 글이 별로 없어 진행하면서 겪었던 과정을 소개하고자 합니다.

<!-- more -->

# MQTT란 ?

---

MQTT(Message Queuing Telemetry Transport)는 경량의 메시지 프로토콜로, 제한된 대역폭과 낮은 전력 소비를 요구하는 환경에서 기기 간 데이터를 전송하기 위해 설계되었습니다. 최소한의 전략과 패킷량으로 통신하기 때문에 사물 인터넷(IoT)와 모바일 앱 등의 통신에 매우 적합한 프토콜입니다.

MQTT는 클라이언트-서버의 구조로 통신이 이루어지는 형태가 아니라 Broker, Publisher, Subscriber 구조로 통신이 이루어집니다.

**Publisher**는 Topic을 발행하고, **Subscriber**는 Topic을 구독합니다.

**Broker**는 이들을 중계하며 하나의 Topic에 대해 여러 Subscriber가 있을 수 있어 1:N 통신에도 유용합니다.

![MQTT](/assets/images/2024-07-21-WearOS-MQTT/MQTT.png)

# MQTT 사용하기

---

MQTT 프로토콜을 구현하는 브로커들은 Mosquitto, RabbitMQ, HiveMQ등 다양한 종류가 있지만 그 중 저희는 **Mosquitto**를 직접 로컬에 설치하여 사용하였습니다.

# Wear OS에서 MQTT 사용하기

---

현재 물?류의 앱은 안드로이드 스튜디오로 개발 중이며 앱에서는 통신 프로토콜로 MQTT를 채택하였습니다.

먼저, 안드로이드 스튜디오에서 MQTT를 사용하기 위해서는 build.gradle 파일에 모듈부터 설정해서 사용할 준비를 해야합니다.

```kotlin
    implementation(libs.org.eclipse.paho.client.mqttv3)
    implementation(libs.org.eclipse.paho.android.service)
```

첫 번째 줄은 MQTT프로토콜을 구현하는 클라이언트 라이브러리를 제공하는 오픈 소스 프로젝트 Eclipse Paho MQTT 에서 MQTT v3.1 프로토콜을 자바 환경에서 MQTT 메시징 기능을 구현하기 위해 사용하였고, 두 번째 줄은 Paho의 안드로이드 버전으로 안드로이드를 기반으로하는 Wear OS에서 MQTT 클라이언트를 구현, 실행할 수 있도록 해줍니다.

다음은 구현 코드입니다.

### 객체 생성 및 초기화

```kotlin
 		// MQTT를 위한 클라이언트 객체 생성
    private lateinit var mqttClient: MqttClient
    // 사용할 주제
    private val mqttTopic = "MyTopic"

    init {
         // 해당 IP로 연결
         mqttClient = MqttClient(${myBrokerIpAddress}, MqttClient.generateClientId(), MemoryPersistence())
    }
```

위 두 변수는 각각 MQTT를 위한 클라이언트 객체와 통신할 주제인 Topic입니다.

선언된 객체들의 정보를 통해 초기화를 진행하며 생성자는 다음과 같은 매개변수를 필요로 합니다. 

**MqttClient( ${ Broker IP }, ${ Client ID }, ${ Persistence } )**

- **Broker IP**
    - 연결하려는 브로커의 URL이 포함되어야 합니다.
- **Client ID**
    - MQTT 클라이언트의 고유 식별자로 해당 클라이언트를 구분하기 위해 사용됩니다.
- **Persistence**
    - 클라이언트의 상태를 저장할 방법을 의미하며 위 코드는 메시지와 클라이언트 상태가 저장되도록 설정되었습니다.

### 연결 및 해제

```kotlin
    // MQTT 브로커에 연결하기
    fun connectToMQTTBroker() {
        try {
            // 연결 설정
            mqttClient.connect()
            // 콜백 함수 설정
            mqttClient.setCallback(object : MqttCallback {
                // 연결이 끊어졌을때 무엇을 해야할까 ?
                override fun connectionLost(cause: Throwable?) {
                    println("연결이 끊어졌습니다.")
                }

                // 메시지가 왔을때 무엇을 해야할까 ?
                override fun messageArrived(topic: String?, message: MqttMessage?) {
                    println("메시지가 도착했습니다.")
                }

                // 메시지가 전송 됐을때 무엇을 해야할까 ?
                override fun deliveryComplete(token: IMqttDeliveryToken?) {
                    println("메시지가 전송 되었습니다.")
                }
            })
        } catch (e: MqttException) {
            println("MQTT 브로커 연결에 실패했습니다.")
        }
    }

    // 연결 해제
    fun disconnect() {
        try {
            mqttClient.disconnect()
        } catch (e: MqttException) {
            e.printStackTrace()
        }
    }
```

MQTT객체의 connect와 disconnect를 통해서 연결 및 해제를 할 수 있습니다.

저는 브로커와의 연결 과정에서 MQTT객체의 callback 함수를 함께 설정해주었습니다. 각 주석이 의미하는대로 연결이 끊어졌을 때, 메시지가 왔을때, 메시지가 전송 되었을 때 어떤 일을 할지 지정할 수 있습니다.

### 메시지 보내기

```kotlin
    // MQTT 메시지 보내기
    fun sendMQTTMessage(msg: String) {
        try {
            // 전달받은 문자열을 MqttMessage로 변환
            val message = MqttMessage()
            message.payload = msg.toByteArray()
            // 특정 주제에게 메시지 publish
            mqttClient.publish(mqttTopic, message)
            println("메시지 전송: $message")
        } catch (e: MqttException) {
            println("메시지 전송 실패")
        }
    }
```

브로커로부터 구독한 주제에 대해 메시지가 들어왔을때는 콜백함수로 처리해주었지만 메시지를 보내는 경우 또한 가정하고 코드를 작성했습니다.

사용자로부터 메시지를 입력받고 입력받은 문자열을 MqttMessage 형태로 변환해서 publish 함수를 사용하면 특정 구독 멤버들에게 메시지를 보낼 수 있게 됩니다.

**MqttClient.publish( ${ Topic }, ${ Message } )**

### 구독과 구독 해제

```kotlin
    // 구독
    fun subscribe(topic: String?) {
        try {
            // 매개변수로 들어온 주제에 대해서 구독
            mqttClient.subscribe(topic ?: "test/topic") // 기본 주제 설정
            println("MQTT 주제 구독: ${topic ?: "test/topic"}")
        } catch (e: MqttException) {
            println("MQTT 주제 구독 실패: ${e.message}")
            e.printStackTrace()
        }
    }

    // 구독 해제
    fun unSubscribe(topic: String?) {
        try{
            mqttClient.unsubscribe(topic ?: "test/topic")
            println("MQTT 구독 중지 : ${topic ?: "test/topic"}")
        } catch (e: MqttException) {
            println("MQTT 구독 중지 실패 : ${e.message}")
            e.printStackTrace()
        }
    }
```

각 Subscriber는 구독을 기준으로 메시지를 받고 Publisher는 구독을 기준으로 메시지를 전달하는 방식으로 MQTT는 작동합니다. 그러므로 주제를 잘 구독해야 메시지를 주고받는데 문제가 없습니다. 

이 또한, MQTT객체에서 제공하는 subsribe와 unsubscribe를 통해 주제를 구독하고 해제할 수 있습니다.

**MqttClient.subscribe( ${ Topic } )**

**MqttClient.unsubscribe( ${ Topic } )**

# Mosquitto 로컬 액세스

---

아무리 로컬에서 Mosquitto 브로커를 활성화해봐도 Wear OS 클라이언트 프로그램에서는 브로커에 연결할 수 없다는 메시지만 띄울 뿐이었는데 다음은 위 문제를 해결한 방법에 대해서 기술하겠습니다.

처음에는 무료로 제공해주는 MQTT 브로커 서비스를 이용하여 동작이 되는지 테스트하려고 했습니다. Eclipse Mosquitto, HiveMQ Public Broker, Emqx 등 다양한 무료 사이트들에 대해 테스트를 시도했지만 모두 연결이 되지 않았습니다. 

이후, **로컬에 설치된 Mosquitto 브로커**를 이용하기로 했습니다. Mosquitto 브로커는 “mosquitto.exe -v”로 실행시 기본적으로 localhost 또는 127.0.0.1에 자동으로 바인딩 됩니다. 이는 로컬에서만 접속 가능하다는 뜻이 되기 때문에 저는 로컬에서 직접 브로커를 실행시켜 통신 테스트를 진행하였으나 이 역시 제대로 연결되지 못했습니다.

먼저 간과했던 것이 안드로이드 스튜디오에서 실행되는 에뮬레이터로는 로컬IP에 접근할 수 없었습니다. 아무리 이들이 로컬 머신과 동일한 네트워크에 속해 있더라도 별도의 네트워크 인터페이스를 통해 통신하기 때문에 개발 머신의 실제 IP 주소를 사용해야 했습니다. 또한, 기본적으로 Mosquitto는 익명 접속을 허용하지 않기 때문에 인증 없이 연결하고 액세스 할 수 있도록 설정해줘야 했습니다.

Mosquitto를 설치했다면, 해당 폴더에 들어가 mosquitto.conf 파일을 찾을 수 있을 것입니다. 이를 관리자 권한으로 실행하여 브로커가 바인딩 될 주소를 적어주고 필요하다면 익명 액세스 또한 허가해주면 됩니다.

```yaml
// mosquitto.conf
bind_address {myIpAddress}
allow_annonymous true
```

마지막으로 방화벽에서 포트 1883을 허용해 외부 클라이언트가 브로커에 접근할 수 있도록 포트를 개방해주어야 제대로 MQTT를 외부 프로그램에서 사용할 수 있었습니다.

# 글을 마무리하며

---

위 과정을 바탕으로 생각보다 서로 다른 네트워크나 프레임워크에서 통신을 주고받는 일이 어렵다는 것을 알 수 있었고 이를 해결하는 방법 또한 그렇게 어렵지 않은 일임을 알 수 있는 경험이었습니다.