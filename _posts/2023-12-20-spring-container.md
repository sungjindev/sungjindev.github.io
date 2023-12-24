---
title: 스프링 컨테이너 완전 정복하기
categories: [Computer science, Object Oriented Programming]
tags: [Object Oriented Programming, object, OOP, spring container, 객체 지향 프로그래밍, 객체, 스프링 컨테이너]
---

이번 포스팅에서는 이전 DI, IoC 포스팅에서 배웠던 DI 컨테이너의 Spring 버전인 Spring container에 대해 기본적인 개념부터 깊이있는 내용까지 모두 다뤄보겠습니다. 그 전에 스프링을 사용하기 전에 순수 Java 코드로 구현한 AppConfig 코드 먼저 살펴보겠습니다.

## AppConfig with 순수 Java
```java
public class AppConfig {
    public MemberService memberService() {
        return new MemberServiceImpl(memberRepository());
    }

    public OrderService orderService() {
        return new OrderServiceImpl(
                memberRepository(),
                discountPolicy());
    }

    public MemberRepository memberRepository() {
        return new MemoryMemberRepository();
    }

    public DiscountPolicy discountPolicy() {
        return new RateDiscountPolicy();
    }
}
```
위 코드를 보면 MemberService, OrderSErvice, MemberRepository, DiscountPolicy라는 추상화된 역할들이 있고 이에 대한 구현체로 MemberServiceImpl, OrderServiceImpl, MemoryMemberRepository, RateDiscuntPolicy 등이 있음을 알 수 있습니다. 위 코드는 생성자를 활용한 의존 관계 주입 방법을 사용한 코드입니다.   
     
이제 이 코드를 Spring 프레임워크의 스프링 컨테이너를 사용해서 바꿔본 코드 살펴보겠습니다.    
     
