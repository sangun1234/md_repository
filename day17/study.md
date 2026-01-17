# OSPF 필기 및 트러블 슈팅

## 1️⃣ 라우터 초기 설정

### R1

```ios
en
conf t
hostname R1
no ip domain lookup
line console 0
logging sync
exec-timeout 0 0
exit
interface Loopback0
ip address 10.10.8.1 255.255.255.0
no shutdown
exit
interface Loopback1
ip address 10.10.9.1 255.255.255.0
no shutdown
exit
interface g0/0
ip add 10.10.0.1 255.255.255.248
no sh
exit
```

### R2

```ios
enable
conf t
hostname R2
no ip domain lookup
line console 0
logging sync
exec-timeout 0 0
exit
interface Loopback 0
ip address 209.165.200.225 255.255.255.224
no shutdown
exit
interface Loopback 1
ip address 192.168.1.1 255.255.255.192
no shutdown
exit
interface g0/0
ip add 10.10.0.2 255.255.255.248
no sh
exit
```

### R3

```ios
enable
conf t
hostname R3
no ip domain lookup
line console 0
logging sync
exec-timeout 0 0
exit
interface Loopback 0
ip address 10.10.24.1 255.255.255.0
no shutdown
exit
interface Loopback 1
ip address 10.10.25.1 255.255.255.192
no shutdown
exit
interface g0/0
ip add 10.10.0.3 255.255.255.248
no sh
exit
```

## 2️⃣ OSPF 구현하기 (Single Area)


### R1

```ios
router ospf 7
network 10.10.0.1 0.0.0.0 area 0
network 10.10.8.1 0.0.0.0 area 0
network 10.10.9.1 0.0.0.0 area 0
end
```

#### 검증 명령어: `show ip protocols`

```ios
Routing Protocol is "ospf 7"
Outgoing update filter list for all interfaces is not set
Incoming update filter list for all interfaces is not set
Router ID 10.10.9.1
Number of areas in this router is 1. 1 normal 0 stub 0 nssa
Maximum path: 4
Routing for Networks:
  10.10.0.1 0.0.0.0 area 0
  10.10.8.1 0.0.0.0 area 0
  10.10.9.1 0.0.0.0 area 0
Routing Information Sources:
  Gateway         Distance      Last Update
Distance: (default is 110)
```

**📝 설명:**
- Router ID가 `10.10.9.1`로 자동 설정됨
- 수동 설정 없으면 Loopback 중 가장 큰 IP가 Router ID로 사용됨


### R2

```ios
router ospf 7
router-id 209.165.200.225
network 10.10.0.0 0.0.0.7 area 0
network 192.168.1.0 0.0.0.63 area 0
network 209.165.200.225 0.0.0.31 area 0
exit
```

**📝 설명:**
- `/29` 비트 마스크로 `10.10.0.0 ~ 10.10.0.7` 범위 선언
- R1의 와일드카드 `0.0.0.0`(정확히 한 호스트만)과는 다름
- Longest Match 규칙에 의해 R3에서 `10.10.0.1`로 ping 시 R1로 우선 라우팅됨

### R3

```ios
(config)
router ospf 10
router-id 10.10.24.1
network 10.10.24.1 0.0.0.0 area 0
network 10.10.25.1 0.0.0.0 area 0
network 10.10.0.3 0.0.0.255 area 0
end
```

**⚠️ 주의사항:**
- `/29` 대역에 와일드카드 `0.0.0.255`를 적용 (과도하게 넓음)
- 확인 명령어: `show ip ospf neighbor`

```ios
Routing Protocol is "ospf 10"
  Outgoing update filter list for all interfaces is not set
  Incoming update filter list for all interfaces is not set
  Router ID 10.10.24.1
  Number of areas in this router is 1. 1 normal 0 stub 0 nssa
  Maximum path: 4
  Routing for Networks:
    10.10.0.0 0.0.0.255 area 0
    10.10.24.1 0.0.0.0 area 0
    10.10.25.1 0.0.0.0 area 0
```

**문제점:** 와일드카드 `0.0.0.255`로 인해 범위가 `10.10.0.0 ~ 10.10.0.255`로 확대됨
- 현재는 문제없지만, 나중에 `10.10.0.100` 같은 주소를 추가할 경우 자동으로 OSPF Area 0에 포함될 위험이 있음

#### Router ID 변경

R3 router-id를 3.3.3.3으로 변경할 때:

```ios
router ospf 10
router-id 3.3.3.3
```

- 오류 메시지: `% OSPF: Reload or use "clear ip ospf process" command, for this to take effect`
- 이미 Router ID가 선언되어 있기 때문에 변경하려면 리로드하거나 `clear ip ospf process` 명령어 필요

