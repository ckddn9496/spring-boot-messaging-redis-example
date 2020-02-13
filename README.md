## Spring boot - Messaging with Redis 예제
Spring 가이드 Messaging with Redis를 공부하며 만든 프로젝트입니다.

### Publisher / Subscriber Pattern
Publisher / Subscriber Pattern (발행-구독 모델)은 Messaging Pattern 중 하나로 발행자(Publisher)와 구독자(Subscriber)가 메시지를 주고받는 패턴이다.
Observer Pattern과 유사한 점이 있지만 서로가 서로를 인지할 필요가 없어 더 결합도가 낮다는 점, 비동기 방식으로 동작한다는 점 등의 차이를 가진다. 

Redis는 캐싱 뿐만이 아니라 Pub/Sub 모델도 지원해준다. 이번 예제 프로젝트에서는 RedisTemplate을 이용하여 message를 송신하고, 수신 후 확인하는 기능을 만들 것이다.

* 참고자료
   * [예제 프로젝트](https://spring.io/guides/gs/messaging-redis/)
   * [Observer Pattern과 Pub/Sub Pattern의 차이점](https://jistol.github.io/software%20engineering/2018/04/11/observer-pubsub-pattern/)
   * [redis 설치](https://redis.io/download)
   * [redis Pub/Sub 사용법](https://redis.io/topics/pubsub)
***

### <h1>프로젝트 구조</h1>
먼저 Pub/Sub 기능을 위해 Redis를 설치하여야 한다. 필자는 리눅스환경에서 Redis를 설치하여 실행하였다. 그 후 메시지를 기다리는 Receiver를 만들고 메시지를 수신할 때 까지 애플리케이션이 대기하도록 하자.
RedisTemplate을 이용해 메시지를 송신하고 이를 수신한 애플리케이션은 메시지를 출력하고 종료한다.

### 준비
먼저 Redis를 설치하자
- [redis 설치](https://redis.io/download)

```shell script
$ wget http://download.redis.io/releases/redis-5.0.7.tar.gz
$ tar xzf redis-5.0.7.tar.gz
$ cd redis-5.0.7
$ make

$ src/redis-server
``` 
Command창에 `redis-server`를 입력하여 아래와 같은 출력을 확인하면 Redis서버가 정상적으로 동작하고있다는 것이다.
```shell script
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.4 (3e403927/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 483
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

483:M 14 Feb 2020 00:03:27.073 # Server initialized

```

의존성에 `spring-boot-starter-data-redis`를 추가해주자
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

Spring Data Redis에서는 Redis와 메시지를 주고받기 위한 components를 제공한다. 우리는 이 component를 이용하여
* A connection factory
* A message listener container
* A Redis template
을 설정해야 한다.

Docs에서는 각각의 설정에 대해 설명헤주고 있다.
> You will use the `Redis template` to `send messages`
, and you will register the `Receiver` with `the message listener container` so that it will `receive messages`.
The `connection factory` drives both `the template and the message listener container`
, `letting them connect to the Redis server`.

쉽게 풀어보자면 위에서 필요한 세 가지는 각각 아래와 같은 역할을 한다
* `RedisTemplate` - 메시지 송신
* `Receiver`, `MessageListenerContainer` - 메시지 수신
* `ConnectionFactory` - Redis 서버와 연결

```java
@SpringBootApplication
public class MessagingRedisApplication {

	private static final Logger LOGGER = LoggerFactory.getLogger(MessagingRedisApplication.class);

    // ConnectionFactory - Redis 서버와 연결
	@Bean
	RedisMessageListenerContainer container(RedisConnectionFactory connectionFactory,
			MessageListenerAdapter listenerAdapter) {

		RedisMessageListenerContainer container = new RedisMessageListenerContainer();
		container.setConnectionFactory(connectionFactory);
		container.addMessageListener(listenerAdapter, new PatternTopic("chat"));

		return container;
	}

    // Receiver, MessageListenerContainer - 메시지 수신
	@Bean
	MessageListenerAdapter listenerAdapter(Receiver receiver) {
		return new MessageListenerAdapter(receiver, "receiveMessage");
	}

	@Bean
	Receiver receiver() {
		return new Receiver();
	}

    // RedisTemplate - 메시지 송신
	@Bean
	StringRedisTemplate template(RedisConnectionFactory connectionFactory) {
		return new StringRedisTemplate(connectionFactory);
	}

	public static void main(String[] args) throws InterruptedException {

		ApplicationContext ctx = SpringApplication.run(MessagingRedisApplication.class, args);

		StringRedisTemplate template = ctx.getBean(StringRedisTemplate.class);
		Receiver receiver = ctx.getBean(Receiver.class);

		while (receiver.getCount() == 0) {

			LOGGER.info("Sending message...");
			template.convertAndSend("chat", "Hello from Redis!");
			Thread.sleep(500L);
		}

		System.exit(0);
	}
}
```
`listenerAdapter`는 수신자인 메시지 수신자 등록을 도와주기 위해 만든 함수이다.
`messageListener`는 반드시 `MessageListenerAdapter`이어야 하지만 `Receiver`는 POJO이므로 Wrapper의 역할을 해주기 위해 존재한다.

`StringRedisTemplate`은 `RedisTemplate`의 종류이다. 특별한점은 `key`,`value`값으로 이루어진 Redis에서 `key`,`value`의 타입을 모두 `String`으로 지정한다.

`ConnectionFactory`는 `RedisConnectionFactory`를 받아 Redis 서버와 연결하고
`MessageListenerAdapter`를 이용하여 **chat** 이라는 토픽에 대해 Receiver를 설정했다(구독자).

`main()`에서 redisTemplate은 `convertAndSend("chat", "Hello from Redis!");`을 통해 **chat**이라는 토픽에대해 **Hello from Redis!** 라는 메시지를 송신한다. 이를 수신한 `recevier`는 카운트가 증가하고 `loop`에서 빠져나와 종료되게 될 것이다.


### 결과

실행 후 로거가 `Sending message...`를 출력하고 메시지를 송신한다. Receiver는 이를 수신하여 count가 증가했으며 loop를 빠져나오고 프로그램이 종료된것을 확인할 수 있다.
```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.2.2.RELEASE)

2020-02-14 01:44:12.442  INFO 12180 --- [           main] c.e.m.MessagingRedisApplication          : Starting MessagingRedisApplication on DESKTOP-C0T6I7B with PID 12180 (C:\spring_guide\gs-messaging-redis-master\initial\target\classes started by ckddn in C:\spring_guide\gs-messaging-redis-master\initial)
2020-02-14 01:44:12.448  INFO 12180 --- [           main] c.e.m.MessagingRedisApplication          : No active profile set, falling back to default profiles: default
2020-02-14 01:44:13.541  INFO 12180 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Multiple Spring Data modules found, entering strict repository configuration mode!
2020-02-14 01:44:13.547  INFO 12180 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data Redis repositories in DEFAULT mode.
2020-02-14 01:44:13.653  INFO 12180 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 29ms. Found 0 Redis repository interfaces.
2020-02-14 01:44:15.551  INFO 12180 --- [    container-1] io.lettuce.core.EpollProvider            : Starting without optional epoll library
2020-02-14 01:44:15.553  INFO 12180 --- [    container-1] io.lettuce.core.KqueueProvider           : Starting without optional kqueue library
2020-02-14 01:44:17.316  INFO 12180 --- [           main] c.e.m.MessagingRedisApplication          : Started MessagingRedisApplication in 5.889 seconds (JVM running for 7.12)
2020-02-14 01:44:17.319  INFO 12180 --- [           main] c.e.m.MessagingRedisApplication          : Sending message...
2020-02-14 01:44:17.341  INFO 12180 --- [    container-2] com.example.messagingredis.Receiver      : Received<Hello from Redis!>

Process finished with exit code 0
```
