---
title: 스프링 의존 관계 자동 주입 파헤치기
categories: [Computer engineering, Backend engineering]
tags: [backend, spring, java, Dependency Injection, DI, 백엔드, 스프링, 자바, 의존 관계 주입]
---

이번 포스팅은 앞서 배운 @Autowired의 연장선으로 스프링 내 의존 관계 자동 주입에 대해 깊이 있게 알아보는 시간을 가지려고 합니다.

## 의존 관계 주입 방법 4가지
의존 관계 주입 방법에는 다음과 같이 크게 4가지가 있습니다.   
    
1. 생성자 주입
2. 수정자 주입(setter 주입)
3. 필드 주입
4. 일반 메서드 주입

**첫 번째로, 생성자 주입은 이름 그대로 생성자를 통해서 의존 관계를 주입받는 방법**이고, 지금까지 살펴본 것들이 모두 생성자 주입입니다. 생성자의 특징 덕분에 객체 인스턴스가 생성되는 해당 시점, 딱 1번만 호출되는 것이 보장됩니다.   
    
따라서, 이는 보통 불변성이 보장되어야하거나 반드시 의존 관계 주입이 일어나야하는 상황에 사용됩니다. 그래서 이는 보통 final 키워드와 함께 많이 사용됩니다. final 키워드가 변수에 붙게되면 값에 대한 초기화가 반드시 이루어져야하고 나중에 값 변경이 안되도록 보장해줍니다. 이를 이용해서 불변성과 필수값을 보장할 수 있습니다.   
    
그리고 클래스 내 생성자가 딱 1개만 있으면 @Autowired를 생략해도 자동으로 주입됩니다. 또 알아두어야 할 것이 @Autowired 애너테이션을 통한 자동 주입은 당연히, Spring Bean 클래스 내에서 사용할 때만 효과가 있습니다.    
    
**두 번째로, 수정자 주입 방식**이 있습니다. 이는 setter라 불리는 필드의 값을 수정하는 메서드를 통해서 의존 관계를 주입하는 방법입니다. setter라는 메서드 자체가 변경을 열어두는 메서드이다 보니 선택적으로 의존 관계 주입이 필요하거나 추후 의존성에 대한 변경 가능성이 있을 때 주로 사용하게 됩니다.   
    
선택적인 의존 관계 주입이 필요한 상황에 대해 설명드리겠습니다. 예를 들어, @Autowired로 의존 관계 주입을 하려고 할 때 주입해야 되는 대상이 되는 객체가 Spring Bean에 등록되어 있지 않을 경우 @Autowired의 기본 동작은, 오류를 발생시키는 것입니다. 이때 선택적으로 주입할 대상이 없어도 동작하게 하려면 @Autowired(required = false)로 지정하여 해결해줄 수 있습니다.   
    
**세 번째로, 필드 주입 방식**도 있습니다. 이름 그대로 필드에 바로 주입하는 방식입니다. 이는 생성자나 setter와 같은 코드가 없어도 되서 코드가 간결해지고 좋아보이지만, 써서는 안될 Anti pattern입니다.    
    
왜냐하면, 필드만 있고 setter와 같이 추후에 해당 필드에 접근해서 변경시킬 방법이 없기 때문에 나중에 테스트 시에 Spring을 띄우지 않고선 필드에 값을 넣을 방법이 없어 테스트하기가 어려워지기 때문입니다. 즉, DI 프레임워크를 띄우지 않고선 테스트 시 아무것도 할 수가 없습니다. 물론 스프링을 띄워서 하는 통합 테스트의 경우 코드를 간결하게 하고 빠르게 테스트 코드를 작성하기 위해서는 필드 주입 방식을 사용해도 괜찮습니다. 하지만, 실제 애플리케이션 코드에서는 필드 주입은 사용하지 않는 것을 강력히 추천드립니다.   
    
**마지막으로, 일반 메서드를 통해서 의존 관계를 주입**받을 수도 있습니다. 이는 사실 setter 메서드랑 다를 것이 없고 차이점이라면 setter는 한 필드에 대해 개별적으로 메서드가 존재하는 반면에 일반 메서드는 한번에 여러 필드를 주입 받는 것이 가능합니다. 하지만, 보통 생성자 주입이나 수정자 주입을 통해 구현하기 때문에 이 방식은 실제로 잘 사용되지 않습니다.

