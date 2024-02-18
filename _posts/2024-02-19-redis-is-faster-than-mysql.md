---
title: Redis는 MySQL보다 항상 빠른가요? 아니요.
categories: [Computer engineering, Backend engineering]
tags: [backend, spring, java, redis, MySQL, autocorrect, autocomplete, refactoring, performance issue, 백엔드, 스프링, 자바, 레디스, 자동 완성, 성능 개선, 성능 이슈]
---

이번 포스팅에서는 이전 포스팅에서 계속 다뤘던 검색어 자동 완성 구현을 위해 Redis와 MySQL을 각각 사용했을 때 성능을 비교하면서 어떤 것을 사용하는 것이 더 좋은지 알아보겠습니다. **결과적으로 제 상황에서 MySQL이 Redis보다 50배 빨랐습니다.**

## Redis와 MySQL의 성능 비교를 위한 상황
**우선 Redis가 좋을지 MySQL이 좋을지 고민하게 된 배경에 대한 소개**를 드리겠습니다. 이전 세 개의 포스팅에 걸쳐 검색어 자동 완성 기능 구현 과정에 대해 공유드렸었는데, 갑작스럽게 요구 사항이 바뀌는 바람에 "어 이러면 Redis 쓰는 의미가 없지 않나?"라는 의구심이 들어 직접 성능 비교를 해보게 되었습니다.   
    
원래 기존의 검색어 자동 완성 기능에 대한 요구 사항은 사용자가 검색한 검색어를 활용해서 DB에서 맨 앞 글자부터 사용자의 검색어와 매칭되는지 여부를 검사하여 필터링하는 방식으로 구현했었는데, 이번에 요구 사항이 바뀌면서 맨 앞 글자가 아니더라도 중간에 사용자의 검색어가 포함되어 있어도 자동 완성의 대상이 되도록 변경해야만 했습니다. 즉 간단히 말하면 String의 Contains()와 같은 로직으로 변경이 됐습니다.   
   
기존에 열심히 Redis 적용시키고 Refactoring 시켰던 것이 아쉽기도해서 Redis를 사용해 Contains()와 같은 로직을 쉽게 구현할 수 있을지 검색해보다가 일단 Redis에는 scan이라는 기능이 있었는데 이를 통해 정규표현식처럼 String 매칭이 가능하다는 것을 알게 되었습니다. 여기까지만 보면 굉장히 희망적이었는데 이때 **1차 고비**가 한번 찾아옵니다.    
    
**그것은 바로 대소문자 구분 여부**입니다. 저희 프로젝트 요구 사항 상 대소문자 구분이 안되어야 하는데, 즉 **case insensitive해야 하는데 Redis 내부적으로 case insensitive와 관련된 기능 제공을 하고 있지 않다**는 것을 알게됩니다. 그래서 **어떻게 이걸 구현할 수 있을지 고민하다가 생각해낸 방법**이 다음 로직입니다.   
    
1. 검색 대상이 되는 모든 가게명을 모두 대문자 혹은 소문자로 변경하고 해당 값을 Redis내 Hash 자료구조의 Key로 저장
2. 그리고 대문자 혹은 소문자로 변경하기 전 원래 가게명은 해당 Key와 매핑되는 Value로 저장
3. 그 후 Hash Key값에 대한 scan을 통해 Contains 로직 구현
4. 조건에 맞는 Hash Key값들은 매핑 되어있는 Value(원래 가게명)을 가져와 List에 저장

위 로직 그대로 Redis의 Hash 자료 구조를 이용해 구현한 코드를 보며 설명드리겠습니다.