## AppConfig with Spring framework
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
    @Bean
    public DiscountPolicy discountPolicy() {
        return new RateDiscountPolicy();
    }
}
```
@Configuration, @Bean 등과 같은 애너테이션 등이 추가된 것을 확인할 수 있습니다. 이렇게 @Bean 애너테이션이 붙은 메서드들은 메서드 이름과 동일한 이름으로 스프링 컨테이너라는 곳에 등록이 되게 됩니다. Key-Value 형식으로 등록이되는데 메서드 명이 Key값이고 그 메서드의 리턴 객체가 Value가 됩니다. 또한, **이렇게 Spring container에 등록된 객체를 Spring Bean**이라고 합니다.   
     
지금까지 스프링 컨테이너에 객체를 등록하는 방법까지는 알아봤습니다. 그럼 지금부터는 어떻게 스프링 컨테이너에 접근해서 등록한 객체들을 사용할 수 있는지 코드로 알아보겠습니다.    
     
```java
public class MemberApp {
    public static void main(String[] args) {
//        AppConfig appConfig = new AppConfig();
//        MemberService memberService = appConfig.memberService();

        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
        MemberService memberService = applicationContext.getBean("memberService", MemberService.class);

        Member member = new Member(1L, "memberA", Grade.VIP);
        memberService.join(member);

        Member findMember = memberService.findMember(1L);
        System.out.println("new member = " + member.getName());
        System.out.println("findMember = " + findMember.getName());
    }
}
```
주석 처리된 부분은 순수 자바 코드로 짠다면 저렇게 구현해야함을 보여줍니다. 주석 처리되지 않은 부분 중에서 집중해야될 부분은 ApplicationContext 관련된 부분입니다. 이 ApplicatinContext가 Spring container라고 생각하면 되고 초기 ApplicationContext 객체 인스턴스를 생성해줄 때 저희는 AppConfig 클래스 파일에 @Configuration 애너테이션을 붙여줬으므로 AnnotationConfigApplicationContext의 생성자를 이용해 만들어주게 됩니다.   
    
이렇게 생성한 ApplicationContext 객체 인스턴스에 접근하여 getBean 등으로 아까 등록해준 Spring Bean 객체들을 꺼내 활용할 수 있습니다.   
     
근데 여기까지만 놓고보면, 스프링 컨테이너를 활용하면 더 코드가 복잡해지고 불편한 기능이라고 생각할 수도 있습니다. 그래서 지금부터는 왜 그럼에도 사람들이 스프링 컨테이너를 쓰는지, 스프링 컨테이너를 쓸 때 장점이 무엇인지는 뒤에서 차차 알아보겠습니다.   
     
그것보다도, 사실 스프링은 Bean을 생성하고, 의존 관계를 주입하는 단계가 아래 그림처럼 나누어져 있습니다.    
    
![1](/assets/img/spring-container/1.png){: w="500" h="300" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

![2](/assets/img/spring-container/2.png){: w="500" h="300" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

![3](/assets/img/spring-container/3.png){: w="500" h="300" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

![4](/assets/img/spring-container/4.png){: w="500" h="300" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

위처럼 자바 코드로 스프링 빈을 등록하게되면 생성자를 호출하면서 이미 의존 관계가 엮여있어서 네번째 그림처럼 의존 관계 주입도 한번에 처리가 됩니다. 하지만 **실제로는 스프링 라이프 사이클에 따르면 위 그림의 3,4번째 단계처럼 명확하게 스프링 빈이 생성되는 단계와 의존 관계가 설정되는 단계가 나누어져있습니다.**    
     
## 생성된 Bean들 조회하기
위에서 생성된 Bean들을 조회할 수 있도록 구현한 코드를 보면 다음과 같습니다. 참고로 아래는 테스트 코드입니다.    
    
```java
    AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(AppConfig.class);

    @Test
    @DisplayName("모든 빈 출력하기")
    void findAllBean() {
        String[] beanDefinitionNames = ac.getBeanDefinitionNames();
        for (String beanDefinitionName : beanDefinitionNames) {
            Object bean = ac.getBean(beanDefinitionName);
            System.out.println("bean = " + beanDefinitionName + " object = " + bean);
        }
    }

    @Test
    @DisplayName("애플리케이션 빈만 출력하기")
    void findApplicationBean() {
        String[] beanDefinitionNames = ac.getBeanDefinitionNames();
        for (String beanDefinitionName : beanDefinitionNames) {
            BeanDefinition beanDefinition = ac.getBeanDefinition(beanDefinitionName);

            if (beanDefinition.getRole() == BeanDefinition.ROLE_APPLICATION) {
                Object bean = ac.getBean(beanDefinitionName);
                System.out.println("name = " + beanDefinitionName + "object = " + bean);
            }

        }
    }
