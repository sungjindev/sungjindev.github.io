---
title: Spring Test 할 때 선택적으로 Property 사용하기
categories: [Computer engineering, Backend engineering]
tags: [backend, spring, java, property, TestPropertySource, JUnit, 백엔드, 스프링, 자바, 프로퍼티, 테스트 프로퍼티 소스, 제이유닛]
---

이번 포스팅에서는 Spring Test 과정에서 Test 환경마다 다르게 선택적으로 property source를 결정해야할 때가 있는데, 어떻게 구현할 수 있는지 trouble shooting 과정까지 함께 살펴보겠습니다.

## 문제의 발단
Spring 기반으로 사이드 프로젝트를 진행하다가 저희 서비스는 Github Actions를 활용한 CI 파이프라인에서 자동으로 JUnit 테스트가 돌고 있었습니다. 이러한 테스트 환경을 위한 application.yml에는 database source로 h2 인메모리 DB를 사용하고 있었고 test 환경이 아닌 main 환경에서는 MySQL local 환경을 development용으로 사용하고 있었습니다.   
   
그 과정에서 저는 요구 사항에 따른 기능 구현을 위해 h2 DB에는 없고 MySQL에서만 있는 MBR쿼리를 사용하였습니다. 여기서 참고로 MBR이란 Minimum Bounding Rectangle의 약자로 해당 도형을 감싸는 최소 크기의 사각형를 의미합니다. 무튼, MBR을 사용하고 이에 대한 테스트 코드를 작성하다보니 test 환경에서는 database source가 h2로 되어있어서 쿼리를 제대로 해석하지 못하는 오류가 발생하였습니다.   
    
그래서 저는, MySQL에 dependent한 test들에 대해서는 어떤 flag 형식의 변수를 넘겨주던 필터링을해서 해당 테스트들에게는 application-mysql.yml property가 적용될 수 있도록 구현하고자 찾아보기 시작했습니다.   
   
그러던 중, Spring test 4버전부터 있는 @TestPropertySource() 애너테이션을 알게되었고 이를 사용하기로 합니다. 

## @TestPropertySource()
```java
@TestPropertySource(locations = "classpath:application-mysql.yml", properties = "spring.profiles.active=mysql")
class StoreService {
    ...
}
```
사용 방법은 굉장히 간단했습니다. 위처럼 설정해주면 됩니다. 그러면 해당 테스트 클래스가 돌 때는 지정한 application-mysql.yml을 프로퍼티로 사용하게 됩니다.   
   
혹시나 저와 같은 실수를 하지 않으셨으면 하는 마음에 추가적으로 말씀드리면, 해당 애너테이션에서 locations만 지정하고 properties로 active 시켜주지 않으면 오류가 발생합니다. 꼭 저 애너테이션에서 spring.profiles.active를 시켜주실 필요는 없지만 Default 사용중이신 application.yml이나 해당 애너테이션 중 한곳에서는 저렇게 active로 만들어줘야 올바르게 PropertySource로 사용할 수 있습니다.   
    
여기까지 해주고 나면, 해당 테스트 클래스에 대해서는 제가 지정한 application-mysql.yml 프로퍼티를 사용하여 테스트가 실행되게 됩니다. 하지만 이렇게 했다고해서 제 문제 상황을 해결할 수는 없었습니다. 왜냐하면, 로컬에서는 local MySQL에 connection이 원활히 잘 되겠지만 Github Actions로 구현되어있는 Auto CI pipeline에는 별도의 local MySQL 환경을 만들어놓지 않았기 때문에 당연스럽게도 오류가 발생합니다.   
    
그때부터, 어떻게 이 문제를 해결할 수 있을지 고민했습니다. 이를 해결하기 위해서는 Github Actions 상에서는 뭔가 해당 테스트를 Skip 시킬 방법이 필요했는데 관련해서 알아보던 중 제가 사용하고 있는 JUnit5부터 지원하는 DisabledIf 애터네이션에 대해 알게되었습니다.

