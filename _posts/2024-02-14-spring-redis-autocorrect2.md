---
title: Spring 프로젝트에서 Redis를 사용하여 빠른 검색어 자동 완성 구현하기 (2)
categories: [Computer engineering, Backend engineering]
tags: [backend, spring, java, redis, autocorrect, autocomplete, 백엔드, 스프링, 자바, 레디스, 자동 완성]
---

이번 포스팅에서는 이전 포스팅에 이어서 Spring 프로젝트에서 실제로 Redis를 사용하여 검색어 자동 완성 기능을 코드로 구현해보고 그때 마주친 이슈들과 해결 방법들을 공유드리려 합니다.

## 검색어 자동 완성 로직을 위해 단어 쪼개기
이전 포스팅에서 자동 완성 로직을 어떻게 구현할 수 있을지 소개드렸습니다. 이번에는 해당 로직을 바탕으로 코드를 직접 구현해보려 합니다. 그래서, 혹시 이전 포스팅을 읽지 않은 채로 이번 포스팅을 보고 계시다면 잘 이해가 되지 않으실 수도 있으므로 반드시 이전 포스팅을 읽고 오시길 권장드립니다.   
    
앞서 소개해드린 로직처럼 우선 **검색의 대상이 되는 데이터들을 음절 단위로 한 글자씩 쪼개어 Redis에 저장**해야 합니다. 간단한 알고리즘이므로 추가적인 설명은 코드로 대체하겠습니다.

```java
private String suffix = "*";    //검색어 자동 완성 기능에서 실제 노출될 수 있는 완벽한 형태의 단어를 구분하기 위한 접미사

private void saveAllSubstring(List<String> allDisplayName) { //MySQL DB에 저장된 모든 가게명을 음절 단위로 잘라 모든 Substring을 Redis에 저장해주는 로직
    // long start1 = System.currentTimeMillis(); //뒤에서 성능 비교를 위해 시간을 재는 용도
    for (String displayName : allDisplayName) {
        redisSortedSetService.addToSortedSet(displayName + suffix);   //완벽한 형태의 단어일 경우에는 *을 붙여 구분

        for (int i = displayName.length(); i > 0; --i) { //음절 단위로 잘라서 모든 Substring 구하기
            redisSortedSetService.addToSortedSet(displayName.substring(0, i)); //곧바로 redis에 저장
        }
    }
    // long end1 = System.currentTimeMillis(); //뒤에서 성능 비교를 위해 시간을 재는 용도
    // long elapsed1 = end1 - start1;  //뒤에서 성능 비교를 위해 시간을 재는 용도
}
```

별다른 설명이 필요없을 정도로 로직 그 자체를 그냥 Java로 옮긴 직관적인 코드입니다. **하지만, 이 코드 그 자체를 Production level에서 사용하기에는 성능상 문제가 있었습니다. 관련해서 포스팅 마지막에 공유**드리겠습니다.    
    
## Redis 관련 Service단 구현
이전 코드에서 redisSortedSetService와 그와 관련된 메서드들이 나오는데 이는 제가 직접 구현한 Redis관련 Service 레이어입니다. 코드 먼저 살펴보겠습니다.
    
```java
@Service
@RequiredArgsConstructor
public class RedisSortedSetService {    //검색어 자동 완성을 구현할 때 사용하는 Redis의 SortedSet 관련 서비스 레이어
    private final RedisTemplate<String, String> redisTemplate;
    private String key = "autocorrect"; //검색어 자동 완성을 위한 Redis 데이터
    private int score = 0;  //Score는 딱히 필요 없으므로 하나로 통일

    public void addToSortedSet(String value) {    //Redis SortedSet에 추가
        redisTemplate.opsForZSet().add(key, value, score);
    }

    public Long findFromSortedSet(String value) {   //Redis SortedSet에서 Value를 찾아 인덱스를 반환
        return redisTemplate.opsForZSet().rank(key, value);
    }

    public Set<String> findAllValuesAfterIndexFromSortedSet(Long index) {
        return redisTemplate.opsForZSet().range(key, index, index + 200);   //전체를 다 불러오기 보다는 200개 정도만 가져와도 자동 완성을 구현하는 데 무리가 없으므로 200개로 rough하게 설정
    }
}
```

