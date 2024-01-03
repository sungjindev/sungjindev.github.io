---
title: JPA 기본키 생성 전략과 @GeneratedValue 사용 시 주의점
categories: [Computer engineering, Backend engineering]
tags: [backend, spring, java, JPA PK generation strategy, GeneratedValue, 백엔드, 스프링, 자바, JPA 기본키 생성 전략]
---

이번 포스팅에서는 JPA 기본키 생성 전략의 종류와 특징에 대해 알아보고 이와 관련된 @GeneratedValue 애너테이션 사용 시 주의점에 대해 알아보겠습니다.

## 사건의 발생
사이드 프로젝트를 진행하다가 처음에 아무 생각 없이 기본키를 생성하기 위해 @Id와 @GeneratedValue 애너테이션을 사용했습니다. 그 후 데이터베이스에 접속해 생성된 테이블과 스키마를 확인하던 중 이상한 점을 발견하게 됩니다.   
    
![1](/assets/img/jpa-pk-strategy/1.png){: w="250" h="150" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

이전 프로젝트에서 사용했을 때와는 다르게 위 화면처럼 제가 생성하지 않은 seq라는 테이블이 함께 생성되어 있었습니다. 테이블 스키마를 살펴보니 뭔가 Auto_increment와 같은 기능을 전담하는 테이블 같아 보여서 @GeneratedValue 애너테이션에 대해 찾아보던 중 원인을 발견하게 됩니다.

## JPA 기본키 생성 전략
그 원인은 뒤에서 살펴보고, 이 원인을 이해하기 위해서는 우선 JPA 기본키 생성 전략에 대해 알아야합니다. JPA를 통해 엔티티와 데이터베이스의 테이블을 매핑할 때 PK로 사용하고자 하는 필드 위에 @Id 애너테이션을 붙여 테이블의 PK와 연결시켜줄 수 있습니다. 참고로 이때 컬럼명을 따로 지정해주지 않으면 관례에따라 field명을 CamelCase로 바꿔 컬럼명으로 지정하게 됩니다.   
    
여기까지하면 엔티티의 식별자 필드와 테이블의 PK를 매핑만 시킨겁니다. 여기서 별다른 액션을 취하지않으면, 엔티티를 생성할 때마다 우리는 수동으로 PK 값을 넣어줘야합니다. 이는 번거로우므로 보통 @GeneratedValue 애너테이션을 사용하여 자동으로 넣습니다. 이 @GeneratedValue 애너테이션의 기본키 생성 전략에는 총 4가지가 존재합니다. 지금부터 하나씩 파헤쳐보겠습니다.

## IDENTITY 전략
```java
@GeneratedValue(strategy = GenerationType.IDENTITY)
```
우선 첫 번째로, 위와 같이 지정해줄 수 있는 IDENTITY 전략이 있습니다. 이는 기본키 생성 관련된 작업을 데이터베이스에 위임하는 전략입니다. 즉, 처음에 id 값에 아무것도 든 것없이 DB한테 넘겨주면 DB가 알아서 AUTO_INCREMENT 해줍니다. IDENTITY 전략은, em.persist()로 객체를 영속화 시키는 시점에 곧바로 insert 쿼리가 DB로 전송되고, 거기서 반환받은 식별자 값을 가지고 1차 캐시에 엔티티를 등록시켜 관리합니다.   
    
이게 굉장히 특이한 부분인데, 원래 JPA는 persist() 시점이 아닌 트랜잭션 커밋 시점에 쓰기 지연 저장소에 차곡차곡 모아놓은 SQL을 한번에 DB로 전송되고, 거기서 반환받은 식별자 값을 가지고 1차 캐시에 엔티티를 등록시켜서 관리하는게 정석입니다. 하지만, IDENTITY 전략은 앞서 말씀드린 것처럼 기본키 생성에 대한 권한을 데이터베이스에 위임하기 때문에, DB에 쿼리가 들어간 뒤 DB에서 AUTO_INCREMENT 같은 액션을 취한 뒤에서야 JPA가 해당 PK값을 알 수 있습니다. 하지만, 영속성 컨텍스트로 엔티티를 관리하려면 1차 캐시에 저장할 key값으로 id 값을 들고 있어야하기 때문에 persist() 시점에서 PK를 알기 위해 특이하게 바로 쿼리를 날리는 것입니다.   
    
그래서, IDENTITY 전략의 단점으로 일반적인 JPA 쿼리 호출 방식과 다르게 모아서 쿼리를 보낼 수 없다는 점을 꼽을 수 있지만 이 부분에 있어서 크게 비약적인 성능차이가 나지는 않는다고 합니다.

## SEQUENCE 전략
```java
@Entity
@SequenceGenerator(
  name = "STORE_SEQ_GENERATOR", 
  sequenceName = "STORE_SEQ", 
  initialValue = 1,
  allocationSize = 1)
public class Member {
  @Id
  @GeneratedValue(strategy = GenerationType.SEQUENCE,
                  generator = "STORE_SEQ_GENERATOR")
  private Long id; 
}
```
두 번째로 살펴볼 전략은 SEQUENCE 전략입니다. 이는 일부 데이터 베이스에 존재하는 특별한 오브젝트인 Sequence Object를 사용하는 방법입니다. Sequence Object은 유일한 값을 순서대로 생성해주는 기능을 합니다. 또한, 테이블마다 별도로 Sequence Object를 두어 관리하는 것도 가능합니다. 테이블마다 별도로 따로 관리하고 싶으면 위 코드에 보이는 것처럼 @SequenceGenerator의 sequenceName 옵션을 추가해주면 됩니다. 이 전략은 데이터베이스에서 Sequence를 제공하는지 안하는지 여부에따라 사용할 수 있을지 없을지가 갈리게 됩니다. 대표적으로 MySQL은 Sequence 기능을 별도로 제공하지 않고 있습니다. 그래서 주로 Oracle, H2, PostgreSQL 등에서 쓰입니다.   
    
참고로, SEQUENCE 전략은 사용하기 전에 DB에 Sequence를 미리 생성해주어야 하며 IDENTITY 전략과 마찬가지로 기본키 생성을 관리하기 때문에 DB를 통해 조회해야 PK 값을 알 수 있습니다. 하지만 IDENTITY와 다르게 이미 SequenceGenerator를 생성한 이후에 Entity를 생성하게 되므로 값을 얻어올 수가 있습니다. 그래서 em.persist() 하기 전에 DB에서 Sequence 값을 가져오면 문제가 없습니다. 그렇게 가져온 PK값을 해당 객체내 id 값으로 넣어주면 됩니다. 그렇게 되면 JPA의 일반적인 흐름처럼 트랜잭션이 커밋되는 시점에 차곡차곡 쌓아둔 Insert 쿼리가 날아가게 됩니다. 즉, IDENTITY와 다르게 Insert 쿼리를 persist() 시점에 바로 날리지 않아도 되어 버퍼링을 할 수 있게 된 것입니다.   
    
하지만 Sequence의 단점으로, DB에서 필요할 때마다 Sequence 값을 조회해서 가져와야되기 때문에 이걸 너무 자주하게 되면 성능상 저하가 올 수 밖에 없습니다. 그래서 도입된 것이 allocationSize입니다. 이 옵션의 default 값은 50인데, 처음 DB에 Sequence 값을 가져오도록 call을 하면, DB에 Sequence를 한번에 50 더해놓고 메모리상에서 1개씩 차감해서 쓰는 방식입니다. 이를 통해 Sequence 조회 횟수를 줄일 수 있습니다.   
    
그렇다면, Sequence를 최대한 한번에 미리 많이 가져와서 메모리에 들고있으면 되지 않냐고 생각할 수도 있는데 이러면 Sequence 번호에 큰 공백이 생기는 문제가 발생할 수 있습니다. 왜냐하면, DB에 Sequence 숫자를 올려놓은 다음에 휘발성을 지닌 메모리에 가져오기 때문에 중간에 애플리케이션이 내려갔다 올라오게 되면 사용하지 않은 Sequence 번호들은 사라져서 그대로 Skip되어 넘어가게 됩니다. 이게 그렇게 큰 문제는 아니지만 그래도 낭비가 되는 것이므로 가급적이면 적절히 50~100으로 설정하는 것이 좋습니다.   

## TABLE 전략
```java
@Entity
@TableGenerator(
	name = "STORE_SEQ_GENERATOR",
    table = "CUSTOM_SEQUENCE",
    pkColumnValue = "USER_SEQ",
    allocationSize = 1
)
public class Store {
    @Id
    @GeneratedValue(
    	strategy = GenerationType.TABLE,
        generator = "STORE_SEQ_GENERATOR"
    )
    private long id;   
}
```
다음은 TABLE 전략입니다. 이 전략은 보통 Sequence Object를 지원하지 않는 데이터베이스에서 Sequence 전략처럼 사용하고 싶을 때 사용할 수 있는 전략입니다. 이는 키 생성 전용 테이블을 별도로 하나 만들어서 데이터베이스 시퀀스 오브젝트를 흉내내는 역할을 합니다. TABLE 전략도 SEQUENCE 전략과 비슷하게 사용하기 전에 미리 키 생성 전용 테이블을 미리 만들어줘야 합니다.   
    
그리고 TABLE 전략은 SEQUENCE 전략과 내부 동작 방식이 거의 같습니다. 하지만, 차이점이라면 TABLE 전략은 값을 조회하기 위해 1번 조회하고, 그 후 값 증가를 직접 시켜줘야 하기 때문에 1번 Update 쿼리를 날려 SEQUENCE 전략 대비 2배의 비용이 소용됩니다. 그래서 비용적으로 비효율적이며, DB에서는 주로 지원하는 방식에 따라 관례적으로 쓰는 전략이 있기 때문에 Sequence를 지원하지 않는 MySQL인데 Cost issue를 가져가면서까지 이걸 굳이 모방해서 쓸 이유는 없습니다.   
    
## AUTO 전략
마지막으로 알아볼 전략은 AUTO 전략입니다. 이름 그대로 hibernate.dialect에 설정된 DB 방언 종류에 따라, 하이버네이트가 자동으로 전략을 선택하게끔 위임하는 전략입니다. 이건 프로젝트 초기에 테스트 등의 가벼운 용도가 아니라면 사용하지 않는 것을 권장드립니다. 왜냐하면, 데이터베이스 방언과 여러 버전에 따라 자동으로 전략을 선택해주기 때문에 관례와 어긋나는 경우도 많아 혼란스럽기도하고 성능적인 이슈가 있을 수 있습니다. 특히, Hibernate 버전 5.0이상부터는 당연하게도 IDENTITY 전략을 사용할 것만 같은 MySQL에 대해 AUTO 전략은 TABLE 전략을 선택하여 적용시킵니다. 그래서 되도록이면, 같이 프로젝트를 진행하는 팀원들과 정한 키 생성 전략에 맞춰 직접 적용해주는 것이 좋습니다.   
    
## 마무리
지금까지 이렇게 JPA 기본키 생성 전략에는 무엇들이 있고 각각 어떤 특징들을 가지게 되는지 알아봤습니다. 이번 포스팅을 꼼꼼하게 정독해오신 분이라면 포스팅 초기에 제가 소개해드린 문제의 상황이 왜 일어난 것인지 이미 정답을 알고 계실겁니다.   
저는, Hibernate 6점대 버전을 사용하고 있었고 그래서 Default Strategy인 AUTO 전략이 MySQL에 TABLE 전략을 자동으로 적용시켜 맞닥뜨렸던 문제입니다.   
    
이번 포스팅에서 정리해본 내용처럼, 가급적이면 프로젝트 내에서 사용하는 데이터베이스의 관례와 같은 전략을 명시적으로 지정하여 사용해주는 것이 좋습니다.