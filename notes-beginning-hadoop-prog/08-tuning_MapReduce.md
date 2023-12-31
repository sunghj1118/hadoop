[[09. 하둡 운영하기]]


여러분들의 맵리듀스 프로그램의 성능을 개선하려면 어떻게 해야 할까요?

가장 직관적인 방법은 하둡 클러스터를 증설하는 것입니다. 다루고 있는 규모가 작으면 간단한 일이지만, 큰 규모에서 (예: 수백 대 이상의 데이터 노드) 다루려면 엄청난 비용이 들것입니다.

그러나, 맵리듀스 프로그램은 이외에 다양한 튜닝 방법이 존재합니다.

*증설: '증설'은 '더 늘려 설치함'을 의미합니다.*


# 8.1 셔플 튜닝

4장에서 셔플은 MAP과 REDUCE 태스크 사이의 전달 과정이라고 설명한 바 있습니다. 셔플 방식을 어떻게 구성하냐에 따라 성능이 개선될 수 있습니다.

### 8.1.1
셔플 Mapper의 출력으로부터 Reducer의 입력까지의 과정을 포괄한다. 

즉, 메모리에 저장되어 있는 Mapper의 출력 데이터를 파티셔닝, 정렬하여 로컬 디스크에 저장하고, 네트워크로 Reducer에게 입력 데이터로 전달되는 과정을 모두 포함한다.

![[Pasted image 20230810185309.png]]

세부적인 과정은 다음과 같다:
1. 맵리듀스 잡의 입력 데이터는 입력 스플릿으로 분리된 후, 스플릿별로 하나의 맵 태스크 실행.
2. 맵 태스크는 메모리 버퍼를 생성한 후, 맵의 출력 데이터를 버퍼에 기록.
3. 버퍼가 일정 크기에 도달하면 하둡은 해당 데이터를 로컬 디스크로 스필 (spill). 하둡은 로컬 디스크에 데이터를 저장하기 전에 데이터를 분리해서 파티션을 생성. 파티션 내의 배경 스레드는 메모리 내에서 키에 따라 정렬을 수행.
4. 맵 태스크가 종료되기 전에 스필 파일들은 하나의 정렬된 출력 파일로 병합.
5. Reducer는 맵 태스크가 완료되면 즉시 필요한 출력 데이터를 네트워크를 통해 복사. 이렇게 복사된 데이터는 리듀스용 태스크트래커의 메모리 버퍼로 복사.
6. 모든 맵 출력 데이터가 복사되면 해당 데이터를 병합.
7. 리듀스 메서드는 정렬된 출력 데이터에서 키별로 호출.

### 8.1.2 정렬 속성 수정

맵 태스크는 최대한 스필 파일을 줄이는 것이 유리합니다. 이후 로컬 디스크에 저장될 파일이 적을수록 데이터의 병합, 전송, 등의 시간이 단축하기 때문입니다.

스필되는 파일을 줄이려면 is.sort.mb를 늘리면 됩니다. 메모리 버퍼가 커질수록 로컬에 저장될 출력 데이터가 줄어들기 때문입니다. 

실제로 예제에서는 io.sort.mb를 200MB로 늘려보니 6장의 예제가 12분 42초에서 9분 50초로 약 3분 단축 된것을 볼 수 있습니다.

### 8.2 콤바이너 클래스 적용

Combiner class는 매퍼의 출력 데이터가 네트워크를 통해 리듀서에 전달되기 전에 매퍼의 출력 데이터의 크기를 줄이는 기능을 수행합니다. 

콤바이너 클래스는 다음과 같이 Job 인터페이스의 setCombinerClass 메서드를 호출해서 적용할 수 있습니다.

`job.setCombinerClass(DelayCountReducer.class)`

콤바이너를 적용하고 나니 읽은 파일 바이트 수가 1,453,088,421개가 16,577개로 줄었고 Spilled Records는 167,244,625개가 1,685개로 줄었습니다.


반면에 HDFS_BYTES_READ, HDFS_BYTES_WRITTEN 부분은 Combiner 적용 전후가 동일한 것을 볼 수 있습니다. 입력/출력 데이터가 동일하기 때문입니다. 수행 시간은 약 40초 줄어든 것을 확인했습니다.

### 8.3 맵 출력 데이터 압축

하둡은 환경설정 파일에서 mapred.compress.map.output 옵션을 true로 설정하면 맵 태스크의 출력 데이터를 압축 파일로 생성합니다.

이렇게 하면 일반적으로 전송되는 텍스트 파일보다 감소할 것입니다. 