이 코드가 Redis 서비스 코드인데, 보시면 아시겠지만 **redisTemplate에서 이미 편의 기능을 많이 제공해주고 있어서 그냥 메서드를 가져와 쓰기**만 하면 됩니다. 제가 이전 포스팅에서 앞으로 자동 완성 기능 구현하면서 사용할 Redis command 몇가지를 소개해드린 적이 있는데 그 커맨드가 이렇게 편리하게 메서드로 제공되고 있었습니다. 또한, 앞서 말씀드린 것처럼 **저희는 정렬이 필요하기 때문에 SortedSet인 ZSet을 사용**했습니다.    
     
## 검색어 자동 완성 Service단 구현
지금까지 자동 완성 로직 구현을 위한 모든 준비를 마쳤습니다. 음절 쪼개기 로직부터 RedisTemplate를 활용한 Redis 관련 서비스 로직에 대해 알아봤는데 지금부터는 이를 이용해 검색어 자동 완성 기능을 구현하는 전체 코드를 소개해드리려 합니다. 저는 프로젝트에서 Store의 displayName(가게명)에 대한 검색어 자동 완성 기능을 구현했기 때문에 이와 관련된 네이밍으로 작성되어 있는 점 참고 부탁드립니다.   
    
```java
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class StoreService {
    private final StoreRepository storeRepository;
    private final RedisSortedSetService redisSortedSetService;
    private String suffix = "*";    //검색어 자동 완성 기능에서 실제 노출될 수 있는 완벽한 형태의 단어를 구분하기 위한 접미사
    private int maxSize = 10;    //검색어 자동 완성 기능 최대 개수

    @PostConstruct
    public void init() {    //이 Service Bean이 생성된 이후에 검색어 자동 완성 기능을 위한 데이터들을 Redis에 저장 (Redis는 인메모리 DB라 휘발성을 띄기 때문)
        saveAllSubstring(storeRepository.findAllDisplayName()); //MySQL DB에 저장된 모든 가게명을 음절 단위로 잘라 모든 Substring을 Redis에 저장해주는 로직
    }

    private void saveAllSubstring(List<String> allDisplayName) { //MySQL DB에 저장된 모든 가게명을 음절 단위로 잘라 모든 Substring을 Redis에 저장해주는 로직
        // long start1 = System.currentTimeMillis(); //뒤에서 성능 비교를 위해 시간을 재는 용도
        for (String displayName : allDisplayName) {
            redisSortedSetService.addToSortedSet(displayName + suffix);   //완벽한 형태의 단어일 경우에는 *을 붙여 구분

            for (int i = displayName.length(); i > 0; --i) { //음절 단위로 잘라서 모든 Substring 구하기
                redisSortedSetService.addToSortedSet(displayName.substring(0, i)); //곧바로 redis에 저장
            }
        }
        // long end1 = System.currentTimeMillis(); //뒤에서 성능 비교를 위해 시간을 재는 용도
        // long elapsed1 = end1 - start1;  //뒤에서 성능 비교를 위해 시간을 재는 용도
    }

    public List<String> autocorrect(String keyword) { //검색어 자동 완성 기능 관련 로직
        Long index = redisSortedSetService.findFromSortedSet(keyword);  //사용자가 입력한 검색어를 바탕으로 Redis에서 조회한 결과 매칭되는 index

        if (index == null) {
            return new ArrayList<>();   //만약 사용자 검색어 바탕으로 자동 완성 검색어를 만들 수 없으면 Empty Array 리턴
        }

        Set<String> allValuesAfterIndexFromSortedSet = redisSortedSetService.findAllValuesAfterIndexFromSortedSet(index);   //사용자 검색어 이후로 정렬된 Redis 데이터들 가져오기

        List<String> autocorrectKeywords = allValuesAfterIndexFromSortedSet.stream()
                .filter(value -> value.endsWith(suffix) && value.startsWith(keyword))
                .map(value -> StringUtils.removeEnd(value, suffix))
                .limit(maxSize)
                .toList();  //자동 완성을 통해 만들어진 최대 maxSize개의 키워드들

        return autocorrectKeywords;
    }
}
```

