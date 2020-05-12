MAP REDUCE 
=====================
![](https://miro.medium.com/max/858/1*yyEkiwQGIESn9UHL8hwjWg.png)

```
* 원하는 형태로 데이터를 가공하는데 효과적인 프로그래밍 모델

* 병행성을 고려하여 설계되어 대용량 데이터셋에서 많이 활용
```

* 원하는 형태로 데이터를 가공하는데 효과적
* 데이터를 가져와 처리하는 것이 아닌, 데이터가 있는 곳에 코드를 전달하여 처리하는 방식
* Hadoop의 특성상 소수의 큰 파일이 처리하기 쉽고 효율적
    * 실시간 처리에 적합하지 않음. 일반 배치 작업에 적합함
    * 오프라인배치 프로세싱에 최적화되어 있어 적은 수의 큰 데이터파일들을 오프라인에서 배치로 처리하는데 적합한 프레임워크

<hr/>

병렬 처리의 복잡함
---------------------
* 일을 항상 동일한 크기로 나누는 것이 쉽고 명확하지 않음
* 각 프로세스의 결과를 모두 합치는데에 더 많은 처리가 필요할 수 있음
* 잡의 전체 과정을 조율하고 프로세스의 실패를 어떻게 처리해야할 지 고민해야함

이러한 이슈 해결을 위해, Hadoop과 같은 프레임워크를 사용하는 것이 큰 도움이 된다.

<hr/>

Job
---------------------
```
* 클라이언트가 수행하는 작업의 기본 단위로 input data, 맵리듀스 프로그램, 설정 정보로 구성
* Hadoop의 경우 Map task와 Reduce task로 나누어 실행하며
* 각 task는 YARN을 이용해 스케쥴링하여 여러 클러스터의 node에서 실행 될 수 있도록 함
* 특정 node의 task가 실패하면 자동으로 다른 node에서 수행
```

![](https://img1.daumcdn.net/thumb/R800x0/?scode=mtistory2&fname=https%3A%2F%2Ft1.daumcdn.net%2Fcfile%2Ftistory%2F2133764B54F929D108)
- Map Task
    - input data를 가공하여 사용자가 원하는 정보를 key-value형태로 변환
    - 데이터 추출을 위한 Map 함수 작성 필요
    - key를 기준으로 sorting, grouping
    - 결과물은 HDFS가 아닌 로컬 디스크에 저장

- Reduce Task
    - Map Task를 끝내고 얻은 중간 산출물의 key를 기준으로, 사용자가 원하는 결과 추출
    - 결과 추출을 위한 Reduce 함수 작성 필요
    - Reduce의 출력 타입은 Map단계의 출력 타입과 같아야 함
    - 결과물은 안정성을 위해 HDFS에 저장
    - Reduce Task의 수는 지정할 수 있다(없을 수도 있다)

Map 함수와 Reduce 함수를 작성하여 Job을 구현하면, 이 Job을 각각의 클러스터에 JAR파일 형태로 배포하여 작동

분산형으로 확장
---------------------
![](https://www.supinfo.com/articles/resources/207908/2807/3.png)

* Hadoop은 Job의 입력을 입력 스플릿<sup>input split</sup>로 분리한다. 각 스플릿마다 하나의 Map task를 생성하고, 스플릿의 record를 사용자 정의 함수로 처리함

* 스플릿으로 분리하여 병렬로 처리하기 때문에 부하 분산<sup>load balance</sup>에 매우 효과적 

* 스플릿의 크기는 통계적으로 HDFS block의 크기와 같으면 좋다고 알려져 있다.
    * 스플릿의 크기가 너무 작으면, 스플릿에 대한 관리와 각 스플릿에 대한 map task 생성으로 인한 오버헤드로 오히려 job의 실행시간이 더 늘어날 수 있기 때문
    * 또한 스플릿의 크기가 너무 크다면, 하나의 스플릿이 여러 HDFS block에 걸쳐있을 수 있으므로 네트워크를 이용해야 하는 오버헤드가 생길 수 있다.

## Data locality
![](https://4.bp.blogspot.com/-GVXvWh8wfos/WtxprHu6j-I/AAAAAAAAAoo/lDiWHZaV1dw5CFSPDuKjwHBtR6wEkpJKgCLcBGAs/s1600/Data-locality.png)

 Hadoop은 HDFS에 input file을 block단위로 나누어 저장하게 되는데, 이때 block의 default size는 128MB이며 3개의 노드에 복제본이 저장된다.
* Data local - split data가 저장되어있는 동일한 node에서 map task를 실행. 가장 빠름. 네트워크 대역폭을 이용하지 않는 방법.
* Rack local - split data가 저장되어있는 같은 Rack의 node에서 map task를 실행. 
* Different Rack - split data가 저장되어 있는 다른 Rack의 node에서 map task를 실행. rack간 네트워크 전송으로 속도가 느린편

Combiner function
---------------------
* Job에서 Map Task와 Reduce Task간 데이터 전송을 최소화 하기 위해 사용
* Map의 결과를 일부 reduce하는 기능. Combiner function의 출력이 reduce의 입력이 되는 것.
* Reduce로 보내기 위한 suffling/sorting 이 발생하기 전에 reduce를 적용하는 것으로, mini-reducer라고도 불림
* 단, reduce function의 속성이 교환법칙및 결합법칙을 만족해야 함

테스트 수행
---------------------
독립모드<sup>standalone</sup>로 Hadoop을 설치하면 로컬 기준에서 테스트 가능. 각 Job에 대한 통계를 볼 수 있어 매우 유용

Hadoop Streaming
---------------------
* Hadoop은 스트리밍 구조로, 직렬화<sup>serialization</sup> 사용. 이러한 최적화된 네트워크 직렬화를 위해 자체적으로 기본 타입셋 제공
    * 직렬화를 통해 네트워크 대역폭 절약, 메시지 전송 오버헤드 감소, 투명성, 다양한 언어로 작성된 클라이언트 지원 가능
* Hadoop과 user program사이의 인터페이스로 유닉스 표준 스트림을 사용하기 때문에, 사용자는 다양한 언어를 이용해 맵리듀스 프로그램을 작성할 수 있다.<br>ex)Ruby, Java, Python
* node 사이의 프로세스 간 통신은 RPC<sup>Remote Procedure Call</sup>를 통해서 이루어진다.


기타 주의사항
---------------------
* Job을 수행하는 시점에서 결과 저장을 위해 지정한 디렉토리가 사전에 존재하지 않도록 해야 함. 덮어쓰기로 인한 데이터손실 방지
* Job을 정의할 때, 컴파일 시점에서 데이터 타입 일치 여부를 확인하지 않기 때문에 데이터 입력 및 출력 타입이 일치하는지 확인
