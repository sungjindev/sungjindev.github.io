---
title: 스프링 컴포넌트 스캔 파헤치기
categories: [Computer engineering, Backend engineering]
tags: [backend, spring, java, component scan, 백엔드, 스프링, 자바, 컴포넌트 스캔]
---

이번 포스팅에서는 앞서 일일히 직접 구현했던 Spring Bean 등록 과정을 더욱 편리하게 해주는 스프링 프레임워크의 컴포넌트 스캔 기능에 대해 알아보겠습니다.

## 컴포넌트 스캔과 의존 관계 자동 주입
지금까지 앞선 포스팅에서는 순수 Java 코드나 xml을 가지고 AppConfig라는 설정 정보 코드를 직접 구현하여 Spring container에 Spring Bean을 등록하는 과정을 일일히 해줬습니다.    
    
위와 같은 과정은 굉장히 반복적이고 귀찮은 작업이 아닐 수 없습니다... 그래서 스프링에서는 이러한 설정 정보 파일이 없어도 이를 컴포넌트 스캔이라는 기능과 의존 관계 자동 주입 기능을 통해 조금 더 편리하게 구현할 수 있도록 도와줍니다. 먼저 이 과정을 코드 예시로 한번 알아보겠습니다.   
    
```java
 @Configuration
 @ComponentScan(
         excludeFilters = @Filter(type = FilterType.ANNOTATION, classes =
 Configuration.class))
 public class AutoAppConfig {
 }
```

우선 컴포넌트 스캔 기능을 활용하기 위한 AppConfig를 만들어줘야 합니다. 저는 위처럼 AutoAppConfig라는 클래스 만들어 줬고 해당 클래스에 @Configuration, @ComponentScan 애너테이션을 붙여줬습니다.   
    
여기서 @ComponentScan 애너테이션이 하는 일은 @Component 애너테이션이 붙은 모든 클래스를 Spring Bean으로 등록해주는 역할을 하게 됩니다. 사실 위 코드에 excludeFilters 이 부분은 평상시에 쓸 일이 없는 기능이긴 한데, 위 코드에 들어가있는 이유는 기존에 만들어뒀던 AppConfig와 충돌을 방지하기 위해 @Configuration 애너테이션이 붙어있는 모든 클래스는 컴포넌트 스캔의 대상에서 제외시키도록 명시해준 것입니다.    
    
위처럼 해야하는 이유는, @Configuration 애너테이션 내부를 타고 들어가보면 내부적으로 @Component 애너테이션을 달고 있기 때문에 위처럼 예외 지정을 안해주면 지금 만드는 AutoAppConfig의 Spring Bean들과 충돌이 날 수 있기 때문입니다.    
    
여기까지하면, 이제 Configuration 코드는 모든 작성이 끝났습니다. 이제, Spring Bean에 등록해야되는 클래스마다 @Component 애너테이션을 아래와 같이 붙여주기만 하면 됩니다.    
    
```java
 @Component
 public class OrderServiceImpl implements OrderService {
     private final MemberRepository memberRepository;
     private final DiscountPolicy discountPolicy;
    @Autowired
     public OrderServiceImpl(MemberRepository memberRepository, DiscountPolicy discountPolicy) {
         this.memberRepository = memberRepository;
         this.discountPolicy = discountPolicy;
     }
}
```

위처럼하면 OrderServiceImpl이 자동으로 Spring Bean으로 등록되게 됩니다. 지금까지의 상황을 정리해보면 Spring Container에 필요한 Spring Bean들을 모두 등록되어 있지만 이 Bean들을 활용하여 의존 관계 주입을 해주고 있지는 않은 그런 상황입니다.   
    
이를 해결하기 위해선, 저희가 이전에 의존 관계 주입을 위해 생성해두었던 생성자에 @Autowired 애너테이션을 붙여주어야 합니다. @Autowired 애너테이션은 생성자 등에 붙어 자동으로 타입이 일치하는 Spring Bean을 주입해주는 그런 역할을 합니다. 이러한 흐름을 그림으로 정리해보면 다음과 같습니다.    
    
