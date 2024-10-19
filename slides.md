---
theme: default
title: Raspberry Pi Pico WでMicroPythonを始めてみましょう
info: |
  ## PyLadies Tokyo10周年記念パーティ/10th Anniversary Party 発表資料
  Raspberry Pi Pico WでMicroPythonを始めてみましょう

  中村草介(Sousuke NAKAMURA)
drawings:
  persist: false
transition: slide-left
mdc: true
layout: cover
---

# PyLadies Tokyo10周年記念パーティ/10th Anniversary Party 発表資料
## MicroPythonとRaspberry Pi Pico Wで始めるワイヤレス通信

中村草介(Sousuke NAKAMURA)

---
layout: center
---

## はじめに
- 資料URL(暫定) https://nkm3.org/raspberry_pi_pico_w_network_slide_pyladies/

---

## 自己紹介

- 中村草介( @nakamurasousuke GitHub: sonkm3)
- Raspberry Piもくもく会スタッフ

<img src='/75330.jpeg' align='right' width='200px'/>


---

## 1. 概要
- Raspberry Pi Pico Wとは
- MicroPythonとは

---
layout: two-cols-header
---

## Raspberry Pi Pico Wとは

::left::

- RP2040を搭載したワンボードマイコンで、WiFi、Bluetoothの接続機能が備わっています
  - Raspberry Pi財団が開発したARM Cortex M0+デュアルコアMPU(最高133MHzで動作)
    - RAM 256kB
    - GPIOx30(UARTx2 I2Cx2 SPIx2 PWMx16 PIOx8 12bitADCx4 USB1.1)
  - 2MBのフラッシュメモリが接続されています
  - Infineon CYW43439がSPIでRP2040に接続されています
    - WiFi 802.11n(2.4 GHz Wi-Fi 4) + Bluetooth® 5.2(BDR(1 Mbps)/EDR(2/3 Mbps)/Bluetooth® LE)
  - ※Linuxなどが動作するRaspberry Piとは違います

<https://www.raspberrypi.com/documentation/microcontrollers/pico-series.html>

::right::

<img src='/IMG_7755.jpg' width='300px' align='right'/>


---

## MicroPythonとは

- Python3のサブセットで、マイクロコントローラー向けに最適化された言語です
- 使い慣れたPythonと標準ライブラリがある程度使えます <https://micropython.org>

```python
>>> help('modules')
__main__          asyncio/__init__  hashlib           rp2
_asyncio          asyncio/core      heapq             select
_boot             asyncio/event     io                socket
_boot_fat         asyncio/funcs     json              ssl
_onewire          asyncio/lock      lwip              struct
_rp2              asyncio/stream    machine           sys
_thread           binascii          math              time
_webrepl          bluetooth         micropython       tls
aioble/__init__   builtins          mip/__init__      uasyncio
aioble/central    cmath             neopixel          uctypes
aioble/client     collections       network           urequests
aioble/core       cryptolib         ntptime           vfs
aioble/device     deflate           onewire           webrepl
aioble/l2cap      dht               os                webrepl_setup
aioble/peripheral ds18x20           platform          websocket
aioble/security   errno             random
aioble/server     framebuf          re
array             gc                requests/__init__
Plus any modules on the filesystem
```

---

## MicroPythonのインストール

- 専用のツール、ボードなどは使わず、USBケーブル一本で(マスストレージ経由)で書き込むことができます
- 手順
  1. uf2ファイルをダウンロード <https://micropython.org/download/RPI_PICO_W/>
  2. Raspeberry Pi Pico WのBOOTSELボタンを押しながらUSBケーブルをコンピューターに接続
  3. USBストレージとして認識されるので、MicroPythonのuf2ファイルをコピーします
  4. コピーが終わると自動的に再起動(アンマウント)されます



---

## 開発環境の準備