## Redis Hash를 이용한 구현
```java
@Service
public class RedisHashService {
    private final HashOperations<String, String, String> hashOperations;
    private final RedisTemplate<String, String> redisTemplate;

    public RedisHashService(RedisTemplate<String, String> redisTemplate) {
        this.hashOperations = redisTemplate.opsForHash();
        this.redisTemplate = redisTemplate;
    }
    
    private String key = "autocorrect"; //검색어 자동 완성을 위한 Redis 데이터

    //Hash에 field-value 쌍을 추가하는 메서드
    public void addToHash(String field, String value) {
        hashOperations.put(key, field, value);
    }

    public Set<String> findAllValuesContainingSearchKeyword(String searchKeyword) {
        //Redis에서는 case insensitive한 검색을 지원하는 내장 모듈이 없으므로 searchKeyword는 모두 소문자로 통일하여 검색하도록 구현
        //당연히 초기 Redis에 field를 저장할 때도 모두 소문자로 변형하여 저장했고 원본 문자열은 value에 저장!
        Set<String> result = new HashSet<>();   //searchKeyword를 포함하는 원래 가게 이름들의 리스트. 최대 maxSize개까지 저장. 중복 허용하지 않고, 자동 사전순 정렬하기 위해 사용
        final int maxSize = 10;   //최대 검색어 자동 완성 개수

        ScanOptions scanOptions = ScanOptions.scanOptions().match("*" + searchKeyword + "*").build();   //searchKeyword를 포함하는지를 검사하기 위한 scanOption
        Cursor<Map.Entry<String, String>> cursor = hashOperations.scan(key, scanOptions);   //기존 Redis Keys 로직의 성능 이슈를 해결하기 위해 10개 단위로 끊어서 조회하는 Scan 기능 사용

        while (cursor.hasNext()) {  //끊어서 조회하다보니 while loop로 조회
            Map.Entry<String, String> entry = cursor.next();
            result.add(entry.getValue());

            if(result.size() >= maxSize)    //maxSize에 도달하면 scan 중단
                break;
        }
        cursor.close();
        return result;
    }

    public void removeAllOfHash() {
        redisTemplate.delete(key);
    }
}
```

우선 위 코드가 **Redis의 Hash 자료 구조를 사용**하는 서비스 로직입니다. 여기서 메인 로직은 **Case insensitive하게 Redis에서 조회해주는 findAllValuesContainingSearchKeyword 메서드**입니다. 앞서 설명드린 로직과 그대로 구현을 했고 조금 자세히 보면 좋을 부분이, 원래 **Keys라는 Redis 기능을 통해서 Key에 대한 조회를 할 수가 있는데 Redis가 싱글 스레드이다 보니 대규모 데이터에 대해서 Keys로 조회를 하게되면 해당 처리를 하는 동안 계속 Redis를 점유하고 있는 문제가 있어서 Scan이라는 기능이 추가**되었다고 합니다. **Scan은 사용자가 지정한 단위대로 Pagination해서 처리**합니다. **Default 값은 10개**로 되어있습니다.   
    
```java
public List<String> autocorrect(String keyword) {   //검색어 자동 완성 로직
    Set<String> allValuesContainingSearchKeyword = redisHashService.findAllValuesContainingSearchKeyword(keyword);  //case insensitive하게 serachKeyword를 포함하는 가게 이름 최대 10개 반환
    if(allValuesContainingSearchKeyword.isEmpty())
        return new ArrayList<>();
    else
        return new ArrayList<>(allValuesContainingSearchKeyword);   //자동 완성 결과가 존재하면 ArrayList로 변환하여 리턴
}
```

다음 로직은 **자동 완성 로직에 대한 구현이 담긴 Service 단**의 코드입니다. 단순히 방금 전까지 살펴본 findAllValuesContainingSearchKeyword() 메서드를 통해 얻어진 최대 10개의 자동 완성 키워드를 반환해주는 기능을 합니다. 이렇게 했을 때 **약 51200개의 가게를 가지고 있는 저희 데이터에 대해 어느정도의 Response time이 나오는지 확인**해봤습니다.

