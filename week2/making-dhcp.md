## #2 DHCP 서버를 만들어보자.

현재 proxmox 위에 여러개의 vm 이 올라가있는 상태인데, 
그중 하나의 vm은 DHCP서버,

또 다른 하나의 vm 은 클라이언트로서 
ip할당요청 -> DHCP서버 -> ip할당됨
의 일련의 과정을 직접 실현시켜보는것이 이번주차의 목표였다.
---

## 1. 배경 지식

### 1. 네트워크 모드

vmware 나 proxmox같은 가상화 툴에는 네트워크 모드라는것이 있다.

1. Host-Only : 외부와 단절된 내부 네트워크를 구축하는 것으로, 구성된 가상머신 즉 vm등의 가상 머신 내에 존재하는 머신들 끼리만 통신이 가능함.
2. NAT : 호스트 pc로부터 ip를 할당 받아 가상머신 프로그램이 자체 DHCP서버를 띄워 내부 네트워크 대역 할당 및 통신, 해당 호스트 PC를 이용하여 외부 네트워크와 통신이 가능함.
3. Bridge : 공유기로부터 IP를 할당 받아, 호스트PC와 동일한 네트워크 대역의 IP를 갖게됨, 공유기를 통해 외부 네트워크통신이 가능.

> [출처](https://hg2lee.tistory.com/entry/Network-%EA%B0%80%EC%83%81%EB%A8%B8%EC%8B%A0%EC%97%90%EC%84%9C-%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC%EC%9D%98-%EC%A2%85%EB%A5%98-Bridged-NAT-Host-onl)

### 2. 이건 그냥 면접에 나왔어서 정리하는.. (라우터 - PC 랜포트 연결된 장치간 통신은 어떻게?)
라우터에 연결된 PC가 **LAN(Local Area Network) 통신**을 수행하는 과정은 **L2(데이터 링크 계층, MAC)와 L3(네트워크 계층, IP)의 패킷 전달 과정**을 통해 이루어집니다. 이를 단계별로 설명하겠습니다.  

---

## **1. LAN 통신 개념**
- **LAN 통신:** 같은 네트워크에 있는 장치들끼리 직접 통신하는 방식.  
- **패킷 전달:** 송신 장치가 **MAC 주소**를 이용해 수신 장치를 찾고, IP 주소를 사용하여 데이터를 전송함.  
- **라우터 역할:** 라우터는 **다른 네트워크(서브넷) 간의 통신을 중계**하지만, 같은 LAN에서는 **스위치 역할**을 수행하여 패킷을 전달함.

---

## **2. LAN 통신의 패킷 동작 과정**
예를 들어, **PC1(192.168.1.10)이 PC2(192.168.1.20)에게 데이터를 전송**한다고 가정하겠습니다.

### **① 목적지 MAC 주소 확인 (ARP 요청)**
PC1이 PC2의 **MAC 주소를 모르면** 먼저 **ARP(Address Resolution Protocol) 요청**을 보냅니다.

**ARP 요청 과정**
- PC1은 **브로드캐스트(FF:FF:FF:FF:FF:FF) 패킷**을 사용하여 "192.168.1.20의 MAC 주소를 알려줘!"라고 요청.
- 이 패킷은 같은 네트워크에 있는 모든 장치에게 전달됨.

📌 **ARP 요청 패킷 구조**
```
출발지 MAC: 00:11:22:33:44:55 (PC1 MAC)
목적지 MAC: FF:FF:FF:FF:FF:FF (브로드캐스트)
출발지 IP: 192.168.1.10
목적지 IP: 192.168.1.20
```

### **② 목적지 PC가 응답 (ARP 응답)**
- PC2는 자신의 IP가 192.168.1.20이라는 것을 확인하고, **자신의 MAC 주소를 포함한 ARP 응답**을 보냄.
- 응답은 **유니캐스트(PC1의 MAC 주소)**로 전송됨.

📌 **ARP 응답 패킷 구조**
```
출발지 MAC: 00:AA:BB:CC:DD:EE (PC2 MAC)
목적지 MAC: 00:11:22:33:44:55 (PC1 MAC)
출발지 IP: 192.168.1.20
목적지 IP: 192.168.1.10
```

- PC1은 이제 **PC2의 MAC 주소를 ARP 캐시(임시 저장소)에 저장**하고, 이후부터는 ARP 없이 바로 통신 가능.

---

### **③ 데이터 패킷 전송 (이더넷 프레임)**
이제 PC1은 **PC2에게 실제 데이터를 전송**할 수 있습니다.

**📌 이더넷 프레임 구조**
```
출발지 MAC: 00:11:22:33:44:55 (PC1)
목적지 MAC: 00:AA:BB:CC:DD:EE (PC2)
출발지 IP: 192.168.1.10
목적지 IP: 192.168.1.20
데이터: "안녕, PC2!"
```
- 패킷이 라우터(또는 스위치)로 전달됨.
- 라우터(스위치)는 목적지 MAC이 PC2의 MAC임을 확인하고 **PC2로 전달**.

---

### **④ PC2가 데이터 수신 및 응답**
PC2는 받은 패킷을 확인하고, 응답 패킷을 생성하여 다시 PC1에게 보냄.

📌 **응답 패킷**
```
출발지 MAC: 00:AA:BB:CC:DD:EE (PC2)
목적지 MAC: 00:11:22:33:44:55 (PC1)
출발지 IP: 192.168.1.20
목적지 IP: 192.168.1.10
데이터: "안녕, PC1!"
```

- 이 과정이 빠르게 반복되면서 LAN 통신이 이루어짐.

---

## **3. 라우터의 역할**
- 라우터는 **같은 네트워크(LAN) 내에서는 패킷을 중계하지 않음.**  
  → 스위치 또는 허브가 MAC 주소 기반으로 직접 전달.
- 다만, **다른 네트워크(서브넷)와 통신할 때**는 라우터가 개입하여 **게이트웨이 역할**을 수행.

📌 **LAN 내부 통신과 라우터**  
✅ **LAN 내부 (같은 서브넷) → 라우터 불필요**  
✅ **다른 서브넷 (192.168.1.10 → 192.168.2.10) → 라우터 필요**  

---

## **4. 전체 LAN 통신 흐름 요약**
1. **PC1 → PC2의 MAC 주소 확인 (ARP 요청)**
2. **PC2가 자신의 MAC 주소를 응답 (ARP 응답)**
3. **PC1이 MAC 주소를 이용해 데이터 전송 (이더넷 프레임)**
4. **PC2가 데이터 수신 후 응답**
5. **라우터는 같은 네트워크에서는 개입하지 않음, 다른 네트워크에서는 중계**

---

## **📌 결론**
라우터에 연결된 PC들이 **같은 네트워크에서 통신할 때**는 **MAC 주소를 기반으로 직접 통신**하며, 라우터는 개입하지 않습니다. 하지만 **다른 네트워크와 통신할 때**는 라우터가 패킷을 전달하는 역할을 합니다.  

---
## 2. 만들어보자.

DHCP server - host only adaptor 카드 2개가 필요하다.   (클라이언트와 같은거, 인터넷이 되는거)

클라이언트 - host only adaptor 카드1개 로 구성되어야한다.

<details>
    <summary> gpt와의 이야기 </summary>
    ---

### 디폴트 설정과 폐쇄망 만드는방법

컴퓨터 하나당 여러개의 랜포트를 만들 수 있다.

---

네, 맞습니다! **DHCP 서버 역할을 할 VM에는 반드시 고정 IP를 할당해야 합니다.**

DHCP 서버가 IP를 제공하려면 **자신의 IP가 변하지 않아야 하기 때문**입니다.

---

### **✅ 1️⃣ DHCP 서버가 될 VM의 고정 IP 할당하기**

고정 IP를 설정하려면 VM의 네트워크 설정을 변경해야 합니다.

### **📌 Debian/Ubuntu (Netplan 사용)**

```bash
sudo nano /etc/netplan/01-netcfg.yaml

```

다음 설정을 추가하거나 수정합니다.

```yaml
network:
  version: 2
  ethernets:
    ens18:
      dhcp4: no
      addresses:
        - 192.168.1.100/24  # DHCP 서버의 고정 IP
      gateway4: 192.168.1.1  # 라우터 주소
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4

```

📌 **설명:**

- `dhcp4: no` → DHCP 사용 안 함 (고정 IP 사용)
- `addresses` → DHCP 서버가 사용할 **고정 IP (예: 192.168.1.100)**
- `gateway4` → 기본 게이트웨이 (라우터 IP)
- `nameservers` → DNS 서버 설정

**설정 적용:**

```bash
sudo netplan apply

```

---

### **📌 Debian/Ubuntu (기존 인터페이스 설정 방식)**

만약 `/etc/network/interfaces` 파일을 사용하는 경우:

```bash
sudo nano /etc/network/interfaces

```

설정을 추가합니다.

```
auto ens18
iface ens18 inet static
    address 192.168.1.100  # DHCP 서버의 고정 IP
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 8.8.4.4

```

설정 적용:

```bash
sudo systemctl restart networking

```

---

### **✅ 2️⃣ DHCP 서버가 VM 내부에서 동작하는지 확인하기**

1. **IP 확인**
    
    ```bash
    ip a show ens18
    
    ```
    
    → `192.168.1.100/24` 가 표시되는지 확인
    
2. **DHCP 서비스 실행 상태 확인 (예: isc-dhcp-server 사용 시)**
    
    ```bash
    systemctl status isc-dhcp-server
    
    ```
    
    → 실행 중이어야 함 (`active (running)`)
    

---

### **🚀 결론**

✔ DHCP 서버가 될 VM에는 **고정 IP를 할당해야 함.**

✔ 설정 방법:

- **Netplan** (`/etc/netplan/01-netcfg.yaml`)
- **기존 방식** (`/etc/network/interfaces`)
✔ 설정 후 **`ip a`로 고정 IP가 적용되었는지 확인** 필요.

필요한 추가 설정이 있으면 알려주세요! 😊

네! 이미 **VM이 192.168.0.139**로 IP를 할당받고 있다면, **그걸 그대로 사용하면 됩니다.**

다만, **DHCP 서버가 실행 중인 동안 이 IP가 바뀌지 않도록 고정해야 합니다.**

---

## **✅ 1️⃣ 현재 IP가 고정인지 확인하는 방법**

### **방법 1: IP 설정 파일 확인 (`/etc/network/interfaces` 또는 Netplan)**

- 만약 **DHCP에서 자동 할당된 것**이라면, 시간이 지나면 IP가 바뀔 수 있음.
- 고정 IP라면 그대로 사용하면 됨.

```bash
ip a show ens18

```

만약 `inet 192.168.0.139/24`가 표시되면, **현재 이 IP를 사용 중인 것**입니다.

---

## **✅ 2️⃣ IP를 완전히 고정하는 방법**

### **📌 방법 1: DHCP 서버에서 IP 예약하기** (권장)

- **라우터(DHCP 서버)에서** MAC 주소를 기반으로 **192.168.0.139를 항상 할당하도록 예약**하면 됨.
- 라우터 설정 페이지에서 **"DHCP IP 예약"** 또는 **"Static Lease"** 같은 기능을 찾아 MAC 주소를 입력하면 됨.
- MAC 주소 확인 방법:

또는
    
    ```bash
    ip link show ens18
    
    ```
    
    ```bash
    cat /sys/class/net/ens18/address
    
    ```
    
- 이 방법을 사용하면 **VM 설정을 바꾸지 않고도 IP가 고정됨.**

---

### **📌 방법 2: 직접 고정 IP 설정하기 (VM 내부에서 설정)**

```bash
sudo nano /etc/network/interfaces

```

아래처럼 설정을 변경합니다.

```
auto ens18
iface ens18 inet static
    address 192.168.0.139
    netmask 255.255.255.0
    gateway 192.168.0.1
    dns-nameservers 8.8.8.8 8.8.4.4

```

**설정 적용:**

```bash
sudo systemctl restart networking

```

---

## **🚀 결론**

✔ **이미 할당된 192.168.0.139를 그대로 사용해도 됨.**

✔ **IP가 바뀌지 않도록 DHCP에서 예약하거나, 직접 고정 IP를 설정하면 안전함.**

✔ 라우터 설정을 바꿀 수 있으면 **DHCP 예약 방식**이 더 편하고 안정적!

**추가로 궁금한 점 있으면 알려주세요!** 😊

네! Netplan을 사용해서 **192.168.0.139**를 고정 IP로 설정하려는 거네요.

설정 자체는 **올바르게 작성**되었습니다. 🚀

---

## **✅ 1️⃣ 적용 방법**

### **📌 설정 파일 저장 후 Netplan 적용**

```bash
sudo netplan apply

```

적용 후 **인터페이스가 정상적으로 설정되었는지 확인**하세요.

```bash
ip a show ens18

```

→ `inet 192.168.0.139/24`가 정상적으로 설정되었는지 확인.

---

## **✅ 2️⃣ 네트워크 정상 작동 확인**

### **📌 게이트웨이 연결 확인 (192.168.0.1로 핑 테스트)**

```bash
ping -c 4 192.168.0.1

```

✔ 응답이 오면 **게이트웨이와 연결 정상!**

### **📌 인터넷 연결 확인**

```bash
ping -c 4 8.8.8.8

```

✔ 응답이 오면 **인터넷 정상!**

✔ 만약 안 된다면, `routes` 부분을 다시 확인해봐야 함.

---

## **🚀 결론**

✔ 설정 파일은 **문제없이 잘 작성됨!**

✔ `sudo netplan apply` 실행 후 **IP 설정 확인!**

✔ `ping 192.168.0.1`과 `ping 8.8.8.8`로 연결 확인!

**추가 질문 있으면 편하게 물어봐 주세요! 😊**

Netplan 적용 시 **"Failed to reload network settings: No such file or directory"** 오류가 발생하는군요.

이는 **Netplan이 올바르게 적용되지 않거나, systemd-networkd가 활성화되지 않은 경우** 발생할 수 있습니다.

---

## **✅ 1️⃣ Netplan 설정 파일 문법 확인**

먼저 **Netplan 설정이 올바른지 확인**하세요.

```bash
sudo netplan try

```

✔ 오류가 없으면 적용됩니다.

✔ 오류가 있다면 수정해야 합니다.

혹시 YAML 문법 문제일 수도 있으니, 아래 명령어로 **들여쓰기 오류가 있는지 체크**하세요.

```bash
sudo netplan generate

```

✔ 에러 메시지가 없으면 문법 문제 없음.

✔ **에러가 나오면 해당 부분 수정!**

---

## **✅ 2️⃣ Netplan 적용 및 강제 재시작**

만약 여전히 `netplan apply`가 안 된다면, **시스템 네트워크 서비스를 강제로 재시작**합니다.

```bash
sudo systemctl restart systemd-networkd

```

그리고 다시 `netplan apply` 시도:

```bash
sudo netplan apply

```

---

## **✅ 3️⃣ NetworkManager가 Netplan을 방해하는 경우**

혹시 **NetworkManager가 사용 중이라면**, Netplan이 제대로 적용되지 않을 수도 있습니다.

확인 방법:

```bash
sudo systemctl status NetworkManager

```

✔ 실행 중이라면, **NetworkManager 대신 systemd-networkd 사용**하도록 변경해야 할 수도 있음.

만약 Netplan을 강제로 적용하려면:

```bash
sudo systemctl stop NetworkManager
sudo systemctl disable NetworkManager
sudo systemctl restart systemd-networkd
sudo netplan apply

```

---

## **🚀 결론**

✔ **1️⃣ `sudo netplan try`로 문법 확인**

✔ **2️⃣ `sudo netplan generate`로 설정 파일 체크**

✔ **3️⃣ `systemd-networkd` 재시작 후 `netplan apply`**

✔ **4️⃣ NetworkManager가 방해하는지 확인 (필요 시 비활성화)**

이대로 해보고 해결 안 되면 추가 로그와 함께 다시 질문 주세요! 😊

Netplan 설정을 적용하는 과정에서 **파일 권한 문제 및 OVS(Open vSwitch) 관련 경고**가 발생하고 있습니다.

각 문제를 하나씩 해결해볼게요! 🚀

---

## **✅ 1️⃣ Netplan 설정 파일의 권한 문제 해결**

에러 메시지에서 **`/etc/netplan/01-network-manager-all.yaml` 파일의 권한이 너무 개방적**이라고 나옵니다.

Netplan 설정 파일은 보안상 **root만 읽을 수 있도록 제한**해야 합니다.

### **📌 파일 권한을 수정**

```bash
sudo chmod 600 /etc/netplan/01-network-manager-all.yaml

```

✔ 이렇게 하면 **root만 읽을 수 있도록 설정됨.**

✔ 다시 Netplan을 시도하세요:

```bash
sudo netplan generate
sudo netplan apply

```

---

## **✅ 2️⃣ Open vSwitch(OVS) 관련 경고 해결**

경고 메시지 중 **`Cannot call Open vSwitch: ovsdb-server.service is not running.`**

이건 Open vSwitch(OVS)가 실행되지 않아서 발생하는 문제입니다.

만약 OVS가 필요 없다면 **무시해도 됩니다.**

### **📌 OVS가 필요 없다면 서비스 비활성화**

```bash
sudo systemctl disable openvswitch-switch
sudo systemctl stop openvswitch-switch

```

이제 다시 `netplan apply` 실행.

### **📌 OVS를 사용해야 한다면 서비스 실행**

```bash
sudo systemctl start openvswitch-switch
sudo systemctl enable openvswitch-switch

```

이후 다시 Netplan 적용:

```bash
sudo netplan apply

```

---

## **🚀 결론**

✔ **1️⃣ Netplan 설정 파일 권한을 `chmod 600`으로 변경**

✔ **2️⃣ OVS 관련 경고가 필요 없다면 비활성화 (`systemctl disable openvswitch-switch`)**

✔ **3️⃣ 모든 설정을 적용 후 `sudo netplan apply` 다시 실행**

이제 네트워크가 정상 작동할 가능성이 높습니다!

혹시 추가 오류가 나오면 로그와 함께 다시 질문 주세요. 😊

좋아요! 지금 **DHCP 서버 설정**까지 완료했네요. 🚀

이제 **DHCP 서버를 실행하고 클라이언트가 정상적으로 IP를 할당받는지 확인**해야 합니다.

---

## **✅ 1️⃣ DHCP 서버 실행**

DHCP 서버를 재시작해서 설정을 적용하세요.

```bash
sudo systemctl restart isc-dhcp-server

```

그리고 서비스 상태 확인:

```bash
sudo systemctl status isc-dhcp-server

```

✔ **Active (running)** 상태인지 확인!

✔ 만약 에러가 발생하면 **로그 확인**:

```bash
sudo journalctl -xe | grep dhcp

```

---

## **✅ 2️⃣ DHCP 서버가 올바른 인터페이스에서 동작하는지 확인**

### **📌 DHCP 서버가 사용할 인터페이스 설정**

`/etc/default/isc-dhcp-server` 파일을 열어서 `INTERFACESv4` 값을 확인하세요.

```bash
sudo nano /etc/default/isc-dhcp-server

```

아래처럼 `ens18`을 설정해야 합니다. (VM의 인터페이스 이름이 `ens18`이 맞다면)

```
INTERFACESv4="ens18"

```

저장 후 다시 **DHCP 서버 재시작**

```bash
sudo systemctl restart isc-dhcp-server

```

---

## **✅ 3️⃣ DHCP 서버가 정상적으로 IP를 할당하는지 확인**

### **📌 DHCP 서버 로그 확인**

아래 명령어로 DHCP 요청 로그를 확인하세요.

```bash
sudo journalctl -u isc-dhcp-server --no-pager | grep DHCPACK

```

✔ 만약 클라이언트가 `DHCPACK`을 받았다면 정상적으로 IP를 할당한 것!

### **📌 클라이언트에서 수동으로 DHCP 요청 보내기**

DHCP 클라이언트가 정상적으로 작동하는지 테스트하려면, DHCP를 받는 VM에서 아래 명령어 실행:

```bash
sudo dhclient -r
sudo dhclient

```

✔ `ip a` 명령어를 실행해서 **IP가 192.168.0.140~200 중 하나로 할당되었는지 확인**

---

## **🚀 결론**

✔ **1️⃣ DHCP 서버 재시작 (`systemctl restart isc-dhcp-server`)**

✔ **2️⃣ `INTERFACESv4="ens18"` 설정 확인 후 다시 시작**

✔ **3️⃣ DHCP 서버 로그 확인 (`journalctl -u isc-dhcp-server`)**

✔ **4️⃣ 클라이언트에서 `sudo dhclient`로 IP 받기 테스트**

이제 **클라이언트가 정상적으로 IP를 할당받는지 확인**해보세요!

추가 문제 있으면 **오류 메시지와 함께 질문 주세요! 😊**

## **🔧 DHCP 클라이언트 설정 방법**

지금 DHCP 서버가 정상적으로 동작하는지 확인한 후, 클라이언트(다른 VM 또는 PC)가 DHCP를 통해 **자동으로 IP를 할당받도록 설정**해야 합니다.

### **📌 1️⃣ 클라이언트 네트워크 설정 확인**

먼저 클라이언트에서 현재 네트워크 인터페이스를 확인하세요.

```bash
ip a

```

✔ `ens18` 또는 `eth0`과 같은 네트워크 인터페이스 이름을 확인하세요.

---

### **📌 2️⃣ 클라이언트를 DHCP 모드로 변경**

DHCP를 사용하도록 클라이언트의 네트워크 설정을 변경해야 합니다.

### **🔹 (Ubuntu / Debian) Netplan 사용 시**

클라이언트의 `/etc/netplan/` 아래에 있는 `.yaml` 파일을 수정합니다.

예를 들어, `/etc/netplan/50-cloud-init.yaml`이 있다면 다음처럼 설정:

```yaml
network:
  ethernets:
    ens18:
      dhcp4: true
  version: 2

```

✔ `dhcp4: true`를 추가한 후 저장하고 적용:

```bash
sudo netplan apply

```

---

### **🔹 (Ubuntu / Debian) `interfaces` 파일 사용 시**

만약 **Netplan이 아니라 `/etc/network/interfaces`를 사용하는 경우**

아래 설정을 적용:

```bash
sudo nano /etc/network/interfaces

```

다음과 같이 인터페이스를 DHCP 모드로 설정:

```
auto ens18
iface ens18 inet dhcp

```

✔ 저장 후 적용:

```bash
sudo systemctl restart networking

```

---

### **📌 3️⃣ 수동으로 DHCP 요청 보내기 (테스트)**

설정이 끝났다면, 클라이언트에서 **수동으로 DHCP 요청을 보내** IP를 할당받을 수 있습니다.

```bash
sudo dhclient -r
sudo dhclient

```

✔ `ip a` 명령어로 **IP가 DHCP 범위(예: 192.168.0.140~200)에서 정상적으로 할당되었는지 확인**

---

### **📌 4️⃣ 클라이언트가 정상적으로 IP를 받았는지 확인**

아래 명령어를 실행해서 현재 네트워크 설정을 확인하세요.

```bash
ip a
ip r

```

✔ **192.168.0.x 범위의 IP가 할당되었는지 확인!**

✔ 기본 게이트웨이가 `192.168.0.1`로 설정되었는지 확인!

---

## **🚀 결론**

✔ **1️⃣ 클라이언트의 네트워크 인터페이스 확인 (`ip a`)**

✔ **2️⃣ Netplan (`dhcp4: true`) 또는 `interfaces` 파일 수정 후 적용**

✔ **3️⃣ DHCP 요청 수동 갱신 (`sudo dhclient`)**

✔ **4️⃣ `ip a`로 정상적으로 IP 할당되었는지 확인**

이제 클라이언트가 DHCP를 통해 자동으로 IP를 받았는지 확인해보세요!

추가 문제가 있으면 **오류 메시지와 함께 질문 주세요! 😊**

### **🚨 `error: ipv4 address already assigned` 해결 방법**

이 오류는 **클라이언트가 이미 IP를 할당받았는데, 다시 요청하려고 할 때 발생**하는 문제입니다.

기존 IP 할당을 해제한 후 다시 DHCP 요청하면 해결할 수 있습니다.

---

## **✅ 해결 방법 1: 기존 DHCP 리스 해제 후 다시 요청**

먼저 기존 DHCP 설정을 해제한 후 다시 요청하세요.

```bash
sudo dhclient -r ens18   # 기존 DHCP IP 해제
sudo dhclient ens18      # 새로운 DHCP 요청

```

✔ `ens18` 대신 클라이언트의 실제 네트워크 인터페이스 이름(`eth0` 등)을 사용하세요.

✔ `ip a`로 새로운 IP가 할당되었는지 확인!

---

## **✅ 해결 방법 2: DHCP 리스 파일 삭제 후 재요청**

DHCP 클라이언트가 이전의 IP를 저장하고 있을 수도 있습니다.

해당 파일을 삭제한 후 다시 DHCP 요청하면 해결됩니다.

```bash
sudo rm /var/lib/dhcp/dhclient.*
sudo systemctl restart networking
sudo dhclient ens18

```

✔ `/var/lib/dhcp/dhclient.*`에는 이전의 IP 정보가 저장되므로 삭제하면 새롭게 요청할 수 있습니다.

---

## **✅ 해결 방법 3: 네트워크 인터페이스 리셋**

네트워크 인터페이스를 **완전히 초기화한 후 다시 DHCP 요청**할 수도 있습니다.

```bash
sudo ip link set ens18 down
sudo ip addr flush dev ens18
sudo ip link set ens18 up
sudo dhclient ens18

```

✔ `ip addr flush dev ens18` 명령어는 **현재 할당된 IP를 제거**하고 새롭게 DHCP 요청할 수 있도록 합니다.

✔ 이후 `ip a`로 새로운 IP가 정상적으로 할당되었는지 확인하세요.

---

## **🚀 결론**

✔ **1️⃣ `sudo dhclient -r ens18`로 기존 DHCP 설정 해제 후 다시 요청**

✔ **2️⃣ `/var/lib/dhcp/dhclient.*` 파일 삭제 후 다시 DHCP 요청**

✔ **3️⃣ `ip addr flush dev ens18`로 IP 제거 후 새롭게 요청**

위 방법을 순서대로 시도해보고, 여전히 문제가 있다면 **오류 메시지와 함께 질문 주세요! 😊**

### **🚨 `networking.service` not found 해결 방법**

**Ubuntu 18.04 이상**에서는 `networking.service`가 기본적으로 사용되지 않고, `systemd-networkd` 또는 `NetworkManager`가 대신 사용됩니다.

---

## **✅ 해결 방법 1: Netplan 재적용**

Ubuntu에서 **Netplan을 사용 중이라면**, 네트워크 설정을 다시 적용하세요.

```bash
sudo netplan apply

```

✔ 이후 `ip a`로 IP가 정상적으로 설정되었는지 확인하세요.

---

## **✅ 해결 방법 2: `systemd-networkd` 재시작 (Netplan 기반)**

만약 `systemd-networkd`를 사용하고 있다면 아래 명령어로 네트워크를 재시작하세요.

```bash
sudo systemctl restart systemd-networkd

```

✔ 이후 다시 `dhclient`로 DHCP 요청:

```bash
sudo dhclient -r ens18
sudo dhclient ens18

```

---

## **✅ 해결 방법 3: `NetworkManager`를 사용하는 경우**

만약 `NetworkManager`를 사용 중이라면 다음 명령어를 실행하세요.

```bash
sudo systemctl restart NetworkManager

```

✔ 이후 `ip a`로 확인하고, 필요하면 수동으로 DHCP 요청:

```bash
sudo dhclient -r ens18
sudo dhclient ens18

```

---

## **🚀 결론**

1️⃣ **Netplan 사용 시** → `sudo netplan apply`

2️⃣ **systemd-networkd 사용 시** → `sudo systemctl restart systemd-networkd`

3️⃣ **NetworkManager 사용 시** → `sudo systemctl restart NetworkManager`

이후 `sudo dhclient -r ens18 && sudo dhclient ens18` 실행해서 다시 DHCP 요청하면 됩니다!

혹시 문제가 지속되면 **오류 메시지와 함께 질문 주세요! 😊**

### **🚀 특정 출발지 IP 또는 인터페이스를 지정하여 `ping` 보내는 방법**

출발지(SRC) IP를 지정해서 `ping`을 보내려면 `-I` 또는 `-S` 옵션을 사용하면 됩니다.

---

## **✅ 방법 1: 출발지 인터페이스 지정 (`I`)**

특정 네트워크 인터페이스(예: `ens18`)를 통해 `ping`을 보낼 수 있습니다.

```bash
ping -I ens18 8.8.8.8

```

✔ `ens18` 대신 **출발지 인터페이스 이름**을 입력하세요.

✔ `ip a` 명령어로 사용 가능한 인터페이스 목록을 확인할 수 있습니다.

---

## **✅ 방법 2: 출발지 IP 주소 지정 (`I` 또는 `S`)**

출발지 IP 주소(예: `192.168.0.139`)를 명시적으로 지정해서 `ping`을 보낼 수도 있습니다.

```bash
ping -I 192.168.0.139 8.8.8.8

```

✔ `-I` 옵션은 리눅스에서 출발지 IP를 지정할 때 사용됩니다.

✔ 일부 OS에서는 `-S` 옵션을 지원하기도 합니다. (`-S`는 일반적으로 Windows 및 BSD 계열에서 사용)

✔ **출발지 IP는 네트워크 인터페이스에 바인딩된 IP여야 합니다.**

---

## **✅ 방법 3: Windows에서 출발지 IP 지정 (`S`)**

Windows에서는 `-S` 옵션을 사용하여 출발지 IP를 지정할 수 있습니다.

```powershell
ping -S 192.168.0.139 8.8.8.8

```

✔ **Windows는 `-S` 옵션을 사용**하고, **Linux는 `-I` 옵션을 사용**합니다.

---

## **🚀 결론**

- **출발지 인터페이스 지정** → `ping -I ens18 대상IP`
- **출발지 IP 지정 (Linux)** → `ping -I 192.168.0.139 대상IP`
- **출발지 IP 지정 (Windows)** → `ping -S 192.168.0.139 대상IP`

필요한 방법을 사용하면 됩니다! 😊 🚀
</details>

여튼 위 명령어들을 이용해서, dhcp관련 netplan 하위 컨피그 파일을 apply하고 클라이언트 측에서 ip를 재할당받을 수 있다.

