# CamillaDSP-OPiZero2W
**OrangePi Zero 2W + Bullseye(RaspiOS 기반 이미지) + ALSA mixbus + CamillaDSP + Bluetooth + AirPlay**

작성 기준:
- 사용자: `<사용자명>` (예: `yuusha`)
- 기기 표시 이름: `<기기명>` (예: `YuushaDSP`)
- 대상 OS: `Orangepizero2w_1.0.2_raspios_bullseye_server_linux6.1.31`
- 목표:
  - USB-C UAC2 Gadget 입력
  - Bluetooth A2DP 입력
  - AirPlay / AirPlay 2 입력
  - 위 3개를 **ALSA mixbus(스테레오)** 에 먼저 합성
  - CamillaDSP로 룸보정 적용
  - OrangePi에 연결된 USB DAC 2채널로 출력
  - CamillaGUI(Web) 사용
  - Bluetooth 자동 페어링 / 자동 discoverable
  - Bluetooth/AirPlay 이름이 모두 `<기기명>`으로 보이도록 구성

---

## 0. 핵심 구조

```text
USB UAC2 Gadget 입력 ----\
Bluetooth A2DP 입력 -----+--> ALSA mixbus(스테레오 합성) --> CamillaDSP --> USB DAC 2ch 출력
AirPlay / AirPlay 2 ----/
```

이 구조를 쓰는 이유:

- Debian Bullseye + OrangePi 환경에서 가장 안정적임
- 모든 소스를 먼저 스테레오로 섞고, 마지막에 룸보정을 한 번만 적용하면 운용이 쉬움
- Bluetooth 동시 연결/동시 재생은 `bluealsa-aplay -> dmix PCM` 구조가 가장 현실적임
- AirPlay는 `Shairport Sync -> ALSA mixbus` 구조가 가장 단순함
- PipeWire/PulseAudio 전체 이행 없이 ALSA만으로 정리 가능
- 44.1 kHz Bluetooth 소스가 와도 `plug`/`rate` 경로를 통해 고정 48 kHz mixbus로 넣을 수 있음

---

## 1. 버전 기준

아래 버전을 기준으로 작성합니다.

- CamillaDSP: `4.1.3`
- CamillaGUI backend: `4.1.0`
- BlueALSA: `v4.3.1` 태그 기준
- Shairport Sync: `5.0.2`
- NQPTP: `1.2.4`

---

## 2. 기본 패키지 설치 + 호스트명 정리

### 2-0. OS 설치
https://github.com/leeboby/raspberry-pi-os-images/releases/download/h618-20240711/Orangepizero2w_1.0.2_raspios_bullseye_server_linux6.1.31.7z  
Orangepizero2w_1.0.2_raspios_bullseye_server_linux6.1.31.img 다운로드 및 설치  
이후 **raspi-config** 로 기본 설정 진행  


### 2-1. 시스템 업데이트

```bash
sudo apt update
sudo apt full-upgrade -y
```

### 2-2. 필요한 패키지 설치

```bash
sudo apt install -y --no-install-recommends \
  git curl wget unzip ca-certificates jq nano \
  build-essential pkg-config autoconf automake libtool cmake \
  python3 python3-venv python3-pip python3-dev python3-dbus python3-gi python3-docutils \
  alsa-utils \
  libasound2-dev libdbus-1-dev libglib2.0-dev libbluetooth-dev libsbc-dev \
  libsamplerate0-dev \
  libavcodec-dev libavformat-dev libavutil-dev \
  bluez avahi-daemon libavahi-client-dev \
  libssl-dev libsoxr-dev libplist-dev libplist-utils libsodium-dev \
  libconfig-dev libpopt-dev uuid-dev libgcrypt20-dev xxd
```

### 2-3. 사용자/기기 이름 정리

```bash
# 여기에 본인의 사용자명과 기기 이름을 입력하세요
# 예: 사용자명이 yuusha 라면 → DSP_USER="yuusha", DEVICE_NAME="YuushaDSP"
export DSP_USER="<사용자명>"
export DEVICE_NAME="<기기명>"

export USER_HOME="/home/${DSP_USER}"
export BIN_HOME="${USER_HOME}/bin"
export CDSP_HOME="${USER_HOME}/camilladsp"
export SRC_HOME="${USER_HOME}/src"
export MAKE_JOBS="2"

export CDSP_VER="4.1.3"
export CAMILLAGUI_VER="4.1.0"
export BLUEALSA_VER="v4.3.1"
export NQPTP_VER="1.2.4"
export SHAIRPORT_VER="5.0.2"

# 실제 카드명은 나중에 aplay -l / arecord -l 결과를 보고 조정
export UAC2_DEV="hw:UAC2Gadget,0"
export DAC_DEV="hw:Audio,0"

export LOOP_PLAY_DEV="hw:Loopback,0,0"
export LOOP_CAP_DEV="hw:Loopback,1,0"
```
기본 변수 설정 후,

