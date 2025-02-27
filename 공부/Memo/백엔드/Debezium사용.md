debezium 사용해서  DB의 변경사항이 발생할때 마다 차트 변경 하는 것이 목적 
(도커를 이용해서 어플리케이션 외부에서 사용 시 https://debezium.io/documentation/reference/1.5/tutorial.html 해당 링크 대로 해주기만 하면 된다.)

-기본적으로 도커를 사용하지 않고 윈도우에서 돌아가게 시도 
  =>connect-standalone 이용해서 연결하기




-어플리케이션 내부에서 돌아가게 하기 (링크에서 제공되는 예제소스에 빠진 부분이 많아서 정보도 부족해서 일일이 소스를 분석해가며 추가해야했다.)
```
package debezium;

import java.io.File;
import java.io.IOException;
import java.util.Properties;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

import io.debezium.engine.ChangeEvent;
import io.debezium.engine.DebeziumEngine;
import io.debezium.engine.format.Json;

public class DebeziumConfig {
	
	public static void main(String[] args) throws IOException {
		// Define the configuration for the Debezium Engine with MySQL connector...
		File offsetStorageTempFile = File.createTempFile("offsets_", ".dat"); -- 추가
        File dbHistoryTempFile = File.createTempFile("dbhistory_", ".dat"); -- 추가
		final Properties props = new Properties();
		props.setProperty("name", "engine");
		 props.setProperty("connector.class", "io.debezium.connector.mysql.MySqlConnector");  --추가 필수
		props.setProperty("offset.storage", "org.apache.kafka.connect.storage.FileOffsetBackingStore");
		props.setProperty("offset.storage.file.filename", offsetStorageTempFile.getAbsolutePath());
		props.setProperty("offset.flush.interval.ms", "60000");
		/* begin connector properties */
		props.setProperty("database.hostname", "localhost");
		props.setProperty("database.port", "3306");
		props.setProperty("database.user", "root");
		props.setProperty("database.password", "1234");
		props.setProperty("database.server.id", "85744");
		props.setProperty("database.server.name", "demo");
		props.setProperty("database.dbname", "demo");
		props.setProperty("database.history", "io.debezium.relational.history.FileDatabaseHistory");
		props.setProperty("database.history.file.filename", dbHistoryTempFile.getAbsolutePath());
		// whitelist => 사라질예정 , include.list 사용
		// Create the engine with this configuration ...
		try (DebeziumEngine<ChangeEvent<String, String>> engine = DebeziumEngine.create(Json.class).using(props)
				.notifying(record -> {
					System.out.println(record);
				}).build()) {
			// Run the engine asynchronously ...
			ExecutorService executor = Executors.newSingleThreadExecutor();
			executor.execute(engine);
			// Do something else or wait for a signal or an event
		}
		// Engine is stopped when the main code is finished
		catch (IOException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
			System.out.println("오류");
		}

	}

}

```
실행하면  

![image](https://github.com/2nho/personal-study/assets/97571604/cb241399-4073-4bb6-979b-9d38116efb90)

이렇게 로그뜨는걸 보면 제대로 되는걸 확인할 수 있다.


DB에서 특정 데이터베이스 및 테이블만을 CDC하고 싶다면 RelationalDatabaseConnectorConfig 클래스를 확인 !





++ debezium engine api 사용하기 (내장 어플리케이션 버전) 위와 설정은 비슷   
자바 어플리케이션으로 main 메소드를 통해 실행하면 잘 실행됐지만 스프링 웹 프로젝트에 적용시키면 계속 쓰레드가 죽어버리길래 왜그런가 찾다보니   
Caused by: java.lang.ClassNotFoundException: org.slf4j.event.Level 오류를 로그를 설정해보니 발견했는데 혹시하고 버전 업데이트를 2.0.4로 하니 잘되서 알아보니   
아래와 같이 1.7.15버전부터 클래스를 지원한다고 한다. 기존 사용하던 버전은 1.7.5 였다..  

[mysqld]
log-bin=debeziumCDC
binlog_format=ROW
max_allowed_packet=256M

설정도 추가 


![image](https://github.com/2nho/personal-study/assets/97571604/b640e798-ffa0-4e86-ae9c-6481eadb4ed8)



DEBUG나 , INFO 모드로 로그 설정시  아래와같이 오류발생  
![image](https://github.com/2nho/personal-study/assets/97571604/4992626e-faff-4ae8-ad35-1227e6dc5eb6)





###24.9.9 추가 
데비지움은 어떻게 실시간 캡쳐하는가 ...? while문..  
![image](https://github.com/user-attachments/assets/880d98a9-399f-4b89-b9f8-94e91d905daa)


