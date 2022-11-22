# Heap 분석 정리

### 개요

최근에 팀내에서 Memory 임계치가 70% 이상을 넘기는 일들이 발생했다.

<img width="1013" alt="image" src="https://user-images.githubusercontent.com/57784077/203184252-cf45ee03-82d6-4e06-8963-ac16ebf50b9a.png">

[사진]

### 지표를 통한 문제 분석

현재 팀내 시스템에는 08 ~ 22 사이에 다수의 일감을 분배해주는 큰 Dispatcher 시스템이 존재한다.  
10초당 한번 씩 동작하고 있으며 DB 작업과 Queue 작업을 수반하고 수백~수천건의 객체들을 사용하게 된다. (중복 발행도 가능한 시스템이라 로드가 어느정도 있다.)
그래서 이 구간에서 **Object Reference** 가 잘 해제되지 않는건가? 라는 의심을 가지고 분석을 시작하게 됬다.

<img width="1915" alt="image" src="https://user-images.githubusercontent.com/57784077/203184360-fd297ca5-50e9-433f-beba-ab63c1d74ae0.png">

의심되는 부분을 일단 확인하기 위해 로그를 확인해봤는데 Dispatcher 시스템은 **새벽시간**대에는 동작하지 않는다는 것을 확인했다.
그래서 해당 시스템이 문제를 일으키지 않는다고는 생각했고 다른 코드 부분에 문제가 있을 것이라고 파악했다.

<img width="1137" alt="image" src="https://user-images.githubusercontent.com/57784077/203184451-87adcf8b-528a-49b5-88d6-c8ddff8eb07f.png">

**[주간 Heap 사용량 그래프 사진]**

일단 주간 지표도 살펴보면 70% 를 넘는 구간이 꽤나 보이는 것을 확인할 수 있다.
즉, 주간에도 70%를 넘는지점들이 종종 있으나 실행주기가 짧은 Dispatcher 시스템으로 인해 Young Generation 이 빠르게 가득차서 **MinorGC** 의 횟수가 많아서 알럿이 잘 뜨지 않았던것으로 파악된다.  

<img width="454" alt="image" src="https://user-images.githubusercontent.com/57784077/203185238-786dad12-fc2b-4c40-91fb-83cd310d7822.png"> 

**[야간 Heap 사용량 그래프 사진]**

지표를 보면 야간에는 주간에 비해 MinorGC 그래프가 훨씬 더 적은것을 확인 할 수 있다.  
결과적으로, 주간이든 야간이든 GC 횟수가 낮아진다면 과반수 인스턴스의 Heap 사용량이 70%를 충분히 넘긴다는 뜻이다.  

여기까지 분석해봤을때 두가지 정도의 결론이 나왔다.

- Young generation 영역의 **임계치(Threshold)** 비율이 70% 에 가까워서 60 후반 ~ 70% 대가 되어야 MinorGC 가 돌게 된다.
- 새벽에 도는 특정 작업 중 하나가 Object Reference 가 잘해제 안되는가 싶었다.

이제 더 확실하게 알기 위해 HeapDump 를 떠보자

### HeapDump 를 통한 분석

HeapDump 를 분석하기 전에 알아야하는 몇가지 용어가 있다.

- **Shallow** : 객체가 메모리에 할당된 사이즈(Byte 기준)
- **Retained** : GC 가 실행되더라도 보존될 사이즈 (누군가 Reference 를 참조하고 있음)
- **GC Roots** : HeapDump 를 뜨는 순간 GC 를 할 수 없었던 객체들의 정보

위 용어를 알아야 HeapDump 를 분석하기 편하다.

## 메모리에 할당된 객체들이 정리안되는 이유 분석

<img width="1087" alt="image" src="https://user-images.githubusercontent.com/57784077/203185564-899db15d-470a-4a8f-83b1-4a3b5cbb8986.png">

분석결과 많이 할당되어 있는 객체는 `byte[]` 였다. 즉, 직렬화된 데이터가 많아졌다는 것이다.  

<img width="1506" alt="image" src="https://user-images.githubusercontent.com/57784077/203185898-037c5dac-c4cc-4d55-8fa0-ddf9cc99ece4.png">

**[JPA Query Cache]**

<img width="1060" alt="image" src="https://user-images.githubusercontent.com/57784077/203185996-c1df4317-a71e-40ea-85fc-cb2d2d63c052.png">

**[Redis Cluster Cache]**