## @Autowired 옵션 처리
앞서 수정자 주입 방식에서 잠깐 살펴봤었는데, 주입할 Spring Bean이 없어도 동작되어야 할 상황이 있을 수 있습니다. 이때 @Autowired를 그대로 사용하면 required 옵션의 default 값이 true로 설정되어 있어서 자동 주입 대상이 Spring Bean으로 존재하지 않게되면 오류가 발생하게 됩니다.    
    
이 문제를 해결하기 위해 자동 주입 대상을 Optional하게 처리하는 방법은 다음과 같이 3가지가 있습니다.
1. @Autowired(required = false): 자동 주입할 대상이 없으면 메서드 자체가 호출이 안됨
2. org.springframework.lang.@Nullable: 자동 주입할 대상이 없으면 null이 입력됨
3. Optional<>: 자동 주입할 대상이 없으면 Optional.empty가 입력됨

위 말만 보아선 잘 이해가 안되기 때문에 각각에 대한 예시 코드를 가져와봤습니다.
```java
//1번 예시
@Autowired(required = false)
public void setNoBean1(Member noBean1) {
    System.out.println("noBean1 = " + noBean1);
}

//2번 예시
@Autowired
public void setNoBean2(@Nullable Member noBean2) {
    System.out.println("noBean2 = " + noBean2);
}

//3번 예시
@Autowired
public void setNoBean3(Optional<Member> noBean3) {
    System.out.println("noBean3 = " + noBean3);
}
```

위와 같이 코드를 작성하고 테스트를 실행해보면, require = false 값으로 준 1번 예시의 경우는 아예 setNoBean1()이라는 메서드 자체가 호출이 되지 않습니다. 2번 예시와 같이 required = true 옵션(이게 default 값)을 쓰고 주입할 대상에 @Nullable 애너테이션을 붙여주면 null 값이 들어간 대상이 주입됩니다. 마지막 세 번째 예시 같은경우에는 주입할 대상을 Optional로 한번 감싸준 경우인데, 이 경우에는 Optional.empty가 주입되게 됩니다.

## 결국은, 생성자 주입을 사용해라!
앞서 모든 의존 관계 주입 방법에 대해 알아봤습니다. 이미 어느정도 눈치채셨겠지만, 결국은 생성자 주입을 최우선순위로 사용하고 Optional한 처리가 필요한 경우에만 수정자 주입을 추가적으로 사용해주는 것이 좋습니다.   
그 이유를 알아보면, 우선 대부분의 의존 관계 주입은 한번 일어나면 애플리케이션 종료 시점까지 의존 관계를 변경할 일이 없습니다. 오히려 변하면 의도치 않은 버그를 초래할 수 있기 때문에 불변해야만 합니다. 그래서 불변성을 보장할 수 있는 생성자 주입을 사용하는 것이 좋습니다.   
    
그리고 수정자 주입과 다르게 수정자 주입은 주입할 대상을 누락하게 되면 Null Point Exception이 Run time에 발생하게 되는데 final 키워드와 생성자 주입을 함께 사용하게 되면 주입할 대상을 누락하게 되었을 때 컴파일 오류가 발생하여 개발자가 오류를 빨리 인지할 수 있게 되고 선제적인 조치를 취할 수가 있습니다.   
    
마지막으로, final 키워드를 사용할 수 있다는 장점이 있습니다. 수정자와 같은 경우에는 생성자 이후에 호출되기 때문에 생성자 호출 시점까지의 초기화만 허용하는 final 키워드를 사용할 수가 없게 됩니다. 따라서, 불변성을 보장해주는 유용한 키워드인 final을 사용하기 위해서는 반드시 생성자 주입 방식을 선택해주어야만 합니다.   
    
이 내용들을 모두 정리해보면, 기본적으로 생성자 주입을 사용하는 것을 강력히 권장하고 Optional한 처리가 필요한 경우에만 수정자 주입 방식을 통해 옵션을 부여하면 됩니다.