![1](/assets/img/redis-is-faster-than-mysql/1.png){: w="1000" h="800" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

위 사진처럼 보시다시피 **Redis를 Full scan할 수 있는 searchKeyword로 검색했을 때 약 2.5초**가 나왔습니다. 여기서 **Full scan할 수 있는 searchKeyword란, 제가 자동 완성 키워드가 10개 만들어지면 조회 loop를 도중에 break하도록 구현해 놓았기 때문에 일부러 10개까지 조회되지 않는 검색어**로 검색하였습니다.    
    
그럼 이번에는 **성능 비교를 위해 MySQL의 like 쿼리를 사용해서 조회**해보겠습니다.

## MySQL LIKE를 이용한 구현
```java
public List<String> findAllBySearchKeyword(String searchKeyword) {
    return em.createQuery("select s.displayName from Store s where s.displayName like concat('%', :searchKeyword, '%')", String.class)
            .setParameter("searchKeyword", searchKeyword)
            .setMaxResults(10)
            .getResultList();
}
```

이번에는 서비스쪽 로직은 그냥 Query 결과를 반환해주는 거 밖에 없을 정도로 간단해서 생략하고 **메인 로직인 Repository 단**의 코드를 가져왔습니다. 쿼리도 별로 어려울 것 없이 **LIKE를 활용해서 사용자의 검색어가 포함되는 가게명들을 조회하였고 like를 활용한 pattern matching을 편리하게 하기 위해 concat을 사용해서 검색어 앞뒤에 %를 붙여줬습니다.**   
    
사실 이 쿼리만 봐도 누가 빠를지 대충 예상이 되긴합니다. **MySQL에는 인덱싱을 적용할 수 있기도하고 현재 Repository 단에서부터 애초에 최대 10개로 Limit를 걸고 filtering해서 가지고 오기 때문에 추후 Service 로직에서 그 많은 모든 데이터에 대해 인스턴스화해서 가지고 있을 필요가 없습니다. 이에 따른 메모리나 성능적인 이점 또한 매우 클 것입니다. 그리고 무엇보다 직접 대문자 혹은 소문자화 해가면서 Case insensitive하게 만드는 로직을 거칠 필요도 없습니다. 왜냐하면 MySQL 쿼리 내 like 절은 애초에 Case insensitive하게 동작하기 때문**입니다.

![2](/assets/img/redis-is-faster-than-mysql/2.png){: w="1000" h="800" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

실제로 Response time을 체크해보면 위 사진과 같이 **약 0.05초** 밖에 걸리지 않습니다. **Redis를 사용했을 때보다 무려 50배가 빨라진 것**입니다.

## Redis VS MySQL 결론
지금까지 살펴본 성능 분석 결과를 정리해보면 다음과 같습니다.   
    
**Case insensitive하게 특정 문자열이 포함되는 값들을 Full scan(전수 조사)해야 되는 상황**이 있었습니다. 이때, **Redis에서는 이러한 기능을 내장하여 제공하고 있지 않기 때문에 별도의 추가 로직 필요하다는 점** 그리고 **Redis의 경우 데이터를 Filtering해서 사이즈를 대폭 줄여줄 수 있는 limit 로직을 Service 단보다 더 이른 시점인 Repository 단에서 걸어줄 수 없다는 점들이 성능 저하의 요인**이 되었습니다.   
    
**특히 대용량 데이터의 Filtering 시점이 늦어짐에 따라 대량의 Repository 조회 결과를 고스란히 인스턴스화시켜서 Service 단까지 가져오고 이를 전체 조회해야 된다는 점이 성능 저하의 주된 요소**로 보여집니다.   
    
그래서 **결론은, "인메모리 DB이므로 Redis는 항상 빠를 것이다."라는 생각은 굉장히 잘못된 일반화의 오류이며 Redis의 특징인 Key-Value 자료 구조의 장점이 잘 부각될 수 있도록 캐싱이나 O(1) 조회 등이 필요한 상황에만 적용해서 사용하는 것이 바람직**할 것입니다.