HeapDump 에 뛰어져 적혀져 있는 것들을 보니 대부분 **Jpa Query Cache** 와 **Redis Cluster Cache** 였다.  
어플리케이션이 빠른 반응속도를 위해 Cache 를 많이 사용하다보면 당연하게도 메모리 사용량은 증가하게 된다.  
그래프를 보면 하나를 알 수 있는데 Shallow 는 엄청 많이 늘었으나 Retained 는 그대로인것을 확인할 수 있다.  
즉, 할당은 엄청많이 됬으나 **Reference 를 계속 물고 있지는 않아 GC 의 타겟**인것을 확인할 수 있다.  
추론해봤을때 **MinorGC 가 byte[] 가 1.8G 가 될때까지 돌지 않았음을 확인**할 수 있다.

## 대응 방안

일단 HeapDump 분석을 간단하게 정리해보면 아래와 같다.

- **대부분의 객체들은 MinorGC 의 수집대상에 포함된다. 하지만, Minor GC 가 수행되지 않았음.**

위의 이유는 사실 수행되지 않았다! 라기 보다는 **MinorGC 가 돌아야 하는 Young Generation 의 임계치(Threshold) 에 도달하지 않았다.** 라는 판단이 들었다.  
그래서 테스트 가설 자체를 Young Generation 의 비율을 기존 비율보다 줄이는 방향으로 했을때 새벽시간대 Heap 사용량이 어떻게 변하는지 테스트 해보는 방향으로 해야겠다는 생각이 들었다.

## NewRatio 비율 조정 테스트 

운영환경에서 바로 테스트 할 수 없어서 베타환경에서 먼져 테스트를 진행하였다.  
일단 테스트 전에 비교할 수 있는 데이터를 측정해야 하므로 새벽시간대 베타의 Heap 사용량을 측정했다. 

<img width="919" alt="image" src="https://user-images.githubusercontent.com/57784077/203186867-eabcb56d-6342-499e-8b3c-963454360a7f.png">

베타 환경은 **1GB 의 Heap 용량**을 가지고 있어 매우 작은 상황이며, **측정한 시각(22:00 ~ 08:00)** 대에는 80% 의 힙 사용량을 보인다.

이제 비교할 데이터 측정도 완료했으니 `newRatio` 비율을 조정하여 Young Generation 부분 Size 를 낮춰보도록 하자.  
실험 조건은 아래와 같다

- VM Option 으로 **-XX:NewRatio=3** 값을 주고 테스트
- XX:NewRatio 값을 아래와 같음
    - Young 제너레이션과 Old 제너레이션 비율 지정 Young : Old = 1 : N

## NewRatio 조정 결과

주간기준으로는 아래와같은 결과를 도출했다.

<img width="851" alt="image" src="https://user-images.githubusercontent.com/57784077/203187596-c56a2955-999f-4c52-8793-2d73ee5d4edd.png">

- **MinorGC 의 횟수 증가**

배포 시간대 기준으로 약 **0.07 -> 0.59 (ops/s) 가 증가**하였다. 이는 원하는 결과긴 했다.  
문제는 HikariCP 의 idle Connection(Thread) 들이 빠르게 정리된다는 것이였는데 이로 인해 역으로 **System Load 가 증가하여 Network Latency 가 증가**했다.  
기존에는 idle Connection 을 그대로 사용했으나, idle Connection 이 빠르게 정리되면서 새롭게 Connection 을 수립하는 비용으로 인해 Network Latency 가 증가한것으로 파악된다.  
위의 결과는 베타 환경에서 적은 Heap 사이즈인 1GB 를 사용하고 있는데 Young Generation 의 비율을 250MB 로 잡다보니 위와 같은 결과가 도출됬다고 생각한다. 
따라서 Heap Size 를 늘린다면 idle Connection 이 정리되는일은 없지 않을까 싶다. (Scale Up 후 테스트도 따로 진행할 예정이다. 운영환경은 Beta 환경의 10 배인 10GB Heap 을 이용하고 있다.)  

우리에게 중요한 새벽시간대의 Heap 사용량을 살펴보도록 하자. 

<img width="829" alt="image" src="https://user-images.githubusercontent.com/57784077/203187485-a38cc009-4278-4c78-9b4a-331f450871f5.png">

**[새벽시간 Heap 사용량]**

새벽시간대 결과로 Heap 사용량이 **80% -> 40%** 가 감소되었다. 다만 놀라운건 GC 의 Count 횟수는 별 차이가 나지 않는다는 것이다. 

## 후기

결과적으로 Heap 사용량은 줄이는데 성공했다. 
이번 Heap 튜닝(?) 을 하면서 느낀건 항상 책에서는 어플리케이션의 병목 혹은 코드 지점을 개선하자! Heap 튜닝은 최대한 피하자! 라는 글을 봤었던것 같은데, 
결국 위와 같은 상황이 오게 되면 Heap 튜닝도 필요하다는 느낌을 받았다. 좋은 방법으로 접근한지 판단하기 위해 팀원분들과 사내 Wiki 에도 피드백을 받기 위해 공유해뒀다. 