## 생성자 주입을 더욱 간결하고 편리하게 사용하는 방법
생성자 주입 방법이 좋은 것은 알겠지만, 필드 주입에 비하면 코드도 길어지고 번거롭습니다. 이러한 문제를 Lombok이라는 라이브러리를 통해 손쉽게 해결할 수 있습니다. 롬복 라이브러리를 사용한 코드를 보며 알아보겠습니다.    
    
```java
@Component
@RequiredArgsConstructor
public class OrderServiceImpl implements OrderService {
    private final MemberRepository memberRepository;
    private final DiscountPolicy discountPolicy;
}
```

@RequiredArgsConstructor가 롬복에서 제공하는 애너테이션인데, 이름 그대로 필수적으로 필요한 Argument들에 대한 생성자를 자동으로 만들어주는 애너테이션입니다. 이걸 붙이게 되면 실제로 코드에서 보이진 않지만 애너테이션 프로세서라는 기능을 이용해서 컴파일 시점에 해당 생성자 코드를 자동으로 생성해줍니다. 이는, final이 붙은 필수적인 필드들을 모두 모아 이를 Argument로 사용하는 생성자를 자동으로 만들어줍니다.    
     
이렇게 하게되면, 롬복에 의해 final 필드에 대한 생성자가 만들어지고 이때 해당 클래스 내 생성자는 1개 뿐이기 때문에 @Autowired를 붙이지 않아도 자동으로 의존 관계 주입이 일어나 의도하는 결과를 쉽게 얻을 수 있게 됩니다.

## @Autowired에서 조회할 Bean이 중복되는 문제
지금까지 알아본 것처럼 @Autowired는 Spring Container에서 타입 기반으로 필요한 Bean들을 조회하여 자동적으로 의존 관계를 주입해주는 역할을 합니다. 이는 마치 getBean(DiscountPolicy.class)와 같은 역할을 하게 되고 이렇게 타입을 기반으로 조회하게 되면 하위 타입까지 모두 함께 조회하게 됩니다. 따라서, DiscountPolicy 하위에 FixDiscountPolicy랑 RateDiscountPolicy가 있고 이 두개 모두 Spring Bean으로 등록되어 있다면 어떻게 될까요?   
    
정답은, 예견한대로 **NoUniqueBeanDefinitionException 오류가 발생**합니다. 당연스럽게도 @Autowired는 타입 기반으로 조회를 하고 1개의 결과를 기대했지만, 여러 개의 후보 Bean들이 나와 무엇을 주입해야될지 몰라 발생한 예외입니다. 지금부터는 이어서 이 문제를 어떻게 해결할 수 있을지 알아보겠습니다.   
    
이때 해결할 수 있는 방법으로 크게 3가지 방법이 있습니다.
1. @Autowired 필드 이름 혹은 파리미터 이름 매칭
2. Bean으로 등록할 클래스에 @Qualifier 추가 -> @Autowired 파라미터에 @Qualifier 추가 -> 둘이 매칭
3. @Primary 사용

우선 첫 번째 해결 방법은 필드 이름 혹은 파리미터 이름을 매칭시켜주는 방법입니다. 예를들어 @Autowired가 생성자에 붙어있고 여러 개의 타입이 조회되어 문제가 되고 있는 생성자 파라미터명으로 discountPolicy를 쓰고 있다면, 이를 rateDiscountPolicy로 바꿔주기만 하면 해결할 수 있습니다.    
    
이것이 가능한 이유는 @Autowired의 특별한 로직 때문입니다. @Autowired는 처음 타입으로 조회를 시도하다가 여러 개의 타입이 조회되버리면 그땐 파라미터 이름이나 필드 이름으로 조회해서 Unique한 Bean이 매칭되는지 검토하게 됩니다. 이때 이름을 같게 해주면 해당 Spring Bean이 주입되는 방식입니다.   
    
다음 두 번째 방법으로 @Qualifier라는 추가 구분자를 붙여 해결할 수 있습니다. 이는 추가적인 구분자를 제공하는 것 뿐이지 결코 Bean 이름 등을 변경하는 것은 아닙니다. 예시 코드를 보겠습니다.   
    
```java
@Component
@Qualifier("mainDiscountPolicy")
public class RateDiscountPolicy implements DiscountPolicy {
    ...
}
```

