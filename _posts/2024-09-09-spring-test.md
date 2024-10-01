---
layout: post
title: "안녕하세뇨"
category: example2
---


## 로직간 강결합 문제

회원가입 했을 때 환영 메세지를 보내는 비즈니스 로직을 개발하고자 한다. 보통 아래와 같이 개발하고자 할 것이다.

```java
class JoinService {
	void join() {
		// 회원 가입 로직...
		messageService.sendWelcomeMessage(message);
	}
}
```

위처럼 코드를 작성할 때 무엇이 문제일까? `JoinService`와 `MessageService`가 강하게 결합하는 문제가 있다. 이런 강결합에는 다음과 같은 문제가 있다.
1. 도메인의 비관심사의 실패로 관심사에 영향을 미쳐서는 안 된다. 즉 비관심사가 롤백되어도 관심사는 롤백되어서는 안 된다.
2. 반대로 관심사가 실패하면 비관심사도 같이 실패해야 한다.
3. 즉, 회원가입이 실패하면 메세지도 보내지면 안 되지만, 메세지 보내는 것을 실패해도 회원가입은 되야 한다.

오늘은 이러한 서비스들의 강결합을 이벤트와 이벤트 리스너로 해결해보겠다. 스프링에서 이벤트를 발행하면 특정 클래스에서 이벤트 리스너를 통해 이벤트를 구독함으로서 이벤트를 처리할 수 있다. 이를 적용한 것을 코드로 확인해보자.

```java
class JoinService {
	void join() {
		// 회원 가입 로직...
		JoinedUserEvent joinedEvent = new JoinedEvent();
		applicationEventPublisher.publishEvent(joinedUserEvent);
	}
}
```

여기서는  `JoinService`에서 회원가입 이후 이벤트를 발행할 것이고, 이벤트를 구독하고 있는 이벤트 리스너에서 `MessageService` 작업을 수행할 것이다. 

`JoinService`를 보자. `JoinService`의 join 메소드는 회원가입 로직만 충실하게 수행 하면 되고 그외의 로직은 비관심사이다. 따라서 위 코드에서는 도메인의 주 관심사인 회원가입을 처리한 이후 `JoinedUserEvent`를 발행하였다.

```java
class JoinedUserEventHandler {
	private MessageService messageService;

	@EventListener
	void sendMessage(JoinedUserEvent event) {
		messageService.sendWelcomeMessage(event.getMessage);
	}
}
```

`JoinedUserEventHandler`에서는 `JoinedUserEvent`를 구독하고 있다. 핸들러 입장에서는 어디에서 이벤트가 발행될 지 모르지만, `JoinedUserEvent`가 발행되면 `MessageService`의 `sendWelcomeMesage()` 함수를 호출한다.

이를 통해 관심사와 비관심사의 로직을 따로따로 처리함으로서 두 서비스간의 강결합을 해결하였다. 하지만 이때 주의해야 할 점이 있다.
1. 이벤트는 동작이 발생한 이후 실행되므로 과거 시제를 써야 한다.
2. 이벤트에는 이벤트로 달성하려는 목적이 들어나서는 안 된다. 이벤트에 목적이 기대되면 물리적으로 분리되어도 논리적으로 분리되지 않는다.
   - `applicationEventPublisher.sendMessage(memberId); -[x]`
   - `applicationEventPublisher.joinedUser(memberId); - [o]`
   * `sendMessage`는 이름 자체로 메세지를 보낼 것이라는 ‘의도’가 담겨있으므로 논리적으로 강결합으로 묶여있다. `joinedUser`는 회원가입 되었다는 이벤트를 발행할 뿐 후속 행위에 대한 의도를 담고 있지 않아 논리적 강결합에서 벗어난다.


이를 통해 서비스간의 강결합 문제는 해결된 것 같다. 하지만 이런 고민이 생긴다. 이벤트를 발행한 트랜잭션이 커밋된 이후 이벤트를 수행할 수 없을까?