```bash
sudo hostnamectl set-hostname ${DSP_USER}
sudo hostnamectl set-hostname --pretty "${DEVICE_NAME}"
hostnamectl status
```

### 2-4. 작업 폴더 만들기

```bash
mkdir -p "${BIN_HOME}" "${CDSP_HOME}"/{bin,configs,coeffs,gui,temp}
mkdir -p "${SRC_HOME}"
sudo chown -R "${DSP_USER}:${DSP_USER}" "${BIN_HOME}" "${CDSP_HOME}" "${SRC_HOME}"
```

---
## 3. USB-C UAC2 Gadget 만들기

> - USB 장치 이름이 `<기기명>`으로 뜨도록 product/manufacturer 문자열 고정

### 3-1. 스크립트 작성

```bash
cat <<EOF > "${BIN_HOME}/usb-gadget-dsp.sh"
#!/bin/bash
set -euo pipefail

G="/sys/kernel/config/usb_gadget/dsp-audio"

modprobe sunxi || true
modprobe configfs
modprobe libcomposite

cleanup() {
  if [ -d "\${G}" ]; then
    echo "" > "\${G}/UDC" 2>/dev/null || true
    rm -f "\${G}/configs/c.1/uac2.usb0" 2>/dev/null || true
    rmdir "\${G}/functions/uac2.usb0" 2>/dev/null || true
    rmdir "\${G}/configs/c.1/strings/0x409" 2>/dev/null || true
    rmdir "\${G}/configs/c.1" 2>/dev/null || true
    rmdir "\${G}/strings/0x409" 2>/dev/null || true
    rmdir "\${G}" 2>/dev/null || true
  fi
}

cleanup

mkdir -p "\${G}"
cd "\${G}"

echo 0x1d6b > idVendor
echo 0x0104 > idProduct
echo 0x0100 > bcdDevice
echo 0x0200 > bcdUSB

mkdir -p strings/0x409
echo "${DEVICE_NAME}0001" > strings/0x409/serialnumber
echo "${DEVICE_NAME}" > strings/0x409/manufacturer
echo "${DEVICE_NAME}" > strings/0x409/product

mkdir -p configs/c.1/strings/0x409
echo "${DEVICE_NAME} UAC2" > configs/c.1/strings/0x409/configuration
echo 120 > configs/c.1/MaxPower

mkdir -p functions/uac2.usb0
echo 3 > functions/uac2.usb0/c_chmask
echo 3 > functions/uac2.usb0/p_chmask
echo 48000 > functions/uac2.usb0/c_srate
echo 48000 > functions/uac2.usb0/p_srate
echo 4 > functions/uac2.usb0/c_ssize
echo 4 > functions/uac2.usb0/p_ssize
echo 1 > functions/uac2.usb0/c_mute_present
echo 1 > functions/uac2.usb0/c_volume_present

ln -s functions/uac2.usb0 configs/c.1/

UDC_NAME="\$(ls /sys/class/udc | head -n1)"
if [ -z "\${UDC_NAME}" ]; then
  echo "No UDC found. USB device mode may not be active on this kernel/port."
  exit 1
fi

echo "\${UDC_NAME}" > UDC
EOF

chmod +x "${BIN_HOME}/usb-gadget-dsp.sh"
```

### 3-2. systemd 서비스 작성

```bash
cat <<EOF | sudo tee /etc/systemd/system/dsp-audio.service >/dev/null
[Unit]
Description=${DEVICE_NAME} USB Audio Gadget
After=local-fs.target
Before=sound.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=${BIN_HOME}/usb-gadget-dsp.sh

[Install]
WantedBy=multi-user.target
EOF
```

### 3-3. 테스트

```bash
sudo systemctl daemon-reload
sudo systemctl enable dsp-audio
sudo systemctl start dsp-audio
sudo systemctl status dsp-audio
arecord -l
dmesg | grep -i -E "uac|gadget|udc"
```

