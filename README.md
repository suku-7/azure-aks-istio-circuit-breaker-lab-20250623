# Model
## azure-aks-istio-circuit-breaker-lab-20250623
https://labs.msaez.io/#/courses/cna-full/2c7ffd60-3a9c-11f0-833f-b38345d437ae/deploy-my-app-2024

## Azure AKS 환경에서 Istio 서비스 메시의 Circuit Breaker 기능 실습
- Azure AKS 노드 풀의 스케일링을 통해 클러스터 리소스를 확장하고, 노드 및 Pod의 리소스 사용량을 모니터링합니다.
- delivery 서비스를 Istio 환경에 배포하고, DestinationRule을 사용하여 서킷 브레이커(아웃라이어 감지) 정책을 설정합니다.
- Siege Pod를 통해 delivery 서비스에 요청을 보내고, 의도적으로 서비스 인스턴스에 장애를 유도하여 서킷 브레이커가 동작하는 것을 확인합니다.

## 사전 준비
Azure 계정 및 구독, Gitpod 워크스페이스, Spring Boot 애플리케이션 코드

![스크린샷 2025-06-23 134902](https://github.com/user-attachments/assets/14cbc3d6-d22c-406c-a850-589b3dd3929a)
![스크린샷 2025-06-23 140547](https://github.com/user-attachments/assets/1b169631-fe8f-4f9f-a5cc-d32d81a5613a)
![스크린샷 2025-06-23 140552](https://github.com/user-attachments/assets/353bf17e-6dd3-4daa-9a99-5d8d1a697853)
![스크린샷 2025-06-23 140937](https://github.com/user-attachments/assets/424c193a-57fc-46e3-b855-ccfcc431862d)
![스크린샷 2025-06-23 141434](https://github.com/user-attachments/assets/b642fd5f-7bd7-4847-9b3f-30ad63995ded)
![스크린샷 2025-06-23 141548](https://github.com/user-attachments/assets/d5305e81-8020-4b12-86e9-774db8e73a61)

---

## 실습 단계별 상세 설명

0. 초기 환경 설정 및 준비
--- 터미널 1 (메인 작업) ---
```
# Gitpod 환경 초기화 (재실행 시 필요한 스크립트)
./init.sh

# Java SDK 설치/업그레이드
sdk install java

# Azure 로그인 및 AKS 클러스터 자격 증명 설정
az login --use-device-code
# (프롬프트에 따라 구독 선택 및 인증 진행)
az aks get-credentials --resource-group a071098-rsrcgrp --name a071098-aks
```
1. 클러스터 노드 및 Pod 리소스 사용량 확인
--- 터미널 1 (메인 작업) ---
```
# 현재 AKS 클러스터 노드들의 CPU 및 메모리 사용량 확인
kubectl top nodes

# 현재 클러스터의 Pod들의 CPU 및 메모리 사용량 확인
kubectl top pod
```
2. Azure AKS 노드 풀 스케일링
--- 터미널 1 (메인 작업) ---
```
# AKS 클러스터의 'agentpool' 노드 풀의 최소/최대 노드 수를 3,3 -> 4,4로 조정
# 이렇게 하면 클러스터에 4개의 노드가 유지되도록 스케일링됩니다.
az aks update --resource-group a071098-rsrcgrp --name a071098-aks --enable-cluster-autoscaler --min-count 4 --max-count 4

# 노드 스케일링 후, 노드 수가 증가했는지 다시 확인
# 새로운 노드가 추가되고 READY 상태가 될 때까지 시간이 걸릴 수 있습니다.
kubectl top nodes
```
3. Delivery 서비스 배포 및 스케일 아웃
--- 터미널 1 (메인 작업) ---
```
# 기존 tutorial 네임스페이스 리소스 정리 (이전 랩의 잔여물이 있다면 삭제)
kubectl delete deployment,service --all -n tutorial
kubectl delete dr --all -n tutorial
kubectl delete vs --all -n tutorial

# Delivery 서비스 배포 (Circuit Breaker 테스트용 이미지 사용)
# 이 이미지는 특정 요청에 대해 장애를 유도할 수 있는 기능을 포함합니다.
kubectl create deploy delivery --image=ghcr.io/acmexii/delivery:istio-circuitbreaker -n tutorial

# Delivery 서비스의 Pod 수를 2개로 스케일 아웃 (Circuit Breaker 테스트를 위함)
kubectl scale deploy delivery --replicas=2 -n tutorial

# Delivery 서비스를 ClusterIP 타입으로 노출
kubectl expose deploy delivery --port=8080 -n tutorial

# tutorial 네임스페이스 내의 모든 리소스 상태 확인 (delivery Pod가 2/2 Ready인지 확인)
kubectl get all -n tutorial
```
4. 배송 서비스 (Delivery)에 Circuit Breaker 설정 적용
--- 터미널 1 (메인 작업) ---
```
# Delivery 서비스에 대한 DestinationRule을 적용하여 서킷 브레이커(Outlier Detection) 설정
# - loadBalancer: ROUND_ROBIN (라운드 로빈 부하 분산)
# - outlierDetection: 아웃라이어(이상 징후)를 감지하여 비정상 Pod를 트래픽에서 제외하는 설정
#   - interval: 10s (10초마다 아웃라이어 감지 수행)
#   - consecutive5xxErrors: 1 (연속적으로 5xx 에러가 1회 발생하면 비정상으로 간주)
#   - baseEjectionTime: 3m (최소 3분 동안 트래픽에서 제외)
#   - maxEjectionPercent: 100 (최대 100%의 인스턴스를 제외할 수 있음)
kubectl apply -f - << EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: dr-delivery
  namespace: tutorial # 또는 현재 작업 중인 네임스페이스
spec:
  host: delivery
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN # 부하 분산 방식: 라운드 로빈
      localityLbSetting:
        enabled: false
    outlierDetection:
      interval: 10s # 아웃라이어 감지 주기
      consecutive5xxErrors: 1 # 연속 5xx 에러 발생 시 비정상 간주 횟수
      baseEjectionTime: 3m # 비정상 인스턴스 격리 시간
      maxEjectionPercent: 100 # 최대 격리 가능한 인스턴스 비율
EOF
```
5. Siege Pod 접속 및 특정 서비스에 장애 유도
--- 터미널 1 (메인 작업) ---
```
# 워크로드 생성기(Siege Pod) 셸로 접속
# (접속 후 셸 프롬프트가 바뀝니다. 이후 명령은 이 셸에서 실행합니다.)
kubectl exec -it siege -c siege -n tutorial -- /bin/bash

--- Siege Pod 내부 셸 (접속 후 이어서 실행) ---

# 배송 서비스(delivery)의 각 Replica에 요청을 보내 호스트 정보 확인
# 이 명령을 2번 이상 호출하여 서로 다른 delivery Pod의 IP 주소와 호스트 이름을 확인합니다.
# 예시 출력: delivery-6c4494db7f-hg8xp/10.244.0.143
# 예시 출력: delivery-6c4494db7f-s7x5w/10.244.3.193
http http://delivery:8080/actuator/echo

# 확인된 IP 중 하나를 선택하여 해당 delivery Pod에 장애 유도 (수동으로 다운시킴)
# [선택한_Pod의_IP] 부분을 위에서 확인한 실제 IP 주소로 대체합니다.
# 예: http PUT http://10.244.3.193:8080/actuator/down
http PUT http://[선택한_Pod의_IP]:8080/actuator/down

# 배송 서비스의 헬스(Health) 상태 확인
# 장애 유도 후, 해당 Pod가 DOWN 상태로 표시되는 것을 확인할 수 있습니다.
http GET http://delivery:8080/actuator/health
```