saveAllSubstring 로직은 가장 처음 소개드렸던 로직이라 넘어가고 autocorrect 로직에 대해 간단히 설명드리겠습니다. 해당 로직에서 핵심은 사용자 검색어와 일치하는 데이터의 Redis 상 인덱스를 찾은 뒤 그 이후에 나타나는 정렬되어있는 데이터를 뭉텅이로 가져와 저희가 완전한 단어 여부를 구분하기 위해 사용했던 suffix(*)로 끝나는지, 그리고 사용자가 검색한 검색어로 시작하는지 여부를 체크합니다.   
    
**suffix에 대해 체크하는 것은 이전 포스팅에서 설명드린 것처럼 불완전한 단어가 자동 완성 결과로 노출되는 것을 막기 위함**이고, **사용자가 검색한 검색어로 시작하는지 여부를 체크하는 이유는 아무리 Redis 상에 사전순으로 정렬되어 있다고 하더라도 사용자 검색어 이후의 대량의 데이터를 가져오다보면 전혀 관련 없는 자동 완성 검색어가 나타나는 순간이 있기 때문에 이를 걸러주기 위한 용도**라고 생각해주시면 될 것 같습니다.    
     
그리고 이 StoreService단에서 눈여겨보면 좋을 부분이 **@PostConstruct 애너테이션이 붙어있는 init() 로직**입니다. **저희가 자동 완성 기능을 구현하기 위해 Redis에 데이터를 저장하는 이런 과정들을 저희 Backend Application이 실행된 이후로 한번만 실행되면 되기 때문에 저는 해당 StoreService에대한 Spring Bean이 만들어진 직후 해당 로직을 실행시켜 Redis에 초기 데이터들이 세팅될 수 있도록 구현**하였습니다.

## 실행 결과
![1](/assets/img/spring-redis-autocorrect2/1.png){: w="1000" h="800" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

![2](/assets/img/spring-redis-autocorrect2/2.png){: w="1000" h="800" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

위 화면에서 보시는 것처럼 **정상적으로 동작하며 API Response time도 27ms로 굉장히 준수**합니다. 하지만 여기에는 아주 큰 문제가 있었는데, 그건 바로 아까 **@PostConstruct 애너테이션을 붙여 실행했던 Init() 로직안에 있는 단어 쪼개기 로직의 성능 이슈**입니다. 되게 간단하게 **Substring을 활용해서 이중 for문으로 구현을 했었는데 이 부분에 병목**이 있었습니다. 그래서 위 첫 번째 사진보시면 아실 수 있다시피 **Spring이 모두 온전히 뜰 때까지 총 158초가 걸린 것**입니다. 이는 물론 저희 **Production DB에 총 가게가 약 51200개 정도 있고 가게 이름이 평균 6글자라고만 하더라도 총 51200*6 = 307200번 정도의 Redis 연산이 순차적으로 실행**되었을 겁니다.   

사실 Backend Application이 뜰 때 딱 1번만 실행되는 로직이라 실제 운영 환경에서 당장은 문제가 안될 것 같기도 하지만 적어도 Dev나 Local 환경에서 계속 빌드할 때마다 저 시간을 기다리기에는 너무 번거롭고 고통스러워서 이 부분을 반드시 개선해야겠다는 생각이 들었습니다. 

그래서 저는 결과적으로 **병렬 프로그래밍을 통해 이 부분을 158초에서 0.009초로 성능을 대폭 개선시켰습니다. 두 번째 사진 속 elapsed1이 개선 전 로직 소요 시간이고 elapsed2가 개선 후 로직 소요 시간입니다. 이와 관련된 자세한 내용은 다음 포스팅에서 병렬 프로그래밍을 함께 소개하며 공유**드리겠습니다. 