- RaspberryPiの公式ドキュメントではThonny (<https://thonny.org>) がお勧めされています
- <https://projects.raspberrypi.org/en/projects/getting-started-with-the-pico>
- ファイル(ローカル、デバイス)、エディター、REPL、デバッグ用のペインが揃っています

<img src='/thonny.png' align='center' width='500px'/>


---

## mpremoteを使って動作確認

- Raspberry Pi Pico W上ではMicroPythonのREPLが起動しているのでシリアルコンソールで接続することができます
- thonnyの通信ペインから接続できます

```shell
Connected to MicroPython at /dev/cu.usbmodem101
Use Ctrl-] or Ctrl-x to exit this shell

>>> import sys
>>> sys.implementation
(name='micropython', version=(1, 23, 0, ''), _machine='Raspberry Pi Pico W with RP2040', _mpy=4870)
>>>
```

---
layout: two-cols-header
---

## まずはhello world(出力)しましょう

- hello worldに相当するLEDの点滅
- Arduino(C言語風のArduino言語を使ったワンボードマイコン向けの統合開発環境)のサンプルコードと同じような流れでLEDを点滅させることができます

::left::
- Arduinoのチュートリアルにあるサンプルコード

```cpp
void setup() {
   pinMode(LED_BUILTIN, OUTPUT);
}

void loop() {
   digitalWrite(LED_BUILTIN, HIGH);
   delay(1000);
   digitalWrite(LED_BUILTIN, LOW);
   delay(1000);
}
```

<https://docs.arduino.cc/tutorials/uno-rev3/Blink/>


::right::

- Micro Python

```python
import machine
import time

led = machine.Pin('LED', machine.Pin.OUT)

while True:
    led.value(1)
    time.sleep(1)
    led.value(0)
    time.sleep(1)
```

---

## MicroPythonでのLチカ

- PythonなのでTimerオブジェクトのコールバックにlambdaを使ってシンプルに書くこともできます

```python
import machine
led = machine.Pin('LED', machine.Pin.OUT)
timer = machine.Timer()
timer.init(freq=2.5, mode=machine.Timer.PERIODIC, callback=lambda _: led.toggle())
```

---

## 内蔵の温度センサ(入力)の値を測ってみましょう

- RP2040内蔵の温度計は5つ目のADコンバーターに接続されています
- 以下の計算をすることでCPUの温度が求められます
  - `T = 27 - (ADC_voltage - 0.706)/0.001721`
- RP2040のデータシート566ページめに記載があります
  - https://datasheets.raspberrypi.com/rp2040/rp2040-datasheet.pdf

```python
import machine
import time

temp_adc = machine.ADC(4) # RP2040内蔵の温度計は5つ目のADコンバーターに接続されています

def get_temperature():
    v = (3.3/65535) * temp_adc.read_u16()
    t = 27 - (v - 0.706)/0.001721
    return t

while True:
    print(get_temperature())
    time.sleep(1)
```

---

## 温度によってLEDの点滅速度を変えてみましょう

```python
import machine
import time


timer = machine.Timer()
temp_adc = machine.ADC(4) # RP2040内蔵の温度計は5つ目のADコンバーターに接続されています
led = machine.Pin('LED', machine.Pin.OUT)


def get_temperature():
    v = (3.3/65535) * temp_adc.read_u16()
    t = 27 - (v - 0.706)/0.001721
    return t

while True:
    t = get_temperature()
    freq = (t - 20) ** 4 * 0.001 # 指で温めた30度前後でいい感じに点滅速度が変わるように温度から周波数を調整
    print(t)

    timer.init(freq=freq, mode=machine.Timer.PERIODIC, callback=lambda _: led.toggle())
    time.sleep(1)
```


---

## 自己紹介(お仕事の紹介)

- 株式会社アーバンエックステクノロジーズ
  - 道路など都市インフラの管理をソフトウェアでサポートするサービスを提供しています
- ソフトウェアエンジニアのみなさま、絶賛募集中です！

<img src='https://urbanx-tech.com/wp-content/uploads/2022/08/cropped-Logo@3x.png' align='right' width='200px'/>
