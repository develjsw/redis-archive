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
  - Redis에서 String 값(JSON 형식)을 저장한 후 bytes 값을 확인하면, 우리가 입력한 크기보다 더 높게 측정됨
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
 
  - Redis maxmemory 설정에 따른 데이터 저장 한계 분석 및 OOM 발생 테스트 (Docker 컨테이너 내부에서 실행)
    - Redis 최대 메모리 설정
      1) 개요
         - Redis를 실행하면 기본적으로 약 1.58MB의 메모리가 사용됨
         - 최대 메모리를 2MB(=2,097,152 bytes)로 설정한 후 데이터 저장 가능 개수를 분석함
         - 영구 적용을 원할 경우 redis.conf 파일에서 maxmemory [메모리값]을 추가하여 설정 가능
      2) Redis 설정 방법
         ~~~
         $ redis-cli -h 127.0.0.1 -p 6377 CONFIG SET maxmemory 2097152  # 최대 메모리 설정
         $ redis-cli -h 127.0.0.1 -p 6377 CONFIG GET maxmemory          # 설정 값 확인
         $ redis-cli -h 127.0.0.1 -p 6377 CONFIG GET maxmemory-policy   # 메모리 정책 확인
         ~~~
      3) Eviction 정책
         - maxmemory-policy 기본값은 noeviction이며, 이 경우 메모리 초과 시 새 데이터 저장이 불가능
         - Eviction 정책을 설정하면, 초과 시 특정 키가 삭제되고 새로운 데이터를 저장할 수 있음 EX) allkeys-lru, volatile-lru
         - 여기서는 noeviction로 진행 예정
    - 사용 가능한 메모리 계산
      1) 초기 상태 (데이터 없음)
         ~~~
         $ redis-cli -h 127.0.0.1 -p 6377 INFO MEMORY
         
         # 결과 #
         used_memory:1660056
         used_memory_human:1.58M
         ~~~
         - Redis가 기본적으로 사용하는 메모리가 1.58MB(=1,660,056 bytes)였음
         - 이 값은 데이터가 없는 상태에서도 Redis 내부 구조 및 시스템 오버헤드로 인해 필요함
      2) 사용 가능한 여유 메모리
         - 2,097,152bytes − 1,660,056bytes = 437,096bytes(=427KB)
      3) 키 당 예상크기
         - Redis는 키-값을 저장할 때 추가적인 메모리 오버헤드가 존재하며, 실험에서는 다음과 같이 가정함
           - 1 ~ 999의 키(key1 ~ key999) → 각 64 bytes
           - 그 이후 키(key1000 ~ key9999) → 각 72 bytes
      4) 이론적인 저장 가능 개수 계산
         - 먼저, 999개의 키(각 64 bytes) 저장 후 남은 메모리 계산
           - 437,096bytes − (999 × 64bytes) = 373,160bytes
         - 나머지 공간에서 72bytes 크기의 키를 저장할 수 있는 개수 계산
           - 373,160bytes / 72bytes = 5,182개
         - 최종 예상 저장 개수
           - 999개 + 5,182개 = 6,181개
    - Redis 대량 데이터 생성 테스트
      - 아래 명령어를 실행하여 key1부터 key687500까지 데이터를 저장함
        ~~~
        $ for i in $(seq 1 6181); do redis-cli -h 127.0.0.1 -p 6377 SET key$i "same_value"; done
        ~~~
    - 결과 확인 및 OOM 발생
      1) 실제 테스트 결과
         - 실제 저장된 데이터 개수: 5,481개
         - 5,000개 초과 시점에서 OOM(Out of Memory) 오류 발생
           ~~~
           (error) OOM command not allowed when used memory > 'maxmemory'
           ~~~
         - 이론적 저장 개수(6,181개)보다 실제 저장 개수가 적음
         - Redis 내부 오버헤드 및 메모리 단편화 영향 때문으로 추정됨
      2) 테스트 후 메모리 정보
         ~~~
         $ redis-cli -h 127.0.0.1 -p 6377 INFO MEMORY
       
         # 결과 #
         used_memory:2079528           # 현재 Redis가 사용 중인 메모리 (바이트 단위)
         used_memory_human:1.98M       # 사람이 읽기 쉬운 단위(MB)로 변환한 used_memory 값
         used_memory_rss:13455360      # 운영체제(OS)에서 Redis 프로세스에 할당한 실제 물리적 메모리
         used_memory_rss_human:12.83M  # 사람이 읽기 쉬운 단위(MB)로 변환한 used_memory_rss 값
         mem_fragmentation_ratio:6.54  # 메모리 단편화 비율 (used_memory_rss / used_memory)
         ~~~
    - 결과 분석
      1) 이론값과 실제값의 차이 발생
      2) 발생 원인
         - Redis 내부 데이터 구조의 오버헤드
           - 해시 테이블 관리, 키-값 메타데이터 저장 등으로 예상보다 많은 메모리가 사용됨
         - 메모리 단편화(Fragmentation) 발생
           - mem_fragmentation_ratio: 6.54 → OS가 Redis에 할당한 물리 메모리가 실제 used_memory보다 훨씬 많음
           - Redis의 메모리 관리 방식에 의해 일부 공간이 비효율적으로 사용됨
    - 결론
      - 운영 환경에서 메모리가 넉넉하지 않다면 OOM(Out of Memory)을 생각보다 빨리 마주할 수 있으며, 대비책으로는 Redis 메모리 정책을 적절히 조절하고 TTL 적절히 사용하며 모니터링을 해줘야 함 