`arecord -l` 에서 UAC2 gadget 카드명이 보이면 성공입니다.  
이때 카드명이 `UAC2Gadget` 이 아니라 `UAC2_Gadget` 같이 다르게 나오면, 위에서 잡아둔 `UAC2_DEV` 값을 나중에 바꾸세요.

---

## 4. ALSA Loopback + mixbus 만들기

### 4-1. `snd-aloop` 모듈 고정

```bash
echo 'options snd-aloop id=Loopback index=10 pcm_substreams=8' | \
  sudo tee /etc/modprobe.d/dsp-aloop.conf >/dev/null

echo 'snd-aloop' | sudo tee /etc/modules-load.d/dsp-aloop.conf >/dev/null

sudo modprobe snd-aloop
aplay -l
arecord -l
```

### 4-2. `/etc/asound.conf` 작성

- `pcm.mixbus`  
  여러 프로세스가 동시에 열 수 있는 dmix 출력 버스
- `pcm.mixcap`  
  mixbus의 반대편 캡처 장치
- `pcm.airplay` / `pcm.bluetooth` / `pcm.usbmix`  
- `plug` 를 사용하므로 44.1 kHz 소스가 와도 48 kHz mixbus로 넣을 수 있음
- 현재 버퍼 사이즈 1024 (약 5.3ms)

```bash
cat <<EOF | sudo tee /etc/asound.conf >/dev/null
pcm.mixbus_dmix {
    type dmix
    ipc_key 147852
    ipc_key_add_uid false
    ipc_perm 0666
    slave {
        pcm "${LOOP_PLAY_DEV}"
        format S32_LE
        rate 48000
        channels 2
        period_time 0
        period_size 256
        buffer_size 1024
    }
}

pcm.mixbus {
    type plug
    slave.pcm "mixbus_dmix"
}

pcm.mixcap {
    type dsnoop
    ipc_key 147853
    ipc_key_add_uid false
    ipc_perm 0666
    slave {
        pcm "${LOOP_CAP_DEV}"
        format S32_LE
        rate 48000
        channels 2
    }
}

pcm.airplay {
    type plug
    slave.pcm "mixbus"
}

pcm.bluetooth {
    type plug
    slave.pcm "mixbus"
}

pcm.usbmix {
    type plug
    slave.pcm "mixbus"
}
EOF
```

### 4-3. mixbus 테스트

```bash
speaker-test -D mixbus -c 2 -r 48000 -F S32_LE
```

지금은 아직 CamillaDSP와 USB DAC 출력이 안 붙었기 때문에 실제 스피커로는 안 나올 수 있습니다.  
하지만 `mixbus` PCM이 에러 없이 열리면 일단 정상입니다.

---

## 5. CamillaDSP + CamillaGUI 설치

### 5-1. arm64용으로 빌드한 CamillaDSP 사용


OrangePi Zero 2w 의 사양으로 인해 직접 빌드는 실패합니다.  
미리 빌드해준 파일을 씁시다.  

