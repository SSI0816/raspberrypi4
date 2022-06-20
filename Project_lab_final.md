1801253 신승일 프로젝트랩 발표자료

※목차
1. 작품 소개
2. 코드 소개
3. 동작 영상
----------------------------------------------------------------------------------------

1. 작품 소개
- Arduino Nano RP2040 6축 IMU 센서를 통해 걷거나 뛰는지 확인

 -> 나온 결과 값에 따라 서보모터를 제어하고 TTS로 결과 음성으로 출력
- 초음파 센서를 이용하여 거리를 측정

 -> 처음 측정된 거리와 나중에 측정된 거리를 비교하여 TTS로 결과 음성 출력

@ 측정 원리
1. 뛰거나 걷는지 측정

- Arduino Nano RP2040 6축 IMU 센서

-> 3D 자이로 스코프, 3D 가속도계

- 측정 방법

![크기변환 20220617_195158](https://user-images.githubusercontent.com/103232864/174289639-3e8de1cc-3b39-49c7-8d13-0e5417529d04.jpg)


- 6축 IMU 센서 측정 값의 방향

![제목 없음](https://user-images.githubusercontent.com/103232864/174297186-fdeeebf4-f5e9-4c14-bee4-5ca40dbfbde1.png)


- 사람이 걷는 경우 팔의 모습

![크기변환 크기변환 png-transparent-computer-icons-walking-walking-miscellaneous-hand-monochrome](https://user-images.githubusercontent.com/103232864/174289775-d421f7c2-60ac-4869-81e5-e5390bc89167.png)


- 사람이 뛰는 경우 팔의 모습

![10873](https://user-images.githubusercontent.com/103232864/174286260-aa8bd5aa-7436-470a-a7b9-39bd1cfca6ee.png)


=> x축의 가속도 측정값의 크기(절대값)을 10초동안 측정하여 평균값을 구한다
   - 평균값이 1.0 이상 -> 뛰는 상황
   - 평균값이 1.0 미만 -> 걷는 상황 
 
2. 가까이 오거나 멀리 가는지 측정

- HC-SR04

![다운로드](https://user-images.githubusercontent.com/103232864/174289946-af692a34-b17d-4cb6-8af4-187b97db5243.jpg)


- 측정 원리

![image](https://user-images.githubusercontent.com/103232864/174287220-04a174d5-db5c-40df-b2bb-5f32022150ac.png)

- 초음파 발생 시간과 물체에 부딪혀 반사되어 돌아오는 시간차를 이용
-  거리 = 시간 X 속력(초음파 속력 = 340m/s로 일정)

=> 처음 측정 거리를 측정-> 3초뒤 측정 거리를 측정 -> 처음 거리에서 나중 거리의 차이 구하기
   - 결과값이 음수인 경우 -> 멀어지고 있는 상태
   - 결과값이 양수인 경우 -> 가까워지고 있는 상태 

----------------------------------------------------------------------------------------

2. 코드 소개

-   Arduino Nano RP2040 6축 IMU 센서 측정(MQTT-Publisher)
```
import network,time
from umqtt.simple import MQTTClient 
from machine import I2C,Pin,Timer
import time
from lsm6dsox import LSM6DSOX
step1 = 0
from machine import Pin, I2C
lsm = LSM6DSOX(I2C(0, scl=Pin(13), sda=Pin(12)))



def WIFI_Connect():
    wlan = network.WLAN(network.STA_IF) 
    wlan.active(True)                   
    start_time=time.time()              

    if not wlan.isconnected():
        print('connecting to network...')
        wlan.connect('13326-2GHz', 'iac-4701') 
        
    if wlan.isconnected():
        print('network information:', wlan.ifconfig())
        return True    

def MQTT_Send(tim):
    client.publish(TOPIC, 'Accelerometer: x:{:>8.3f} y:{:>8.3f} z:{:>8.3f}'.format(*lsm.read_accel()))
    print('Accelerometer: x:{:>8.3f} y:{:>8.3f} z:{:>8.3f}'.format(*lsm.read_accel()))
    print("")
    time.sleep_ms(100)

if WIFI_Connect():
    SERVER = '192.168.0.99'   # my rapa ip address , mqtt broker가 실행되고 있음
    PORT = 1883
    CLIENT_ID = 'Shin' # clinet id 이름
    TOPIC = 'Acceler' # TOPIC 이름
    client = MQTTClient(CLIENT_ID, SERVER, PORT,keepalive=30)
    client.connect()
    tim = Timer(-1)
    tim.init(period=1000, mode=Timer.PERIODIC,callback=MQTT_Send)
    
```

- 걷거나 뛰는지 판단(MQTT-Subscriber)
```
import random
import time
import paho.mqtt.client as mqtt_client
import re
from pymata4 import pymata4
from gtts import gTTS 
import os 
import time 
import playsound 
import pygame

pygame.mixer.init()

board = pymata4.Pymata4()

servo = board.set_pin_mode_servo(11)

def move_servo(v):                  
    board.servo_write(11, v)
    time.sleep(1) 


def speak_save(text):
    tts = gTTS(lang='en', text=text ) #ko')
    filename='/home/pi/Desktop/Rapa_Mqtt/voice.mp3'
    tts.save(filename) 

def speaker_out():
    pygame.mixer.music.load("/home/pi/Desktop/Rapa_Mqtt/voice.mp3")
    pygame.mixer.music.play()
    while pygame.mixer.music.get_busy() == True:
        continue

global count
count = 0
global x_accel 
x_accel = 0
global y_accel 
y_accel = 0
global z_accel 
z_accel = 0

broker_address = "localhost"
broker_port = 1883

topic = "Acceler"
topic2 = 'ultra'


def connect_mqtt() -> mqtt_client:
    def on_connect(client, userdata, flags, rc):
        if rc == 0:
            print("Connected to MQTT Broker")
        else:
            print(f"Failed to connect, Returned code: {rc}")

    def on_disconnect(client, userdata, flags, rc=0):
        print(f"disconnected result code {str(rc)}")

    def on_log(client, userdata, level, buf):
        print(f"log: {buf}")

    # client 생성
    client_id = f"mqtt_client_{random.randint(0, 1000)}"
    client = mqtt_client.Client(client_id)

    # 콜백 함수 설정
    client.on_connect = on_connect
    client.on_disconnect = on_disconnect
    client.on_log = on_log

    # broker 연결
    client.connect(host=broker_address, port=broker_port)
    return client


def subscribe(client: mqtt_client):
    def on_message(client, userdata, msg):
        print(f"Received `{msg.payload.decode()}")
        value = msg.payload.decode()
        numbers = re.findall(r'\d+.\d+', value)
        print(numbers)
        global count
        global x_accel 
        global y_accel 
        global z_accel 
        if count == 0:
            F_dis = float(numbers[0])
        if count == 10: 
            x_result = x_accel / 10
            y_result = y_accel / 10
            z_result = z_accel / 10
            print("============================================================")
            print(f"result : {x_result}', '{y_result}', '{z_result}")
            print("============================================================")
            x_accel = 0
            y_accel = 0
            z_accel = 0
            count = 0
            if x_result > 1.0:
              speak_save("I'm running.")
              speaker_out()
              move_servo(180)
              move_servo(0)  
              move_servo(180)
              move_servo(0) 
            if x_result < 1.0:
              speak_save("I'm walking.")
              speaker_out() 
              move_servo(90)
              move_servo(0)  
              move_servo(90)
              move_servo(0)   
        else :
            count = count +1 
        x_accel = x_accel + float(numbers[0])
        y_accel = y_accel + float(numbers[1])
        z_accel = z_accel + float(numbers[2])
    client.subscribe(topic) #1
    client.on_message = on_message



def run():
    client = connect_mqtt()
    subscribe(client)
    client.loop_forever()


if __name__ == '__main__':
    run()
```

- 초음파 센서 거리 측정(MQTT-Publisher)
```
import random
import time
import paho.mqtt.client as mqtt_client
import re
from gtts import gTTS 
import os 
import time 
import playsound 
import pygame

from sklearn.datasets import load_digits

pygame.mixer.init()

def speak_save(text):
    tts = gTTS(lang='en', text=text ) #ko')
    filename='/home/pi/Desktop/vsc_ws/voice.mp3'
    tts.save(filename) 

def speaker_out():
    pygame.mixer.music.load("/home/pi/Desktop/vsc_ws/voice.mp3")
    pygame.mixer.music.play()
    while pygame.mixer.music.get_busy() == True:
        continue

global count
count = 0
global distance
distance = 0
global F_dis
F_dis = 0

broker_address = "localhost"
broker_port = 1883

topic = "ultra"


def connect_mqtt() -> mqtt_client:
    def on_connect(client, userdata, flags, rc):
        if rc == 0:
            print("Connected to MQTT Broker")
        else:
            print(f"Failed to connect, Returned code: {rc}")

    def on_disconnect(client, userdata, flags, rc=0):
        print(f"disconnected result code {str(rc)}")

    def on_log(client, userdata, level, buf):
        print(f"log: {buf}")

    # client 생성
    client_id = f"mqtt_client_{random.randint(0, 1000)}"
    client = mqtt_client.Client(client_id)

    # 콜백 함수 설정
    client.on_connect = on_connect
    client.on_disconnect = on_disconnect
    client.on_log = on_log

    # broker 연결
    client.connect(host=broker_address, port=broker_port)
    return client


def subscribe(client: mqtt_client):
    def on_message(client, userdata, msg):
        print(f"Received `{msg.payload.decode()}` from `{msg.topic}` topic")
        dis = msg.payload.decode()
        number = re.findall(r'\d+.\d+', dis)
        print(number)
        global count
        global distance
        global F_dis
        if count == 0:
            F_dis = float(number[0])
        if count == 3:
            L_dis = float(number[0])
            distance = F_dis - L_dis
            print("============================================================")
            print(f"result : {distance}'")
            print("============================================================")
            if distance > 0:
                print("I'm getting closer.")
                speak_save("I'm getting closer")
                speaker_out()
            if distance < 0:
                print("I'm moving away.")
                speak_save("I'm moving away.")
                speaker_out()
            distance = 0
            count = 0

        else :
            count = count + 1
        

    client.subscribe(topic) #1
    client.on_message = on_message


def run():
    client = connect_mqtt()
    subscribe(client)
    client.loop_forever()


if __name__ == '__main__':
    run()
```

- 가까워 지는지 멀어지는지 판단(MQTT-Subscriber)
```
import random
import time
import paho.mqtt.client as mqtt_client
import re
from gtts import gTTS 
import os 
import time 
import playsound 
import pygame

from sklearn.datasets import load_digits

pygame.mixer.init()

def speak_save(text):
    tts = gTTS(lang='en', text=text ) #ko')
    filename='/home/pi/Desktop/vsc_ws/voice.mp3'
    tts.save(filename) 

def speaker_out():
    pygame.mixer.music.load("/home/pi/Desktop/vsc_ws/voice.mp3")
    pygame.mixer.music.play()
    while pygame.mixer.music.get_busy() == True:
        continue

global count
count = 0
global distance
distance = 0
global F_dis
F_dis = 0

broker_address = "localhost"
broker_port = 1883

topic = "ultra"


def connect_mqtt() -> mqtt_client:
    def on_connect(client, userdata, flags, rc):
        if rc == 0:
            print("Connected to MQTT Broker")
        else:
            print(f"Failed to connect, Returned code: {rc}")

    def on_disconnect(client, userdata, flags, rc=0):
        print(f"disconnected result code {str(rc)}")

    def on_log(client, userdata, level, buf):
        print(f"log: {buf}")

    # client 생성
    client_id = f"mqtt_client_{random.randint(0, 1000)}"
    client = mqtt_client.Client(client_id)

    # 콜백 함수 설정
    client.on_connect = on_connect
    client.on_disconnect = on_disconnect
    client.on_log = on_log

    # broker 연결
    client.connect(host=broker_address, port=broker_port)
    return client


def subscribe(client: mqtt_client):
    def on_message(client, userdata, msg):
        print(f"Received `{msg.payload.decode()}` from `{msg.topic}` topic")
        dis = msg.payload.decode()
        number = re.findall(r'\d+.\d+', dis)
        print(number)
        global count
        global distance
        global F_dis
        if count == 0:
            F_dis = float(number[0])
        if count == 3:
            L_dis = float(number[0])
            distance = F_dis - L_dis
            print("============================================================")
            print(f"result : {distance}'")
            print("============================================================")
            if distance > 0:
                print("I'm getting closer.")
                speak_save("I'm getting closer")
                speaker_out()
            if distance < 0:
                print("I'm moving away.")
                speak_save("I'm moving away.")
                speaker_out()
            distance = 0
            count = 0

        else :
            count = count + 1
        

    client.subscribe(topic) #1
    client.on_message = on_message


def run():
    client = connect_mqtt()
    subscribe(client)
    client.loop_forever()


if __name__ == '__main__':
    run()
```

----------------------------------------------------------------------------------------

3. 동작 영상

- 걷기, 뛰기 판별


https://user-images.githubusercontent.com/103232864/174296542-3a2917ca-7e69-4dd6-b3cf-e66a524f85c1.mp4


- 서브모터 

-> 걷는 경우 : 0~90도 2번 움직임

-> 뛰는 경우 : 0~180도 2번움직임


- 멀어지기, 가까워지기 판별


https://user-images.githubusercontent.com/103232864/174296627-cc580d1f-1fb1-4f7e-b08a-b1b57764b7f1.mp4


- 위 2가지 기능 동시 사용



https://user-images.githubusercontent.com/103232864/174296666-a1c99b67-8a3f-49d1-806b-d901bf204a3f.mp4






*사진 출처
 1. Arduino Nano RP2040(https://www.steamrobot.co.kr/shop/item.php?it_id=1578633954)
 2. 걷는 사람(https://www.pngwing.com/ko/free-png-nibwy)
 3. 뛰는 사람(https://www.flaticon.com/kr/free-icon/running-person_10873)
 4. HC-SR04([https://ko.aliexpress.com/item/32583948734.html](https://embedscope.com/96)
 5. HC-SR04측정 원리(https://m.blog.naver.com/PostView.naver?isHttpsRedirect=true&blogId=eduino&logNo=220858628911)