do show ip route ospf 명령어 R1에서 확인:

```ios
Gateway of last resort is not set

      10.0.0.0/8 is variably subnetted, 8 subnets, 3 masks
O        10.10.24.1/32 [110/2] via 10.10.0.3, 00:02:41, GigabitEthernet0/0
O        10.10.25.1/32 [110/2] via 10.10.0.3, 00:02:41, GigabitEthernet0/0
      192.168.1.0/32 is subnetted, 1 subnets
O        192.168.1.1 [110/2] via 10.10.0.2, 00:09:52, GigabitEthernet0/0
      209.165.200.0/32 is subnetted, 1 subnets
O        209.165.200.225 [110/2] via 10.10.0.2, 00:09:52, GigabitEthernet0/0
```

**Q: 왜 10.10.0.2/0.3은 안 나옴?**
- **A:** 10.10.0.2/0.3은 직접 연결된 인터페이스이기 때문에 OSPF 라우팅 테이블에 표시되지 않음

```ios
R1#show ip ospf int brief 
Interface    PID   Area            IP Address/Mask    Cost  State Nbrs F/C
Lo0          7     0               10.10.8.1/24       1     LOOP  0/0
Lo1          7     0               10.10.9.1/24       1     LOOP  0/0
Gi0/0        7     0               10.10.0.1/29       1     DR    2/2

R2#show ip ospf int brief
Interface    PID   Area            IP Address/Mask    Cost  State Nbrs F/C
Lo1          7     0               192.168.1.1/26     1     LOOP  0/0
Lo0          7     0               209.165.200.225/27 1     LOOP  0/0
Gi0/0        7     0               10.10.0.2/29       1     BDR   2/2

R3#show ip ospf int brief 
Interface    PID   Area            IP Address/Mask    Cost  State Nbrs F/C
Lo0          10    0               10.10.24.1/24      1     LOOP  0/0
Lo1          10    0               10.10.25.1/26      1     LOOP  0/0
Gi0/0        10    0               10.10.0.3/29       1     DROTH 2/2
```

**역할 분석:**
- **R1**: DR (Designated Router - 반장)
- **R2**: BDR (Backup Designated Router - 부반장)
- **R3**: DROTHER (따라가는 라우터)


### R3 - 기본경로 설정

#### 1단계: Static Default Route 설정

```ios
ip route 0.0.0.0 0.0.0.0 lo0
```

**목적:** 정확한 목적지가 없는 패킷을 Loopback0으로 전송

**경고 메시지:**
```
% Default route without gateway, if not a point-to-point interface, may impact performance
```
게이트웨이가 없는 기본 경로는 성능에 영향을 미칠 수 있습니다.

#### 2단계: OSPF 기본경로 광고

```ios
router ospf 10
default-information originate
end
```

**R3의 라우팅 테이블:**
```ios
R3#show ip ro st | b Gate 
Gateway of last resort is 0.0.0.0 to network 0.0.0.0

S*    0.0.0.0/0 is directly connected, Loopback0
```

**R1의 라우팅 테이블 (기본경로 수신):**
```ios
R1#show ip ro os | b Gate
Gateway of last resort is 10.10.0.3 to network 0.0.0.0

O*E2  0.0.0.0/0 [110/1] via 10.10.0.3, 00:03:13, GigabitEthernet0/0
      10.0.0.0/8 is variably subnetted, 8 subnets, 3 masks
O        10.10.24.1/32 [110/2] via 10.10.0.3, 00:13:47, GigabitEthernet0/0
O        10.10.25.1/32 [110/2] via 10.10.0.3, 00:13:47, GigabitEthernet0/0
      192.168.1.0/32 is subnetted, 1 subnets
O        192.168.1.1 [110/2] via 10.10.0.2, 00:20:58, GigabitEthernet0/0
      209.165.200.0/32 is subnetted, 1 subnets
O        209.165.200.225 [110/2] via 10.10.0.2, 00:20:58, GigabitEthernet0/0
```

**결과:** R3에서 설정한 기본경로가 `O*E2` (External Type 2) 형태로 다른 라우터에 광고됨

### Passive Interface 설정

**목적:** LAN 구간을 passive로 선언해도 라우터가 해당 네트워크 주소는 잘 간직하여 광고함. 보안상 이유로 중요함.

#### 방법 1: 개별 인터페이스 설정

```ios
router ospf 7
passive-interface Loopback 0
passive-interface Loopback 1
```

#### 방법 2: 기본 설정 후 예외 지정

```ios
router ospf 10
passive-interface default
no passive-interface g0/0
```

**결과:**