## 특정 트랜잭션 페이즈 이후 처리되는 방법
쉽게 말해 이런 고민이다. 우리는 회원가입이 정상적으로 커밋되거나 롤백된 이후 이벤트를 처리하고 싶다. 즉, 이벤트를 발행한 서비스의 트랜잭션이 **특정 페이즈**가 되었을 때 이벤트를 수행하고 싶은 것이다. 이럴 때는 스프링이 제공하는 `TransactionSynchronizationManager`를 사용하면 된다. `TransactionSynchronizationManager`는 간단히 `@TrasanctionalEventListener` 어노테이션만 붙이면 사용할 수 있다.

```java
class EventHandlerJoinedUser {
	private MessageService messageService;

	@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
	void sendMessage(JoinedUserEvent event) {
		messageService.sendWelcomeMessage(event.getMessage);
	}
}
```

트랜잭션 페이즈에는 `BEFORE_COMMIT`, `AFTER_COMMIT`, `AFTER_ROLLBACK`, `AFTER_COMPLETION`이 있다. 각각 커밋 이전, 커밋 이후, 롤백 이후, 완료 이후 이벤트가 수행되도록 페이즈를 설정하는 것이다.

위 코드에서는 이벤트 발행한 곳에서 커밋된 이후 환영 메세지를 보내도록 설정했다. 

어노테이션 `@TransactionalEventListener`를 사용할 때 `@TransactionalEventListener`는 `TransactionSynchronizationManager`를 추상화한 것이다. 이를 명심하고 다음 상황으로 넘어가자.

## 요구사항 추가
회원가입할 때 가입 성공 메세지와 더불어 언제 가입했는지 로그도 남기고 싶다. 하지만 이것을 개발하는 것은 너무 쉽다. 하나의 이벤트를 여러 개의 리스너로 구독할 수 있기 때문이다. 잡담이지만, 이렇게 리스너를 사용하면 코드의 양과 사용되는 클래스가 늘어나는 단점이 있지만, 부가 로직의 확장이 너무 간편해진다. 또 회원가입 로직을 건들이지 않아도 된다는 점에서 OCP도 준수할 수 있다.

```java
class EventHandlerJoinedUser {
	private MessageService messageService;
	private JoinLoggingService joinLoggingService;

	@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
	@Transactional(readOnly = true)
	void sendMessage(JoinedUserEvent event) {
		messageService.sendWelcomeMessage(event.getMessage);
	}

	@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
	@Transactional
	void logRegisteredUser(JoinedUserEvent event) {
		joinLoggingService.log(event.getUser);
	}
}
```

트랜잭션 이벤트 리스너를 하나 더 추가했다. 커밋 이후 이벤트가 수행되도록 설정하였고, 커밋 이후 수행되기 때문에 트랜잭션을 따로 선언하여 처리하였다.

하지만 문제가 발생한다.
* 회원 가입 완료되었다.
* 메세지도 잘 갔다.
* **그런데 로깅은 Insert Query가 나가지 않아 DB에 저장되지 않는다.**

트랜잭션도 새롭게 선언하였는데 왜 이런 현상이 발생할까? `@Transactional`의 기본 전파 옵션은 `REQUIRED`이다. 즉, 이미 활성 트랜잭션이 있는 경우 트랜잭션에 참여하며, 그렇지 않으면 자체 트랜잭션을 생성하는 것이다.

`@Transactional == @Transactional(propagation = Propagation.REQUIRED)`


우리는 트랜잭션 페이즈를 `AFTER_COMMIT`, 즉 커밋 이후에 이벤트가 발동되도록 설정했다. 그럼 개발자는 커밋 이후에 트랜잭션이 종료되므로 자연스럽게 트랜잭션은 참여하는 것이 아닌 생성하는 것이라고 생각한다. 하지만 이는 착각이다. 이런 문제는 트랜잭션 이벤트 리스너에서 기인한다.

`@TransactionEventListener`는 기본으로 페이즈를 `AFTER_COMMIT`으로 설정한다. 즉 커밋 이후에 이벤트가 발동되도록 기본 설정되어 있다. 그리고 이것이 우리가 기대한 바이다. 하지만 스프링에서 이를 구현할 때 주의할 점이 있다.

트랜잭션 이벤트 리스너는 트랜잭션 동기화 매니저, `TransactionSynchronizationManager`를 추상화한 것이다. 스프링은 커넥션풀을 얻는 과정부터 트랜잭션을 관리하는 과정까지 추상화하였다. 이때 트랜잭션 동기화 매니저는 커넥션을 보관하고 사용하는 용도로 사용한다. 트랜잭션 동기화 매니저는 **쓰레드 로컬**로 관리된다.