1. 외부 PC에서 Bullseye arm64 환경으로 `camilladsp v4.1.3` 바이너리를 빌드한 파일 가져오기(https://github.com/chardiet/CamillaDSP-OPiZero2W/blob/main/camilladsp 다운로드)
2. OrangePi의 camilladsp/bin 폴더 안으로 복사
3. 아래처럼 버전 확인

```bash
"${CDSP_HOME}/bin/camilladsp" --version
```

### 5-2. CamillaGUI backend 설치

```bash
cd "${CDSP_HOME}/temp"
curl -L -o "camillagui.zip" \
  "https://github.com/HEnquist/camillagui-backend/releases/download/v${CAMILLAGUI_VER}/camillagui.zip"

unzip -o "camillagui.zip" -d "${CDSP_HOME}/gui"
```

### 5-3. venv 만들기

```bash
python3 -m venv "${CDSP_HOME}/venv"
"${CDSP_HOME}/venv/bin/pip" install --upgrade pip wheel setuptools
"${CDSP_HOME}/venv/bin/pip" install -r "${CDSP_HOME}/gui/requirements.txt"
"${CDSP_HOME}/venv/bin/pip" install websocket-client pyalsaaudio pyyaml
```

### 5-4. CamillaDSP 설정 파일 작성

> `DAC_DEV` 는 반드시 실제 USB DAC 카드명에 맞게 확인하세요.  
> `aplay -l` 로 확인 후 필요하면 다시 export 하세요.  
> 예: `export DAC_DEV="hw:Pro,0"`

```bash
aplay -l
arecord -l
```

```bash
cat <<EOF > "${CDSP_HOME}/configs/default.yml"
***
devices:
  samplerate: 48000
  chunksize: 1024
  queuelimit: 4
  enable_rate_adjust: true
  capture:
    type: Alsa
    channels: 2
    device: "${LOOP_CAP_DEV}"
    format: S32_LE
  playback:
    type: Alsa
    channels: 2
    device: "${DAC_DEV}"
    format: S32_LE
EOF
```

> 필터와 파이프라인이 없는 최소 설정입니다. 에러 없이 동작하며, 이후 REW biquad 필터나 FIR convolution을 적용할 때는 GUI를 사용해 추가하면 됩니다.

### 5-5. CamillaGUI 설정 작성

```bash
mkdir -p "${CDSP_HOME}/gui/config"

cat <<EOF > "${CDSP_HOME}/gui/config/camillagui.yml"
***
camilla_host: "127.0.0.1"
camilla_port: 1214

bind_address: "0.0.0.0"
port: 1215

ssl_certificate: null
ssl_private_key: null
gui_config_file: null
config_dir: "${CDSP_HOME}/configs"
coeff_dir: "${CDSP_HOME}/coeffs"
default_config: "${CDSP_HOME}/configs/default.yml"
statefile_path: "${CDSP_HOME}/statefile.yml"
log_file: "${CDSP_HOME}/camilladsp.log"

on_set_active_config: null
on_get_active_config: null
supported_capture_types: null
supported_playback_types: null
EOF
```

### 5-6. statefile / log 파일 권한 설정

```bash
sudo touch "${CDSP_HOME}/statefile.yml" "${CDSP_HOME}/camilladsp.log"
sudo chown "${DSP_USER}:${DSP_USER}" "${CDSP_HOME}/statefile.yml" "${CDSP_HOME}/camilladsp.log"
sudo chmod 0644 "${CDSP_HOME}/statefile.yml" "${CDSP_HOME}/camilladsp.log"
```

### 5-7. CamillaDSP / CamillaGUI systemd 서비스 작성

```bash
cat <<EOF | sudo tee /etc/systemd/system/camilladsp.service >/dev/null
[Unit]
Description=CamillaDSP Service
After=sound.target network.target
StartLimitIntervalSec=10
StartLimitBurst=10

[Service]
Type=simple
ExecStart=${CDSP_HOME}/bin/camilladsp ${CDSP_HOME}/configs/default.yml -g 0.0 -a 127.0.0.1 -p 1214 -o ${CDSP_HOME}/camilladsp.log -s ${CDSP_HOME}/statefile.yml
Restart=always
RestartSec=2
User=root
Group=root
CPUSchedulingPolicy=fifo
CPUSchedulingPriority=10

[Install]
WantedBy=multi-user.target
EOF
```

`camillagui.service` 는 config 파일을 명시적으로 지정하고,
CamillaDSP websocket 포트가 열릴 때까지 잠깐 기다리도록 둡니다.

```bash
cat <<EOF | sudo tee /etc/systemd/system/camillagui.service >/dev/null
[Unit]
Description=CamillaGUI Backend
After=network.target camilladsp.service
Requires=camilladsp.service

[Service]
Type=simple
User=${DSP_USER}
Group=${DSP_USER}
WorkingDirectory=${CDSP_HOME}/gui
ExecStartPre=/bin/bash -lc 'for i in $(seq 1 20); do (echo >/dev/tcp/127.0.0.1/1214) >/dev/null 2>&1 && exit 0; sleep 0.5; done; exit 1'
ExecStart=${CDSP_HOME}/venv/bin/python ${CDSP_HOME}/gui/main.py -c ${CDSP_HOME}/gui/config/camillagui.yml
Restart=always
RestartSec=2

[Install]
WantedBy=multi-user.target
EOF
```

### 5-8. USB gadget 입력을 mixbus로 보내는 브리지 서비스

이제는 **USB 입력도 mixbus로 보내고**, CamillaDSP는 mixbus 캡처만 읽습니다.

```bash
cat <<EOF | sudo tee /etc/systemd/system/uac2-mixbus.service >/dev/null
[Unit]
Description=${DEVICE_NAME} UAC2 Gadget to ALSA mixbus bridge
After=dsp-audio.service sound.target
Requires=dsp-audio.service

[Service]
Type=simple
ExecStart=/bin/bash -lc '/usr/bin/arecord -D ${UAC2_DEV} -f S32_LE -c 2 -r 48000 -q | /usr/bin/aplay -D mixbus -f S32_LE -c 2 -r 48000 -q'
Restart=always
RestartSec=1

[Install]
WantedBy=multi-user.target
EOF
```

---

## 6. Bluetooth 수신기 구성 (BlueZ + BlueALSA)

### 6-1. Bluetooth 기본 설정

```bash
cat <<EOF | sudo tee /etc/bluetooth/main.conf >/dev/null
[General]
NAME = "${DEVICE_NAME}"
Class = 0x200414
DiscoverableTimeout = 0
AlwaysPairable = true
PairableTimeout = 0
JustWorksRepairing = always

[Policy]
AutoEnable=true
EOF

sudo systemctl restart bluetooth
```

### 6-2. 자동 페어링 agent 작성

이 스크립트는:

- `org.bluez.Agent1` 사용
- `NoInputNoOutput` capability로 자동 승인
- 새 기기를 자동 trust
- 어댑터 alias를 `<기기명>`으로 설정
- discoverable / pairable 을 항상 켜 둠

```bash
cat <<EOF > "${BIN_HOME}/speaker-agent.py"
#!/usr/bin/env python3
import dbus
import dbus.service
import dbus.mainloop.glib
from gi.repository import GLib

BUS_NAME = "org.bluez"
AGENT_INTERFACE = "org.bluez.Agent1"
AGENT_PATH = "/dsp/agent"
DEVICE_NAME = "${DEVICE_NAME}"

bus = None

class Rejected(dbus.DBusException):
    _dbus_error_name = "org.bluez.Error.Rejected"

def set_trusted(path):
    try:
        props = dbus.Interface(
            bus.get_object(BUS_NAME, path),
            "org.freedesktop.DBus.Properties"
        )
        props.Set("org.bluez.Device1", "Trusted", dbus.Boolean(True))
    except Exception as exc:
        print(f"set_trusted failed for {path}: {exc}")

def setup_adapter():
    props = dbus.Interface(
        bus.get_object(BUS_NAME, "/org/bluez/hci0"),
        "org.freedesktop.DBus.Properties"
    )
    props.Set("org.bluez.Adapter1", "Powered", dbus.Boolean(True))
    props.Set("org.bluez.Adapter1", "Alias", dbus.String(DEVICE_NAME))
    props.Set("org.bluez.Adapter1", "PairableTimeout", dbus.UInt32(0))
    props.Set("org.bluez.Adapter1", "DiscoverableTimeout", dbus.UInt32(0))
    props.Set("org.bluez.Adapter1", "Pairable", dbus.Boolean(True))
    props.Set("org.bluez.Adapter1", "Discoverable", dbus.Boolean(True))

class Agent(dbus.service.Object):
    def __init__(self, bus, path):
        super().__init__(bus, path)

    @dbus.service.method(AGENT_INTERFACE, in_signature="", out_signature="")
    def Release(self):
        print("Agent released")

    @dbus.service.method(AGENT_INTERFACE, in_signature="o", out_signature="s")
    def RequestPinCode(self, device):
        set_trusted(device)
        print(f"RequestPinCode: {device}")
        return "0000"

    @dbus.service.method(AGENT_INTERFACE, in_signature="o", out_signature="u")
    def RequestPasskey(self, device):
        set_trusted(device)
        print(f"RequestPasskey: {device}")
        return dbus.UInt32(0)

    @dbus.service.method(AGENT_INTERFACE, in_signature="os", out_signature="")
    def DisplayPinCode(self, device, pincode):
        print(f"DisplayPinCode: {device} {pincode}")

    @dbus.service.method(AGENT_INTERFACE, in_signature="ouq", out_signature="")
    def DisplayPasskey(self, device, passkey, entered):
        print(f"DisplayPasskey: {device} {passkey} entered={entered}")

    @dbus.service.method(AGENT_INTERFACE, in_signature="ou", out_signature="")
    def RequestConfirmation(self, device, passkey):
        set_trusted(device)
        print(f"RequestConfirmation: {device} {passkey}")
        return

    @dbus.service.method(AGENT_INTERFACE, in_signature="o", out_signature="")
    def RequestAuthorization(self, device):
        set_trusted(device)
        print(f"RequestAuthorization: {device}")
        return

    @dbus.service.method(AGENT_INTERFACE, in_signature="os", out_signature="")
    def AuthorizeService(self, device, uuid):
        set_trusted(device)
        print(f"AuthorizeService: {device} {uuid}")
        return

    @dbus.service.method(AGENT_INTERFACE, in_signature="", out_signature="")
    def Cancel(self):
        print("Request canceled")

def register_agent():
    manager = dbus.Interface(
        bus.get_object(BUS_NAME, "/org/bluez"),
        "org.bluez.AgentManager1"
    )
    try:
        manager.UnregisterAgent(AGENT_PATH)
    except Exception:
        pass
    setup_adapter()
    manager.RegisterAgent(AGENT_PATH, "NoInputNoOutput")
    manager.RequestDefaultAgent(AGENT_PATH)
    print("AronaDSP Bluetooth agent registered")

def name_owner_changed(name, old_owner, new_owner):
    if name != BUS_NAME:
        return
    if new_owner:
        print("org.bluez appeared, registering agent")
        GLib.idle_add(register_agent)

def main():
    global bus
    dbus.mainloop.glib.DBusGMainLoop(set_as_default=True)
    bus = dbus.SystemBus()

    Agent(bus, AGENT_PATH)
    bus.add_signal_receiver(
        name_owner_changed,
        signal_name="NameOwnerChanged",
        dbus_interface="org.freedesktop.DBus",
        path="/org/freedesktop/DBus",
        arg0=BUS_NAME,
    )

    dbus_service = bus.get_object("org.freedesktop.DBus", "/org/freedesktop/DBus")
    dbus_dbus = dbus.Interface(dbus_service, "org.freedesktop.DBus")
    if dbus_dbus.NameHasOwner(BUS_NAME):
        register_agent()

    print("AronaDSP Bluetooth agent running")
    GLib.MainLoop().run()

if __name__ == "__main__":
    main()
EOF

chmod +x "${BIN_HOME}/speaker-agent.py"
```

### 6-3. 자동 페어링 agent systemd 서비스

```bash
cat <<EOF | sudo tee /etc/systemd/system/speaker-agent.service >/dev/null
[Unit]
Description=${DEVICE_NAME} Bluetooth pairing agent
After=bluetooth.service dbus.service
Requires=bluetooth.service dbus.service
PartOf=bluetooth.service

[Service]
Type=simple
ExecStart=/usr/bin/python3 ${BIN_HOME}/speaker-agent.py
Restart=always
RestartSec=2

[Install]
WantedBy=multi-user.target
EOF
```

### 6-4. AAC (`fdk-aac`)

```bash
cd "${SRC_HOME}"
rm -rf fdk-aac
git clone https://github.com/mstorsjo/fdk-aac.git
cd fdk-aac
autoreconf -fi
./configure --prefix=/usr/local --enable-shared --enable-static
make -j"${MAKE_JOBS}"
sudo make install
sudo ldconfig
```

### 6-5. aptX / aptX HD (`libopenaptx`)

```bash
cd "${SRC_HOME}"
rm -rf libopenaptx
git clone https://github.com/pali/libopenaptx.git
cd libopenaptx
make -j"${MAKE_JOBS}"
sudo make install
sudo ldconfig
```

LDAC은 source로만 작동하기에 여기선 추가하지 않았습니다.  


### 6-6. BlueALSA 빌드

```bash
export PKG_CONFIG_PATH="/usr/local/lib/pkgconfig:/usr/local/lib/aarch64-linux-gnu/pkgconfig:${PKG_CONFIG_PATH}"

cd "${SRC_HOME}"
rm -rf bluez-alsa
git clone --branch "${BLUEALSA_VER}" --depth 1 https://github.com/arkq/bluez-alsa.git
cd bluez-alsa
autoreconf --install --force
mkdir -p build
cd build

../configure \
  --prefix=/usr/local \
  --enable-systemd \
  --enable-cli \
  --enable-aac \
  --enable-aptx \
  --enable-aptx-hd \
  --with-libopenaptx

make -j"${MAKE_JOBS}"
sudo make install
sudo ldconfig
```
Bullseye 버전이라 BlueALSA를 사용합니다.

### 6-7. BlueALSA daemon 서비스

첫 연결에서 너무 큰 소리가 나지 않도록 `--initial-volume=50`을 사용합니다.  
이 값은 처음 연결 시에만 의미가 있고, 이후에는 기기에 저장된 이전 볼륨 값이 우선합니다.

```bash
cat <<'EOF' | sudo tee /etc/systemd/system/bluealsa.service >/dev/null
[Unit]
Description=BlueALSA service
Documentation=man:bluealsa(8)
Requisite=dbus.service
After=bluetooth.service
Requires=bluetooth.service

[Service]
Type=dbus
BusName=org.bluealsa
StateDirectory=bluealsa
StateDirectoryMode=0700
ExecStart=/usr/local/bin/bluealsa -S -p a2dp-sink --keep-alive=5 --initial-volume=50 -c aptx -c aptx-hd
Restart=on-failure

[Install]
WantedBy=bluetooth.target
EOF
```

### 6-8. Bluetooth → mixbus 브리지 서비스

`bluealsa-aplay`는 `--volume=software` 로 둡니다.

- ALSA mixer를 건드리지 않음
- source device별 저장 볼륨 때문

```bash
cat <<'EOF' | sudo tee /etc/systemd/system/bluealsa-aplay.service >/dev/null
[Unit]
Description=${DEVICE_NAME} BlueALSA to mixbus bridge
After=bluealsa.service bluetooth.service dbus.service sound.target
Requires=bluealsa.service bluetooth.service dbus.service

[Service]
Type=simple
ExecStartPre=/bin/sh -c 'for i in $(seq 1 20); do /usr/local/bin/bluealsa-cli status >/dev/null 2>&1 && exit 0; sleep 0.5; done; exit 1'
ExecStart=/usr/local/bin/bluealsa-aplay --pcm=mixbus --volume=software
Restart=always
RestartSec=2

[Install]
WantedBy=multi-user.target
EOF
```

### 6-9. Bluetooth 시작

```bash
sudo systemctl daemon-reload
sudo systemctl enable speaker-agent
sudo systemctl enable bluealsa
sudo systemctl enable bluealsa-aplay

sudo systemctl restart bluetooth
sudo systemctl start speaker-agent
sudo systemctl start bluealsa
sudo systemctl start bluealsa-aplay

sudo systemctl status bluealsa --no-pager -l
sudo systemctl status bluealsa-aplay --no-pager -l
sudo systemctl status speaker-agent --no-pager -l
```

---

### 6-10. BlueALSA 상태

```bash
bluealsa-cli status
```

```bash
bluealsa-cli list-pcms
```

예:

```text
/org/bluealsa/hci0/dev_XX_XX_XX_XX_XX_XX/a2dpsnk/source
```

특정 PCM 정보 보기

```bash
bluealsa-cli info /org/bluealsa/hci0/dev_XX_XX_XX_XX_XX_XX/a2dpsnk/source
```

여기서 볼 수 있는 것:

- Format
- Sampling
- Available codecs
- Selected codec
- SoftVolume
- Volume

> 실제로 해당 codec으로 연결되려면 **상대 기기도 그 codec을 지원**해야 합니다.

---

## 7. AirPlay / AirPlay 2 구성 (NQPTP + Shairport Sync)

### 7-1. ALAC 라이브러리 빌드

```bash
cd "${SRC_HOME}"
rm -rf alac
git clone --depth 1 https://github.com/mikebrady/alac.git
cd alac
autoreconf -fi
./configure --prefix=/usr/local
make -j"${MAKE_JOBS}"
sudo make install
sudo ldconfig
```

### 7-2. NQPTP 빌드 / 설치

```bash
cd "${SRC_HOME}"
rm -rf nqptp
git clone --branch "${NQPTP_VER}" --depth 1 https://github.com/mikebrady/nqptp.git
cd nqptp
autoreconf -fi
./configure --with-systemd-startup
make -j"${MAKE_JOBS}"
sudo make install
```

### 7-3. Shairport Sync 빌드 / 설치

```bash
cd "${SRC_HOME}"
rm -rf shairport-sync
git clone --branch "${SHAIRPORT_VER}" --depth 1 https://github.com/mikebrady/shairport-sync.git
cd shairport-sync
autoreconf -fi
./configure \
  --prefix=/usr/local \
  --sysconfdir=/etc \
  --with-alsa \
  --with-soxr \
  --with-avahi \
  --with-ssl=openssl \
  --with-systemd-startup \
  --with-airplay-2 \
  --with-apple-alac
make -j"${MAKE_JOBS}"
sudo make install
```

### 7-4. Shairport 설정

- `output_device = "mixbus";`


```bash
cat <<EOF | sudo tee /etc/shairport-sync.conf >/dev/null
general = {
  name = "${DEVICE_NAME}";
  output_backend = "alsa";
  interpolation = "soxr";
};

alsa = {
  output_device = "mixbus";
};

sessioncontrol = {
  session_timeout = 60;
};
EOF
```

### 7-5. AirPlay 시작

```bash
sudo systemctl daemon-reload

sudo systemctl enable avahi-daemon
sudo systemctl start avahi-daemon

sudo systemctl enable nqptp
sudo systemctl start nqptp

sudo systemctl enable shairport-sync
sudo systemctl start shairport-sync

sudo systemctl status avahi-daemon --no-pager -l
sudo systemctl status nqptp --no-pager -l
sudo systemctl status shairport-sync --no-pager -l
```

---

## 8. 전체 서비스 활성화

```bash
sudo systemctl daemon-reload

sudo systemctl enable \
  dsp-audio \
  uac2-mixbus \
  camilladsp \
  camillagui \
  speaker-agent \
  bluealsa \
  bluealsa-aplay \
  avahi-daemon \
  nqptp \
  shairport-sync
```

권장 재부팅:

```bash
sudo reboot
```

서비스 상태 확인:

```bash
systemctl status dsp-audio --no-pager -l
systemctl status uac2-mixbus --no-pager -l
systemctl status camilladsp --no-pager -l
systemctl status camillagui --no-pager -l
systemctl status bluetooth --no-pager -l
systemctl status speaker-agent --no-pager -l
systemctl status bluealsa --no-pager -l
systemctl status bluealsa-aplay --no-pager -l
systemctl status avahi-daemon --no-pager -l
systemctl status nqptp --no-pager -l
systemctl status shairport-sync --no-pager -l
```

장치 확인:

```bash
aplay -l
aplay -L
arecord -l
arecord -L
```

로그 확인:

```bash
journalctl -u dsp-audio -b --no-pager
journalctl -u uac2-mixbus -b --no-pager
journalctl -u camilladsp -b --no-pager
journalctl -u camillagui -b --no-pager
journalctl -u bluetooth -b --no-pager
journalctl -u speaker-agent -b --no-pager
journalctl -u bluealsa -b --no-pager
journalctl -u bluealsa-aplay -b --no-pager
journalctl -u avahi-daemon -b --no-pager
journalctl -u nqptp -b --no-pager
journalctl -u shairport-sync -b --no-pager
```

---
## 9. CamillaDSP Web GUI 연결
인터넷 창을 열고 (같은 공유기에서):
```text
<설정한 DSP 이름 또는 IP주소>:1215
예: yuushadsp:1215
```
---
## 10. 참고 링크

### 공식
- CamillaDSP 릴리스: https://github.com/HEnquist/camilladsp/releases
- CamillaGUI backend 릴리스: https://github.com/HEnquist/camillagui-backend/releases
- CamillaDSP websocket 문서: https://github.com/HEnquist/camilladsp/blob/master/websocket.md
- BlueALSA README: https://github.com/arkq/bluez-alsa
- BlueALSA INSTALL(v4.3.1): https://github.com/arkq/bluez-alsa/blob/v4.3.1/INSTALL.md
- BlueALSA `bluealsa` 문서(v4.3.1): https://github.com/arkq/bluez-alsa/blob/v4.3.1/doc/bluealsa.8.rst
- BlueALSA `bluealsa-aplay` 문서(v4.3.1): https://github.com/arkq/bluez-alsa/blob/v4.3.1/doc/bluealsa-aplay.1.rst
- BlueZ Agent API: https://bluez.readthedocs.io/en/latest/agent-api/
- BlueZ Adapter API: https://bluez.readthedocs.io/en/latest/adapter-api/
- Shairport Sync 릴리스: https://github.com/mikebrady/shairport-sync/releases
- Shairport Sync 샘플 설정: https://github.com/mikebrady/shairport-sync/blob/master/scripts/shairport-sync.conf
- NQPTP 릴리스: https://github.com/mikebrady/nqptp/releases
