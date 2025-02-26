### Redis Archive

- Redis API Server 구축 (NestJS)
  - https://github.com/develjsw/redis-api


- Docker로 Redis Server 구축
  - 로컬에서 간단하게 Redis Server 실행하기
  - https://github.com/develjsw/redis-docker


- 분산락 구현
  - https://github.com/develjsw/concurrency-issue


- NestJS Bull Queue 구현
  - https://github.com/develjsw/producer-api
  - https://github.com/develjsw/consumer-api


- Docker로 Redis HA 설정 및 테스트
  - 작성 예정


- Redis Memory 사용량 측정
  - Redis에서 String 값(JSON 형식)을 저장한 후 byte 값을 확인하면, 우리가 입력한 크기보다 더 높게 측정됨
    - Key: "deps1:10", Value: {"key1": 1, "key2": 2} 저장
    - Value 크기: 22B, Key 크기: 8B → 총 30B여야 하지만, 실제 메모리 사용량은 88B로 측정됨
      <img width="708" alt="Image" src="https://github.com/user-attachments/assets/3b948151-3327-4d8f-a625-8a1fc45aec09" />
    - 원인을 확인해보니 내부적으로 메타 데이터(데이터 구조 정보 등)를 추가하기 때문이라고 함
  
  - key의 길이가 길어질수록 더 많은 메모리를 사용함
    - deps1과 new-deps1:10, new-deps1:10:new-deps2:20, new-deps1:10:new-deps2:20:new-deps3:30은 value는 모두 같지만 key size의 차이만 존재
      <img width="521" alt="Image" src="https://github.com/user-attachments/assets/18698dee-9f0b-4fe0-b6ef-4c5dbfa527e6" />
    - deps1:10과 loger-key-deps1:10의 value는 동일하지만 key size 차이만 존재
      <img width="710" alt="Image" src="https://github.com/user-attachments/assets/9aed1032-23c3-4167-a19a-a3688aa82bb8" />
      <img width="787" alt="Image" src="https://github.com/user-attachments/assets/726d625a-4d7c-45f4-9656-6d97160b4084" />
    - 간혹 Key 길이가 다름에도 (value 값은 동일) 불구하고, 아래 이미지처럼 동일한 바이트 크기로 표시되는 경우가 있음. 이는 Redis의 메모리 할당 방식(jemalloc)에 따른 것
    - Redis는 jemalloc 메모리 할당기를 사용하여 메모리를 블록 단위로 관리함
    - 예를 들어, 64B, 96B, 128B 같은 고정 크기 블록으로 할당되기 때문에, "deps1:10"(8자)과 "no-deps"(7자)처럼 길이가 조금 다른 Key들도 같은 블록에 저장될 수 있음
      <img width="710" alt="Image" src="https://github.com/user-attachments/assets/43eb095f-6e80-44a8-bf8c-ab1eabe12d58" />
      
  - Redis maxmemory 설정에 따른 데이터 저장 한계 분석 및 OOM 발생 테스트
    - 작성중