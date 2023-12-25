---
title: 싱글톤 컨테이너에 대한 모든 것
categories: [Computer engineering, Backend engineering]
tags: [backend, spring, java, singleton container, 백엔드, 스프링, 자바, 싱글톤 컨테이너]
---

이번 포스팅에서는 앞서 배웠던 Spring container에 이어서 싱글톤 컨테이너(Singleton container)에 대해 알아보는 시간을 가지도록 하겠습니다.

## 웹 애플리케이션과 싱글톤
우선 싱글톤(Singleton)이라는 것에 대해 간단히 알아보겠습니다. 싱글톤은 다들 익히 아시고 계신 것처럼 객체가 현재 나의 java jvm 안에 딱 하나만 있도록하는 그러한 디자인 패턴을 말합니다. 즉, 클래스의 인스턴스가 딱 1개만 생성되도록 보장하는 디자인 패턴입니다.    
    
이러한 싱글톤 패턴은 웹 애플리케이션 프로젝트에 많이 사용되게 되는데 그 이유를 알아보겠습니다. 일반적으로 웹 애플리케이션은 아래 그림처럼 보통 여러 클라이언트가 동시에 요청을 하는 경우가 많이 발생하게 됩니다. 

![1](/assets/img/singleton-container/1.png){: w="500" h="300" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

위 그림처럼 클라이언트 A,B,C가 각각 동시에 memberService에 대해 요청을 하게 되면 일반적으로 싱글톤 패턴이 적용되지 않은 경우 그림처럼 서로 다른 memberService 객체를 만들어 반환을 해주게 됩니다. 서비스의 규모가 커지면 커질수록 이러한 메모리 낭비는 엄청 더 커질 것입니다. 이때 필요한 것이 바로 싱글톤 패턴입니다. 객체를 1개씩만 생성하도록 만들고 이미 생성된 객체를 공유하는 방식으로 구현하면 메모리 낭비를 막을 수 있습니다.   
   
이러한 싱글톤 패턴을 순수 Java로 구현해보면 아래와 같습니다.

```java
public class SingletonService {

    //1. static 영역에 객체를 딱 1개만 생성해둔다.
    //이렇게 하면 자바가 뜰 때 얘기 생성되어 올라간다.
    private static final SingletonService instance = new SingletonService();

    //2. public으로 열어서 객체 인스턴스가 필요하면 오직 이 static 메서드를 통해서만 조회하도록 허용한다.
    public static SingletonService getInstance() {
        return instance;
    }

    //3. 생성자를 private로 선언해서 외부에서 new 키워드를 사용한 객체 생성을 못하게 막아버린다.
    private SingletonService() {
    }

    public void login() {
        System.out.println("싱글톤 객체 로직 호출");
    }
}
```

위 코드처럼 static을 통해 오직 1개의 객체만을 생성할 수 있도록 만들고, private 생성자를 통해 외부에서 추가적인 객체 생성을 못하게 막아버리면 싱글톤 패턴 구현이 간단하게 끝납니다.   
   
하지만, 이렇게 만능일 것만 같은 싱글톤 패턴은 다음과 같은 수많은 단점들이 존재합니다. 지금부터는 싱글톤 패턴의 단점에 대해 알아보겠습니다.

## 싱글톤 패턴의 여러가지 단점
1. 싱글톤 패턴을 구현하는 코드 자체가 많이 들어간다.
2. 의존 관계상 클라이언트가 구체 클래스에 의존한다. 따라서, DIP를 위반하게 된다.
3. 클라이언트가 구체 클래스에 의존해서 OCP 원칙을 위반할 가능성이 높다.
4. 인스턴스를 미리 정해놓기 때문에 유연하게 테스트하기 어렵다.
5. 내부 속성을 변경하거나 초기화하기 어렵다.
6. private 생성자로 자식 클래스를 만들기 어렵다.
7. 결론적으로 유연성이 떨어지게 된다.
8. 그래서 안티 패턴이라고 불리기도 한다.

위에 나열한 것처럼 코드가 길어지는 장점을 시작으로 자바가 뜰 때 인스턴스가 미리 정해져 있기 때문에 코드의 유연성이 떨어지고 이로인해 변경하기가 굉장히 어려워진다는 단점들이 존재합니다.   
하지만, 스프링 프레임워크는 위 싱글톤 패턴의 문제점들을 전부 다 해결하고 제거하면서 객체를 싱글톤으로 관리해줍니다. 어떻게 이게 가능한 것인지는 뒤에서 알아보겠습니다.

## 싱글톤 컨테이너
스프링 컨테이너는 싱글톤 패턴의 문제점들을 해결하면서 객체 인스턴스를 싱글톤(오로지 1개만 생성)으로 관리합니다. 이전 포스팅에서 계속 알아봤던 Spring Bean이 바로 싱글톤으로 관리되는 인스턴스입니다.   
    
앞서 살펴본 것처럼 스프링 컨테이너는 Key-Value 쌍으로 각 1개의 Key에 대해 하나의 객체 레퍼런스만을 가지고 있습니다. 이렇게 싱글톤으로 구현이 되어있고 이렇게 싱글톤 객체를 생성하고 관리하는 기능을 **싱글톤 레지스트리**라고 부릅니다.  스프링 컨테이너의 이런 기능 덕분에 싱글톤 패턴의 모든 단점을 해결하면서 객체를 싱글톤으로 유지할 수 있게 되는 것입니다. 참고로 스프링에서 싱글톤 방식만 지원하는 것은 아닙니다. 위와 다르게 요청할 때마다 새로운 객체를 생성해서 반환하는 기능도 스프링에서 별도로 제공하고 있습니다. 이는 뒤에서 차차 알아보겠습니다.   
    

## 싱글톤 방식을 사용할 때 주의점
스프링 프레임워크를 통해, 싱글톤 방식의 여러가지 단점들을 제거하며 사용할 수 있다고 배웠습니다. 하지만, 이때도 싱글톤 방식을 사용할 때 주의해야하는 부분이 있습니다. 아무리 단점들을 제거했다고 하더라도 **싱글톤 패턴이라는 것은 객체 인스턴스를 공유하는 방식이기 때문에 stateful(상태를 유지)하게 설계하면 안됩니다.** 즉, stateless(상태가 없도록)하게 설계해야 합니다.   
   
stateless하게 설계하기 위해서는 특정 클라이언트에 의존적인 필드가 있으면 안되고, 특정 클라이언트가 값을 변경할 수 있는 필드가 있으면 안됩니다. 또한, 가급적 읽기만 가능하게 만들어야되고 필드 대신에 자바에서 공유되지 않는 지역변수, 파라미터, ThreadLocal 등을 사용하는 것이 좋습니다. 싱글톤인 Spring Bean의 Field에 공유값을 설정해버리면 정말 큰 장애가 발생할 수 있습니다. 그러므로, 싱글톤 방식을 사용할 때는 공유하는 필드를 만들지 않도록 주의해야 합니다.

## 스프링 컨테이너가 싱글톤을 보장하는 방법
지금부터는 스프링 컨테이너가 어떻게 싱글톤을 보장해줄 수 있는지 알아보겠습니다. 우선 그 전에 앞서, 저희가 기존에 작성했던 AppConfig 코드를 가져와봤습니다.
```java
@Configuration
 public class AppConfig {
     @Bean
     public MemberService memberService() {
         return new MemberServiceImpl(memberRepository());
     }
     @Bean
     public OrderService orderService() {
         return new OrderServiceImpl(
                 memberRepository(),
                 discountPolicy());
}
     @Bean
     public MemberRepository memberRepository() {
         return new MemoryMemberRepository();
     }     
    ... 
}
```

위 코드를 보면 memberService, orderService, memberRepository를 Bean으로 등록하는 과정에서 memberRepository 메서드만 해도 여러번 중복 호출되어 여러 개의 MemoryMemberRepository 객체 인스턴스가 생성되어야 할 것처럼 보입니다. 즉, 위 코드를 순수 Java 언어적으로만 생각해보면 싱글톤이 보장될 수가 없는 구조입니다. 하지만, 위 코드를 실제로 테스트해보면 스프링이 어떤 동작을 해줘서 위 메서드는 각각 1번씩만 호출되고 싱글톤을 보장해준다는 것을 알 수 있습니다. 어떻게 이런 일이 가능한 것일까요?   
   

![2](/assets/img/singleton-container/2.png){: w="500" h="300" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

그 비밀은 바로 @Configuration 애너테이션에 있습니다. 사실 Spring에서는 저희가 구현한 AppConfig를 그대로 사용하지 않습니다. 위 그림처럼 cglib라는 바이트 코드 조작 라이브러리를 활용해서 기존 AppConfig를 상속받아서 다른 클래스를 하나 만들고 새로 만든 이 클래스를 Spring Bean에 등록해버립니다. 그리고 이 **Spring이 바꿔치기 해버린 AppConfig@CGLIB가 싱글톤이 보장되도록 해줍니다.** 이게 보장될 수 있는 이유는 아래 코드를 보면 바로 이해가 되실겁니다.

```java
 @Bean
 public MemberRepository memberRepository() {
    if (memoryMemberRepository가 이미 스프링 컨테이너에 등록되어 있으면?) { return 스프링 컨테이너에서 찾아서 반환;
    } else { //스프링 컨테이너에 없으면
    기존 로직을 호출해서 MemoryMemberRepository를 생성하고 스프링 컨테이너에 등록 return 반환
    }
}
```

위 코드처럼 @Bean 애너테이션이 붙은 메서드마다 위처럼 스프링 컨테이너에 이미 등록되어 있는지 여부를 확인하여 그에 따라 이미 존재한다면 새로 생성하지 않고 스프링 컨테이너에서 기존에 존재하는 인스턴스를 꺼내 반환해주고 이미 존재하고 있지 않다면 스프링 컨테이너에 등록과 동시에 새로운 객체를 반환해주는 방식으로 저희가 구현해놨던 AppConfig의 바이트 코드를 변환하여 사용하게 됩니다. 바로 이렇게 cglib를 사용해서 스프링이 싱글톤을 보장할 수 있게 해주는 것입니다.   
   
즉 다시 말해, 만약에 @Configuration 애너테이션을 지워버리면 Spring Bean은 정상 등록되겠지만 정말 순수 Java 코드 로직대로 돌아 싱글톤 패턴이 깨지게됩니다.

## 마무리
이로써 싱글톤 패턴의 정의부터 싱글톤 패턴을 사용했을 때 장단점, 그리고 이러한 단점을 스프링 프레임워크는 어떻게 극복하여 처리하고 있는지 내부 로직까지 모두 자세하게 알아봤습니다.   
    
이번 포스팅에서 배운 것처럼 싱글톤에 대해 잘 알고 사용하면 메모리를 아낄 수 있고 많은 장점이 되지만, 필드를 공유해서 사용하는 등의 잘못된 사용은 정말 큰 장애를 불러일으킬 수 있기 때문에 조심해서 사용해야 함을 배울 수 있었습니다.    
    
앞으로 Spring 프로젝트를 진행하면서 싱글톤 패턴이 적용된 Spring Bean의 경우 stateless하게 공유되는 필드를 만들지 않도록 주의해야겠습니다.