```ios
R1#show ip ro os | b Gateway
Gateway of last resort is not set

      10.0.0.0/8 is variably subnetted, 8 subnets, 3 masks
O        10.10.24.1/32 [110/2] via 10.10.0.3, 00:30:36, GigabitEthernet0/0
O        10.10.25.1/32 [110/2] via 10.10.0.3, 00:30:36, GigabitEthernet0/0
      192.168.1.0/26 is subnetted, 1 subnets
O        192.168.1.0 [110/2] via 10.10.0.2, 00:03:43, GigabitEthernet0/0
      209.165.200.0/27 is subnetted, 1 subnets
O        209.165.200.224 [110/2] via 10.10.0.2, 00:03:54, GigabitEthernet0/0
```

### Point-to-Point 설정

**개념:** R2 라우터의 루프백 주소가 `/26`, `/27`로 설정되어 있으나, 다른 라우터에게 전송될 때는 `/32`로 변환됨. 원래 비트마스크로 전송하려면 point-to-point 설정 필요.

**설정 방법:**

```ios
interface loopback 0
ip ospf network point-to-point
interface loopback 1
ip ospf network point-to-point
```

**이유:**
- 설계도에서 `/26`, `/27`로 명시되어 있는데, 화면에는 `/32`로 나오면 혼동될 수 있음
- `/26`는 64개의 IP를 가진 네트워크 범위를 의미하는데, `/32`는 하나의 호스트만 의미함
- 이렇게 설정하면 "저기 64개 네트워크가 있구나"를 직관적으로 파악 가능


### 링크 코스트 (Link Cost)

**배경:**
- OSPF는 오래된 프로토콜로 Fast Ethernet과 Gigabit Ethernet을 구분하지 못함
- 자동 코스트 참조 대역폭(Auto-cost reference-bandwidth)을 수동으로 설정하면 cost 값이 변경됨

**설정 예시:**

```ios
R1#conf t 
R1(config)#router ospf 7 
R1(config-router)#auto-cost reference-bandwidth 10000 
R1(config-router)#end
R1#show ip ro os | b Gateway
Gateway of last resort is not set
      10.0.0.0/8 is variably subnetted, 8 subnets, 3 masks
O        10.10.24.1/32 [110/11] via 10.10.0.3, 00:00:39, GigabitEthernet0/0
O        10.10.25.1/32 [110/11] via 10.10.0.3, 00:00:39, GigabitEthernet0/0
      192.168.1.0/26 is subnetted, 1 subnets
O        192.168.1.0 [110/11] via 10.10.0.2, 00:00:39, GigabitEthernet0/0
      209.165.200.0/27 is subnetted, 1 subnets
O        209.165.200.224 [110/11] via 10.10.0.2, 00:00:39, GigabitEthernet0/0
```

**📝 설명:** 
- Cost가 11이 되는 이유: 10 + 네트워크 루프백 cost 1 = 11


### Process ID 설정

> OSPF 구성할 때 process-id 값을 한 토폴로지 내에서 일정하게 주는 것이 관리상 용이함

### DR/BDR 선출 방식

**배경:**
- 10개의 라우터가 동일한 L2 스위치에 연결되어있으면, 45개의 이웃관계(adjacency)가 필요함
- OSPF 트래픽이 엄청날 수 있으므로 DR(반장)과 BDR(부반장)을 선출하여 트래픽 통제

**선출 규칙:**
- 모든 라우터는 기본 Priority 값 1을 가짐
- Priority 값이 동일하면 Router ID 값이 높은 라우터가 DR로 선출
- 수동으로 DR을 지정할 수도 있음

**수동 DR 설정:**

```ios
interface g0/0
ip ospf priority 255 
clear ip ospf process
```

- Priority 값을 255로 설정하면 해당 라우터가 DR이 될 확률이 매우 높음
- `clear ip ospf process` 명령어로 OSPF 프로세스를 재시작해야 적용됨

### Process ID 통일의 중요성

> OSPF 구성 시 토폴로지 내에서 process-id를 동일하게 설정하는 것을 권장


## 🎓 Multi-OSPF의 이해

### Area의 역할

**Single Area vs Multi Area:**
- **Single Area:** area를 0으로 통일
- **Multi Area:** area를 다르게 설정

### Area 0의 중요성

**Area 0 (Backbone Area):**
- 역할: Backbone(척추)
- 모든 네트워크는 **반드시** Area 0을 거쳐서 통신됨

**Area 1, 2, 3... 등:**
- Area 0을 제외한 Area들은 자기들끼리 통신할 때 Area 0을 거칠 필요 없음

### 구성 시 주의사항

> ⚠️ 팀원들과 토폴로지를 구성할 때, area값을 중복시키지 않도록 주의해야 함
> 중복되면 혼란이 발생할 수 있음