![트랜잭션 동기화 매니저 구조](1.png)
> 위 이미지는 인프런 김영한님의 데이터베이스 1편 강의의 PPT 내용이다.

다시 문제의 상황으로 넘어가자. 이벤트를 구독하고 있는 트랜잭션 이벤트 리스너의 페이즈는 `AFTER_COMMIT`인 상황이다.

1. 회원 가입이 완료되었다. 그 이후 트랜잭션이 커밋된다.
2. `TransactionSynchronizationManager`에서 같은 쓰레드 환경임으로 커넥션을 사용한다. 이때 트랜잭션은 이미 **커밋된 트랜잭션**이다.
3. `@Transactional`이 선언되었다. 기본 전파 옵션은 `REQUIRED`인데 이 옵션은 기존 트랜잭션이 있으면 ‘참여’한다. 상황 1과 2를 거쳐 이미 트랜잭션은 살아 있고 커밋된 트랜잭션에 참여한다.
4. 따라서 INSERT 쿼리가 나가지 않는다. 이미 커밋된 트랜잭션이기 때문이다.

위 사항은 트랜잭션 이벤트 리스너의 페이즈 `AFTER_COMMIT`에 대한 설명은 트랜잭션 동기화의 `afterCommit()` 메소드의 주석으로 확인할 수 있다.

```java
참고: 트랜잭션은 이미 커밋되었지만 트랜잭션 리소스는 여전히 활성 상태이고 액세스할 수 있습니다.
결과적으로, 이 시점에서 트리거된 데이터 액세스 코드는 별도의 트랜잭션에서 실행해야 한다고 명시적으로 선언하지 않는 한
여전히 원래 트랜잭션에 '참여'하여 일부 정리를 수행할 수 있습니다(더 이상 커밋을 따르지 않고!).
따라서 여기에서 호출되는 모든 트랜잭션 작업에는 PROPAGATION_REQUIRES_NEW를 사용하세요.
default void afterCommit() {
}
```

주석에서는 트랜잭션이 이미 커밋되어도, 기존 트랜잭션에 ‘참여’한다고 한다. 따라서 트랜잭션을 선언할 때 전파 옵션을 `REQUIRES_NEW`로 설정하라고 가이드 하고 있다. 참고로 이런 설정은 `AFTER_COMMIT`, `AFTER_ROLLBACK`, `AFTER_COMPLETION` 페이즈에서 해야 한다.

주석에서 스포일러가 되었지만, 이 문제를 해결하기 위해서는 전파 옵션을 `REQUIRES_NEW`로 설정하여 이 문제를 해결할 수 있다. 이 옵션은 트랜잭션이 있어도 참여하지 않고, 새로운 트랜잭션을 사용한다.

```java
class EventHandlerJoinedUser {
	private MessageService messageService;
	private JoinLoggingService joinLoggingService;

	@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
	void sendMessage(JoinedUserEvent event) {
		messageService.sendWelcomeMessage(event.getMessage);
	}

	@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
	@Transactional(propagation = Propagation.REQUIRES_NEW)
	void logRegisteredUser(JoinedUserEvent event) {
		joinLoggingService.log(event.getUser);
	}
}
```

위처럼 트랜잭션 전파 옵션을 설정하면 이벤트를 구독한 핸들러의 트랜잭션이 이벤트를 발행한 클래스의 트랜잭션에 참여하지 않고 새롭게 만들어 처리한다. 트랜잭션이 분리되면서 다음과 같은 이점을 얻었다.

1. 회원가입 로직이 실패하면 이벤트가 실행되지 않는다. 이벤트를 구독하는데 페이즈가 커밋 이후로 설정되어 있기 때문이다.
2. 회원가입 로직이 성공하면 이벤트가 실행된다. 이때 각각 환영 메세지와, 로그를 남긴다. 로그 남기는 것을 실패해도 트랜잭션이 분리되어 있기 때문에 회원가입 로직이 롤백되지 않는다.

