---
aliases:
  - K8s
  - 쿠버
  - 쿠버네티스
---


- Service : 외부와 연결
- Pod : Container 들을 담는다.
	- Config / Secret
- Volume : 영속 데이터를 위함

- Master : 
- Node : 

- Namespace : 격리기술
	- ResourceQuota / LimitRange : 자원 제한

- Controller
	- ReplicaSet : 파드 갯수, 회복 등
	- Deployment : 배포 및 업그레이드, 롤백 기술
	- DaemonSet : (?)
	- CronJob : 스케쥴링을 위함 (?)


```mermaid
```


# K8s 의 Object 들

## Container, Pod

Container 하나 ~ 아주 가벼운 VM 같은 느낌
Pod 엔 여러 Container 들이 들어갈 수 있고 Container 들 내부에선 포트들이 중복될 수 있다. 당연히 노출되는 포트는 겹치면 안됨.

K8s 내부 ip 할당됨 (K8s 클러스터 내부 네트워크 전용)

### request, limits

Pod 설정시 cpu, memory 제한을 걸 수 있는데 두 가지임.

memory 는 limit 넘으면 OOM kill 당하지만, cpu 는 넘으려 해도 제한만 걸고 죽이진 않음.

request 만큼 할당 가능한 여유로운 node 로 자동 할당하는 기능도 있음.

### nodeSchedule

Pod 가 뜰 때 자원이 여유로운 노드에 띄워준다. (혹은 `nodeSelector`  가 정해준다.) 


## Label

오브젝트 분류를 위한 key-value. (e.g. `env:dev`)
pod 설정시 metadata 에 적히며 service 에선 selector 에 적힘. 서비스와 연결을 위해 사용.
노드 연결을 위해서도 사용 (pod 의 `noeSelector`)


## ReplicationController

연결된 Pod 를 관리해줌



## Service

Pod 는 매우 휘발적임. (ex: ip)

Service 는 직접 바꿔주지 않는 한 고정적임. 

### type: ClusterIp

디폴트 값. K8s 클러스터 내부에서 접근 가능.
Pod ip 와의 차이
- 비휘발
- 각 연결된 pod 에 적절히 부하 분배.

- **내부** 에서 접근 가능하니 운영, 개발인원만 사용

### type: NodePort

ClusterIp 할당됨.

모든 노드의 **특정 포드** 에 연결. 이는 외부와 연결된다. (Ext)
이 특정 포트 (nodePort) 는 30000~32767 사이에서 정함.

- 호스트 IP 를 통해 접근 가능.
보통 이렇게 직접 접근 안하기 때문에, 내부망에서 사용하거나, 데모, 임시 연결용.

- externalTrafficPolicy: local 인 경우 node 에 속한 pod 로만 트래픽 보냄. (설정 없었다면 랜덤하게 부하 분산)

### type: LoadBalancer

NodePort 할당됨.

외부 IP 지원 플러그인 필요. (GCP, AWS...)

- 내부 정보는 감춘다.
- 외부에 시스템 노출해 서비스하는 용도



## Volume

### emptyDir

휘발성이 있는, Pod 내부 Container 들이 공유하기 위한 공간.

팟이 죽으면 사라진다.

### hostPath

노드에 귀속된, pod 들이 공유하기 위한 공간.

pod 입장에선 다른 node 에서 되살아날수도 있어서... 항상 같은 공간을 본다는 보장은 없다.

따라서 노드 자신을 위한 설정이나 로그 등을 위해 사용

hostPath 가 pod 설정에 있는데 경로가 없으면 **에러**남

### PVC/PV

Persistent Volume.

Persistent Volume Claim 을 통해 연결됨. (인터페이스라 생각하셈)

storageClassName 에 빈값을 넣는지, "" 를 넣는지, PV 네임을 넣는지에 따른 동작이 다름.