## @DisabledIfSystemProperty()
@Disabled, @Enabled 관련한 여러 애너테이션이 JUnit 버전 5부터 지원됩니다. 그 중에서 저는 임의로 System Property를 전달해줘서 해당 값이 일치하면 Test를 skip시키도록 구상하였습니다.
```java
@DisabledIfSystemProperty(named = "mysql", matches = "false")
``` 
코드로 보면 위와 같은데, mysql이라는 이름의 System Property의 값이 false인 경우에는 해당 애너테이션이 붙은 테스트 클래스 혹은 테스트 메서드가 skip됩니다. 여기서 System Property의 key:value는 본인이 원하신 걸로 지정하시고 넘겨줄 때만 동일하게 잘 넘겨주면 됩니다.   
   
```
./gradlew test -Dmysql=false
```
이를 테스트해보기 위해 직접 위처럼 -D 옵션을 줘서 mysql=false라는 System Property를 넘겨줬고 Local에서 테스트했을 때는 의도했던 것과 같이 정상적으로 작동했습니다.   
    
하지만, 이상하게 Github Actions에서만 정상적으로 동작하지 않는 문제가 있었고 관련해서 gradle Github Repository에서 issue를 찾아보던 중... 아래와 같은 코드를 build.gradle에 명시적으로 선언해주지 않으면 cli argument로 받은 값을 System Property로 전달하지 않는다는 것을 알게되었습니다. 그래서 build.gradle에 아래와 같이 반드시 추가해주고 gradle update 해주셔야 합니다.    
   
```java
...
test {
	systemProperties = System.properties
}
...
```

이렇게 잘 해결되나 싶었는데, 끝이 아니었습니다. 또다시 Github Actions에서만 에러가 발생합니다.

## 끊임없는 에러

![1](/assets/img/optionally-use-properties/1.png){: w="1000" h="600" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}
    
지금까지 작성한 제 코드를 먼저 살펴보면 위와 같습니다. 그리고 Github Actions Auto CI에서 발생하는 에러는 다음과 같습니다.

![2](/assets/img/optionally-use-properties/2.png){: w="1000" h="600" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

뭔가 Application Context 단에서 에러가 발생하는 것으로 보입니다. 그래서 긴 error stack을 따라가다 보니 아래와 같은 힌트를 발견할 수 있었습니다.

![3](/assets/img/optionally-use-properties/3.png){: w="1000" h="600" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

![4](/assets/img/optionally-use-properties/4.png){: w="1000" h="600" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

마지막 사진의 문장을 보고 확실히 알게 되었습니다. jdbc.url이 제대로 인식되지 못하고 있구나. 저는 해당 테스트 클래스에 application-mysql.yml로 프로퍼티 소스를 설정해놨는데 local MySQL이 없는 환경이라 해당 url에 제대로 접근하지 못하고 있던 것이었습니다.   

그래서 이걸 어떻게 해결할지 고민하다가 아래와 같이 간단히 메서드 쪽에 붙어있던 Disabled 애너테이션을 클래스 단으로 올려서 클래스가 준비되는 시점부터 skip 될 수 있도록 구현하였습니다.

![5](/assets/img/optionally-use-properties/5.png){: w="1000" h="600" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

이제서야 비로소 모든 문제를 해결하고 local 환경과 Github Actions CI 환경에서까지 모두 테스트를 성공적으로 마칠 수 있었습니다.

## 마무리
이번 포스팅에서는 Spring test와 JUnit에서 제공하는 애너테이션들을 활용하여, 서로 다른 테스트 환경일 때 선택적으로 property source를 지정해주고 필요하다면 test를 skip시켜주는 방법에 대해 알아봤습니다. 오랜 시간 삽질하긴 했지만 결국 포기하지 않고 고민한 덕에 끝내 해결해서 다행이라고 생각합니다. 다음에도 프로젝트 진행하면서 기록할만한 좋은 고민이 있으면 함께 공유드리겠습니다.