참고로 sendMessage() 메소드에 트랜잭션을 선언하지 않았는데, 선언하지 않으면 기존의 커밋된 트랜잭션에 참여한다. sendMessage()는 CUD가 필요없기 때문에 트랜잭션을 선언하지 않았다. 또한 커밋된 트랜잭션의 커넥션이어도 읽기만 하기 때문에 상관 없다.

지금까지 다음과 같은 문제를 해결하였다.
1. 서비스간의 강결합을 이벤트로 해결하였다.
2. 트랜잭션 이벤트 리스너를 도입하여 관심사의 트랜잭션의 페이즈에 따라 이벤트를 시행할지 말지를 결정할 수 있게 되었다.
3. 트랜잭션을 분리함으로서 비관심사의 롤백이 관심사의 롤백으로 이어지지 않도록 설계하였다.

하지만 다음과 같은 문제가 남아 있다. (문제가 참 많다.)

## 동기적인 문제
```java
class EventHandlerJoinedUser {
	private MessageService messageService;
	private JoinLoggingService joinLoggingService;

	@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
	void sendMessage(JoinedUserEvent event) {
		messageService.sendWelcomeMessage(event.getMessage);
	}

	@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
	@Transactional(propagation = Propagation.REQUIRES_NEW)
	void logRegisteredUser(JoinedUserEvent event) {
		joinLoggingService.log(event.getUser);
	}
}
```

`JoinedUserEvent`를 발행하고 구독함으로써 서비스간 강결합이 해결된 것처럼 보인다. 하지만 생각해볼 문제가 있다. 회원가입 -> 환영 메세지 발행이 하나의 쓰레드로 이어지고 있다. 즉, 회원가입부터 환영 메세지가 발행될 때까지 클라이언트는 응답을 기다려야 한다. 예를 들어 회원가입이 1초, 환영 메세지 발행이 3초가 걸렸다면 클라이언트는 총 4초를 기다려야 한다.

그럼 이런 의문이 들 수밖에 없다. 클라이언트가 비관심사를 처리할 때까지 기다려야 할까? 아니다. 그리고 4초까지 기다려줄 착한 클라이언트는 없다. 따라서 이때는 비동기로 처리해야 한다. 비동기로 처리하면 클라이언트는 회원가입 1초만 기다리면 된다.

```java
class EventHandlerJoinedUser {
private MessageService messageService;
	private JoinLoggingService joinLoggingService;

	@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
	@Transactional(readOnly = true)
	@Async
	void sendMessage(JoinedUserEvent event) {
		messageService.sendWelcomeMessage(event.getMessage);
	}

	@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
	@Transactional
	@Async
	void logRegisteredUser(JoinedUserEvent event) {
		joinLoggingService.log(event.getUser);
	}
}
```

메소드 레벨에 `@Async`를 선언함으로서 새로운 쓰레드로 이벤트를 처리한다. 비동기 처리에 관련하여 주의할 점은 이번 포스팅의 범위를 넘는 거 같아 기재하지는 않겠다. 

코드를 보면 트랜잭션의 전파 옵션을 기본으로 다시 돌려놨다. 왜냐하면 메소드를 비동기로 설정하면 새로운 쓰레드가 생성되고, 새로운 쓰레드에는 새로운 트랜잭션 동기화 매니저가 생성된다. 따라서 기존에 이벤트를 발급한 트랜잭션 동기화 매니저와는 상관이 없다. 이에 이벤트 리스너를 비동기로 설정하면 전파 옵션을 기본으로 설정해도 무방하다.


## 참고
- [Knee-deep in Spring Boot, Transactional Event Listeners and CGLIB proxies](https://dev.to/peholmst/knee-deep-in-spring-boot-transactional-event-listeners-and-cglib-proxies-1il9)
- [스프링 이벤트를 활용해 로직간 강결합을 해결하는 방법](https://velog.io/@eastperson/%EC%8A%A4%ED%94%84%EB%A7%81-%EC%9D%B4%EB%B2%A4%ED%8A%B8%EB%A5%BC-%ED%99%9C%EC%9A%A9%ED%95%B4-%EB%A1%9C%EC%A7%81%EA%B0%84-%EA%B0%95%EA%B2%B0%ED%95%A9%EC%9D%84-%ED%95%B4%EA%B2%B0%ED%95%98%EB%8A%94-%EB%B0%A9%EB%B2%95#conclusion)
- https://techblog.woowahan.com/7835/