![1](/assets/img/component-scan/1.png){: w="500" h="300" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

우선 위처럼 @ComponentScan이 모든 클래스 Path를 조사하여 @Component 애너테이션이 붙어있는 클래스들을 찾고, 이들을 모두 Spring Bean으로 등록합니다. 이때 Bean 이름과 Bean 객체를 key-value 쌍으로 저장하게 되는데, Bean 이름은 클래스 명의 앞글자만 소문자로 바꾼 채 저장하게 됩니다.   
    
![2](/assets/img/component-scan/2.png){: w="500" h="300" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

그 후 @Autowired 애너테이션이 타입이 맞는 객체들을 Spring Bean에서 찾아 자동으로 의존 관계를 주입해주게 됩니다. 위 그림을 예시로 들면 MemberRepository 타입과 DiscountPolicy 타입인 인스턴스들을 Spring Container에서 찾아 주입해주게 됩니다. 이는 마치 앞서 배운 getBean(MemberRepository.class)와 같다고 생각해도 좋습니다.   
    
하지만 이때, 자동으로 주입될 수 있는 Spring Bean이 여러 개로 중복된다던지 할 때는 문제가 발생할 수도 있는데 이와 관련해서는 뒤에서 자동 의존 관계 주입을 주제로 더욱 자세하게 알아보겠습니다.

## 컴포넌트 스캔 탐색 위치
지금까지 살펴본 컴포넌트 스캔이 만약 모든 Class Path를 다 조회한다면 프로젝트 규모가 커지면 커질수록 훨씬 더 많은 시간이 소요될 것입니다. 그래서 이러한 탐색 위치를 지정해줄 수가 있는데 아래와 같이 basePackage를 사용하면 됩니다.   
    
```java
 @ComponentScan(
         basePackages = "hello.core",
}
```

이렇게 지정해주면 "hello.core"라는 패키지를 시작으로 하위 패키지까지 모두 조회하게 됩니다. 만약 복수 개의 패키지를 탐색 위치로 지정하고 싶으면 다음과 같이 해줄 수도 있습니다.   
    
```java
 @ComponentScan(
         basePackages = {"hello.core", "hello.service"},
}
```

이 외에도 "basePackageClasses"라는 옵션이 있는데 이는 이 옵션으로부터 지정된 클래스가 속한 패키지를 탐색 위치로 잡고 그 하위 패키지를 모두 조회하게 됩니다.   
    
만약에 아무것도 지정하지 않으면 어떻게 될까요? 정답은 바로 @ComponentScan 애너테이션이 붙은 클래스의 패키지가 탐색 시작 위치가 됩니다. 그래서 보통 관례적으로 @ComponentScan이 붙을 Configuration 클래스 같은 경우에는 프로젝트 최상단에 두게 됩니다.    
    
참고로 스프링 부트를 사용하면 스프링 부트의 대표 시작 정보인 @SpringBootApplication 애너테이션도 @ComponentScan 애너테이션을 가지고 있고 이 메인 클래스를 프로젝트 최상단에 두고 있습니다.

## 컴포넌트 스캔 대상
컴포넌트 스캔의 대상이 되는 것은 사실 지금까지 알아본 @Component 애너테이션이 붙은 클래스 외에도 몇가지 더 있습니다. 다음 나열된 애너테이션들이 ComponentScan의 대상이 되고 일부 애너테이션은 아래와 같이 부가적인 기능도 함께 가지고 있습니다.   
    
1. @Controller: 스프링 MVC 컨트롤러로 인식
2. @Repository: 스프링 데이터 접근 계층으로 인식하고 데이터 계층의 예외를 스프링 계층의 예외로 변환
3. @Configuration: 스프링 설정 정보로 인식하고 Spring Bean이 싱글톤을 유지할 수 있도록 추가적인 처리를 해줌
4. @Service: 별도의 추가적인 처리를하진 않지만, 개발자들이 여기에 비즈니스 로직이 담기겠구나 쉽게 유추할 수 있게 해줌

위 부가적인 기능 대부분 쉽게 이해할 수 있을 것이라 생각되지만, @Repository의 예외 변환 기능에 대해 잘 이해가 되지 않으실 수 있습니다. 현실에는 여러가지 데이터 소스들이 있고 이는 저마다 다른 예외를 가지고 있을 것입니다. 아무래도 Repository 계층은 이 데이터 소스에 접근하는 종속적인 계층이다보니 예외 자체를 데이터 소스에 그대로 의존하게 되면 코드의 유연성이 확 떨어지게 됩니다. 따라서 스프링에서는 이런 각양각색의 데이터 계층 예외를 스프링의 예외로 변환하여 처리해주게 됩니다.