```
위 코드 방식에 대해 간단히 살펴보면, 우선 AppConfig.class를 활용해서 AnnotationConfigApplicationContext를 만들어줍니다. 이 ApplicationContext에 getBeanDefinitionNames() 메서드를 사용하면 Spring container에 존재하는 모든 Bean들의 이름을 불러올 수 있습니다. 이렇게 얻어진 각각의 이름들을 이용하여 ac.getBean()을 해주면 bean에 대한 객체 레퍼런스를 얻을 수 있게 됩니다.   
     
여기서 만약 개발자가 직접 구현한 bean만 보고싶다면, 즉 Spring 내부적으로 자동적으로 생성하는 bean들은 제외하고 보고싶다면 ac.getBeanDefinition()을 가지고 각각의 BeanDefinition을 먼저 뽑아준 뒤, 여기에 beanDefinition.getRole()을 사용하여 Role이 BeanDefinition.ROLE_APPLICATION과 일치하는지 보면됩니다. 개발자가 직접 구현한 bean은 ROLE_APPLICATION을 가지게 되고, 스프링 내부적으로 자동 생성된 bean은 ROLE_INFRASTRUCTURE를 갖게 됩니다.    
    
## ApplicationContext란?
그나저나 위에서 명확한 설명 없이 ApplicationContext를 통해 Bean을 가져오는 등 여러가지 작업들을 했었는데 지금부터는 이 ApplicationContext가 무엇인지 알아보겠습니다.   
     
ApplicationContext는 인터페이스이고 사실 이 인터페이스는 BeanFactory라는 인터페이스를 상속받아 구현되어 있습니다. 그 구조를 보면 아래와 같습니다.    
    
![5](/assets/img/spring-container/5.png){: w="200" h="100" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

위 그림을 보면 알 수 있다시피 BeanFactory를 상속받은 ApplicationContext가 있고, 이에 대한 구현체로 AnnotationConfigApplicationContext가 있습니다. 저희가 사용했던 getBean(), getBeanDefinitionNames() 등의 메서드들은 사실 모두 BeanFactory 인터페이스에서 제공하는 기능들입니다. 그렇다면 굳이 BeanFactory를 쓰지 않고 ApplicationContext를 사용하는 이유가 무엇일까요?   
    
그 이유는 바로 ApplicationContext는 이러한 빈 컨테이너와 관련된 기능 외에도 애플리케이션 개발하는데 필요한 많은 부가적인 기능을 제공하고 있기 때문입니다. 그 기능들에 대해 살펴보면 다음과 같습니다.   
    
![6](/assets/img/spring-container/6.png){: w="500" h="300" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

간단히 살펴보면, 클라이언트가 웹 애플리케이션에 들어오는 region에 맞춰서 다국어를 제공해주는 메시지 소스, 환경 변수에 대한 관리, 애플리케이션 이벤트에 대한 관리, 리소스 관리 등을 관장해주는 인터페이스들에 대한 상속까지 모두 되어 있는 것이 ApplicationContext입니다. 그래서 실제로 웹 애플리케이션을 개발할 때 BeanFactory만 사용하는 경우는 거의 없고 ApplicationContext를 많이 이용하게 됩니다. 이러한 BeanFactory나 ApplicationContext를 모두 스프링 컨테이너라고 부릅니다.    
    
## BeanDefinition이란?
지금까지 알아본 것처럼 스프링 컨테이너에 빈을 등록하기 위해서 AppConfig 클래스를 자바로 작성하였습니다. 요즘엔 자바로 작성한 Config 파일을 많이 사용하긴 하지만, 이 외에도 xml이나 자기가 다른 포맷으로 만들어 설정할 수도 있는데, 어떻게 이것이 가능한지 알아보겠습니다.   
    
결론부터 말하면, 이렇게 다양한 포맷으로 Bean에 대한 Config를 할 수 있는 이유는 스프링 프레임워크에서 역할과 구현을 잘 나눠 **BeanDefinition**이라는 추상화에만 의존하기 때문입니다. 이러한 BeanDefinition은 빈 하나당 1개씩 만들어지게 됩니다.   
    
![7](/assets/img/spring-container/7.png){: w="500" h="300" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

![8](/assets/img/spring-container/8.png){: w="500" h="300" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

위 구조에서 볼 수 있는 것처럼 Java, Xml 등 다양한 포맷에 대한 Configuration을 지원하고 각각에 구현되어 있는 BeanDefinitionReader가 이러한 Config 파일을 읽어서 BeanDefinition(Bean에 대한 메타 데이터)을 생성합니다.   
   
스프링 컨테이너는 이렇게 생성된 BeanDefinition에만 의존하고 이 메타 정보를 바탕으로 실제 Spring Bean을 생성하게 됩니다.

## 마무리
이번 포스팅에서는 순수 자바코드로 Bean에 대한 설정 정보를 담고있는 AppConfig 파일을 작성하는 방법부터 ApplicationContext의 구조와 더 나아가 BeanDefinition에 대해 살펴봤습니다. 이를 통해 어떤 구조와 흐름으로 Spring container에 Bean이 생성되는지 알 수 있었습니다. 이번 공부를 하면서도 느낀거지만 스프링 프레임워크는 참 역할과 구현이 잘 나누어져 있다는 생각이 듭니다.    
    

## References
* 인프런 내 김영한 강사님의 스프링 핵심 원리 - 기본편