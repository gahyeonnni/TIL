# private 메서드에 @Transactional이 적용될까?

- 실제 코드를 확인해보면 AbstractFallbackTransactionAttributeSource ( 메서드나 클래스에 붙은 `@Transactional` 정보를 찾아내는 클래스 ) → computeTransactionAttribute (트랜잭션 규칙을 적용해야겠네 하고 결정을 내려주는 함수) 경로로 확인해볼 수 있다.
    - 스프링이 실행될 때 `AbstractFallbackTransactionAttributeSource`가 애플리케이션의 코드를 스캔하다가 `@Transactional`을 발견하면, `computeTransactionAttribute`함수를 실행해서 트랜잭션을 어떻게 걸지 분석을 시작

```sql
@Nullable
	protected TransactionAttribute computeTransactionAttribute(Method method, @Nullable Class<?> targetClass) {
		// Don't allow non-public methods, as configured.
		if (allowPublicMethodsOnly() && !Modifier.isPublic(method.getModifiers())) {
			return null;
		}
```

⇒ @Transactional 애너테이션은 private 메서드에 적용할 수 없다 

- 그 이유는 AOP가 프록시 패턴을 사용하기 때문
    - Spring은 AOP를 사용해서 @Transactional 애너테이션을 처리하고, 이 과정에서 동적으로 프록시 객체를 생성
    - 생성된 프록시 객체는 원래의 Bean 객체를 대신해서 호출
- 프록시는 원본 클래스를 상속하거나 인터페이스를 구현해서, 대상 메서드를 **오**버라이드하는 방식으로 트랜잭션 처리를 끼워 넣는다. 그런데 private 메서드는 오버라이드 대상이 아니기 때문에, 프록시가 해당 메서드를 가로챌 수 없다. → @Transactional을 붙여도 무시된다.
    - @Transactional 애너테이션을 가지고 있는 private 메서드에 접근하려고 해도 프록시 객체를 생성할 수 없기 때문에 해당 애너테이션을 무시
- `@Transactional`이 적용된 public 메서드가 프록시를 통해 호출되면 그 시점에 트랜잭션이 시작되고, 그 안에서 호출되는 내부 메서드(private 포함)는 **같은 트랜잭션 컨텍스트를 공유**한다.

### 스프링 AOP

- **관점 지향 프로그래밍**
    - 관점 지향은 쉽게 말해 어떤 로직을 기준으로 핵심적인 관점, 부가적인 관점으로 나누어서 보고 그 관점을 기준으로 각각 모듈화하겠다는 것
    - 모듈화란 어떤 공통된 로직이나 기능을 하나의 단위로 묶는 것
- AOP에서 각 관점을 기준으로 로직을 모듈화한다는 것은 코드들을 부분적으로 나누어서 모듈화하겠다는 의미
- 이때, 소스 코드상에서 다른 부분에 계속 반복해서 쓰는 코드들을 발견할 수 있는 데 이것을 흩어진 관심사
    - ⇒ 흩어진 관심사를 Aspect로 모듈화하고 핵심적인 비즈니스 로직에서 분리하여 재사용하겠다는 것이 AOP의 취지