위처럼 Spring Bean으로 등록할 클래스에 추가적인 @Qualifier 애너테이션을 통해 추가 구분자를 붙여줍니다. 이 추가 구분자의 이름은 "mainDiscountPolicy"로 주었습니다. 그 이후 @Autowired가 붙은 생성자 코드에 아래와 같이 작성해줍니다.   
    
```java
 @Autowired
 public OrderServiceImpl(MemberRepository memberRepository, @Qualifier ("mainDiscountPolicy") DiscountPolicy
 discountPolicy) {
    this.memberRepository = memberRepository;
    this.discountPolicy = discountPolicy;
}
```

위처럼 @Autowired가 붙은 메서드의 파라미터에 특정 @Qualifier로 매칭을 시켜주면 해당 객체가 주입되게 됩니다.   
    
마지막 방법은 @Primary 애너테이션을 통해 더 높은 우선순위를 가지는 Bean 클래스를 지정하는 방법입니다. 동작 원리는 굉장히 간단합니다. 타입으로 Spring Bean 조회 시 중복으로 조회될 때 @Primary 애너테이션이 붙은 Bean 클래스가 선택됩니다. 다만 여기서 좀 헷갈릴 수 있는 부분이 그렇다면, @Qualifier와 @Primary가 경쟁하면 누가 우선 순위가 더 높을까요? 이런 고민에 대한 정답은 보통 좀 더 Specific한 것, 그러니까 좀 더 명시적이고 자세한 것이 우선 순위가 높습니다. 따라서 명시적으로 지정해준 @Qualifier가 @Primary보다 더 높은 우선 순위를 가지게 되어 둘이 경쟁한다면 @Qualifier로 매칭된 Bean이 주입되게 됩니다.

## 마무리
지금까지 살펴본 의존 관계 자동 주입을 모두 정리하여 어떻게 운영하면 좋을지 정리해봤습니다. 먼저, 자동 의존 관계 주입과 수동 의존 관계 주입 둘 중에서 어떤 것을 실무에 활용하면 좋을지 생각해봤을 때, ComponentScan이나 Autowired와 같은 자동 의존 관계 주입을 적극적으로 사용하는 것이 좋습니다.    
    
왜냐하면, Spring과 Spring Boot 모두 설정 정보 같은 경우에는 자동화하는 추세이기도 하고 무엇보다 프로젝트의 사이즈가 작을 땐 크게 번거롭지 않지만 관리할 Bean 많아지면 많아질수록 설정 정보를 관리하는 것 자체가 부담이 됩니다. 그리고 결정적으로 자동 의존 관계 주입을 하더라도 OCP, DIP를 똑같이 지키면서 실용성을 챙길 수 있다는 장점이 있다.   
    
그렇다면 수동 Bean 등록은 언제 사용하는 것이 좋을까요? 이에 대해 정답은 없지만, 설정 정보를 한눈에 모아서 봐야하는 경우, 즉 비즈니스 로직에 다형성이 필요하여 여러가지 케이스로 나뉘는 경우 명시적으로 하나의 설정 파일에서 수동 Bean 등록을 해주어 모아주면 나중에 유지보수하거나 다른 개발자들이 한눈에 보기 좋기 때문에 그럴 때 활용하면 좋습니다.   
    
혹은 Controller, Service, Repository 계층 이외에 데이터 베이스 연결이나 공통 로그 처리 등과 같은 공통 관심사, 즉 Application 전반적으로 공유해서 쓰는 로직일 경우에는 가급적 수동 Bean 등록을 사용해서 문제가 발생했을 때 조금 더 명확히 드러날 수 있도록 해주는 것이 좋습니다.   
    
다시 한번 요약해보면, **"편리한 자동 의존 관계 주입을 적극적으로 사용하되, Application 전반에 광범위하게 사용되는 기술적인 객체들은 한 눈에 설정 파일에서 쉽게 볼 수 있게 수동 등록을 해주고, 예외적으로 Controller, Service, Repository 레이어와 같은 비즈니스 로직이라고 하더라도 다형성을 적극적으로 활용하는 비즈니스 로직은 모아 보여주기 위해 수동 등록을 고민해보는 것이 좋다."** 정도로 요약하며 이번 포스팅을 모두 마치도록 하겠습니다. 감사합니다.

## References
* 인프런 내 김영한 강사님의 스프링 핵심 원리 - 기본편