### 8.3.1 Gzip 적용
Gzip은 GNU zip의 준말이며, 리눅스에서 파일을 압축할 경우 대부분 Gzip을 이용합니다.

압축을 위해서는 configuration은 다음과 같이 설정하였다.
`conf.setBoolean("mapred.compress.map.output", true);`
`conf.set("mapred.map.output.compression.codec", "org.apache.hadoop.io.compress.GzipCodec");`

출력 포맷은 다음과 같이 시퀀스 파일로 설정하고, 압축 포맷을 Gzip으로 설정합니다.
`job.setOutputFormatClass(SequenceFileOutputFormat.class);`
`SequenceFileOutputFormat.setCompressOutput(job, true);`
`SequenceFileOutputFormat.setOutputCompressorClass(job, GzipCodec.class);`
`SequenceFileOutputFormat.setOutputCompressionType(job, CompressionType.BLOCK);`

위와 같이 설정하고 잡을 실행한 결과 13분에서 7분으로 줄어들었습니다.

### 8.3.2 스내피 설치 및 적용
스내피는 구글에서 개발한 압축 라이브러리로 압축률을 높이는것보다 적당한 압축률을 빠르게 수행하는것을 목표로 합니다. 

스내피를 설치 및 적용하니 잡은 약 5분 30초로 줄어들었습니다.


### 8.4 DFS 블록 사이즈 수정
맵리듀스 잡은 수행되는 맵 태스크와 리듀스 태스크개수에 따라 성능에 영향을 받게 됩니다. HDFS에 파일을 업로드하면 64MB 단위로 파일이 기본값으로 올라간다.

이때, 64MB보다 작은 크기로 분리하면 더 많은 블록으로 분리되면서 맵 태스크도 그만큼 빨리 실행될겁니다. 

`hdfs-site.xml`을 수정할 시 HDFS에 업로드되는 모든 파일에 대해 설정이 변경이 되며, 특정 파일 블록 사이즈만 변경을 원할 경우 다음 명령어를 사용하면 된다.
`./bin/hadoop distcp -Ddfs.block.size=[HDFS block size][input path][output path]`

distcp는 원래 파일 복사에 사용하는 옵션이지만, dfs.block.size를 지정하여 블록사이즈를 변경합니다.

미국 운항 항공 데이터를 32MB 블록 단위로 재구성하여 실행해보니 9분 이상이 소요되었습니다. 반면에 기존의 64MB 블록 단위는 5분 45초가 나왔었습니다. 더 많은 맬 태스크를 실행했으면 더 좋은 결과가 나와야 할 텐데, 이러한 이유는 태스크트래커가 동시에 실행할 수 있는 맵 태스크 개수가 2개이기 때문입니다. 

하둡은 [0.95 x num_data_nodes ~ 1.75 x num_data_nodes] 를 권장합니다.



### 8.5 JVM 재사용
태스크트래커는 맵 태스크와 리듀스 태스크를 실행할 때 각각 별도의 JVM을 실행합니다. 태스크가 JVM을 실행하는데 걸리는 시간이 1초 내외로 걸려서 사용자가 인지를 못할 뿐이지만 이 또한 전체 잡 실행 시간에 영향을 줄 수 있습니다. 

### 8.6 투기적인 잡 실행
투기적인 (speculative)?
네이버 어학사전에 따르면 투기적인 실행은 "조건 분기를 처리할 때와 같이 어떤 결과가 나올지 모르는 상태에서 다음 명령을 미리 실행하는 일"이라 합니다.

말 그대로 하둡은 어떤 작업이 천천히 진행되고 있는 작업들에 대해서 해결하려 하지 않고 오히려 빨리 처리되고 있는 데이터노드에서 새로 태스크를 생성합니다.

어떤 결과가 나올지 모르면서 예상을 해보기 때문에 투기적인 실행이라고 명칭이 생겼습니다.


이렇게 들었을때 투기적인 실행은 좋은것 같은데 왜 이런 설정을 수정해야 할까요? 투기적인 실행에 대한 설정 기본값은 기본적으로 true로 설정되어 있습니다. 투기적인 실행은 잡이 안정적으로 수행이 되도록 도와주지만 오히려 성능에 역효과를 줄 수 있습니다. 따라서 투기적인 실행 설정은 false로 설정합니다.


[[https://community.cloudera.com/t5/Support-Questions/what-is-speculative-execution/td-p/241741]]

[[https://saturncloud.io/blog/what-is-hadoop-speculative-task-execution-a-guide-for-data-scientists/#:~:text=Hadoop%20speculative%20task%20execution%20is,and%20ultimately%2C%20the%20entire%20job.]]



