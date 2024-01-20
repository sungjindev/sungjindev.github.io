---
title: Spring Project에서 API Response time을 21초에서 0.3초로 줄여본 이야기
categories: [Computer engineering, Backend engineering]
tags: [backend, spring, java, refactoring, performance issue, 백엔드, 스프링, 자바, 성능 개선, 성능 이슈]
---

이번 포스팅에서는 Spring으로 Project를 진행하다가 마주친 API 성능 이슈를 소개하고 어떻게 20초 걸리던 API call을 0.5초로 줄였는지 공유해보려고 합니다.

## 문제의 발단
![1](/assets/img/reduce-api-response-time/1.png){: w="1000" h="600" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}
저는 현재 네이버 지도나 카카오맵과 같이 여러 전국 가게들에 대한 정보를 지도에 띄워주는 Application에서 Spring 기반의 Backend project를 진행하고 있습니다.   
    
그 과정에서, 특정 좌표 영역 내 위치하고 있는 가게들에 대한 상세 정보를 리턴해주는 API가 필요했는데 이 API를 전부 구현하고 테스트를 진행해보니 위 사진처럼 약 21초의 응답 시간이 걸리는 문제가 있었습니다. 당연히 이 응답 시간으로는 절대 실사용이 불가능했고 빠르게 1차 출시 전 Refactoring이 필요했습니다.   
    
## 원인 분석
문제 상황은 충분히 인지했으니, 저는 원인 분석에 곧장 나섰습니다. 해당 API 로직이 실행시키는 메서드 구석 구석에 들어가 실행 시간을 측정하였습니다.   
   
이때 메서드 처리 시간은 AOP를 활용해도 되지만 로직 자체가 그렇게 복잡하지 않고 당장 급히 수정이 필요했기 때문에 아래와 같은 로직으로 간단하게 시간 측정을 진행하였습니다.
```java
long startTime = System.currentTimeMillis();
long endTime = System.currentTimeMillis();
long elapsedTime = endTime - startTime;
System.out.println("elapsedTime = " + elapsedTime);
```

우선 Repository단부터 확인하여 쿼리 속도를 측정해봤는데 0.2초 정도로 굉장히 양호하게 나왔습니다. 그 후 Controller단, Service단을 측정하였는데 문제가 되는 로직을 찾았습니다.
```java
List<Long> duplicatedIds = allStoreIds.stream()
        .filter(e -> allStoreIds.indexOf(e) != allStoreIds.lastIndexOf(e))  //중복된 StoreId가 있는 경우
        .distinct() //해당 id를 모아서 1번씩만(중복 제거) 리스트에 담아 전달
        .collect(Collectors.toList());
```

![2](/assets/img/reduce-api-response-time/2.png){: w="1000" h="600" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

바로 위 로직에서만 약 20초의 병목이 걸리는 것을 확인하였습니다. 위 로직은 특정 테이블에서 중복으로 들어가있는 가게 데이터의 id값을 찾기 위한 로직이었는데 기능 구현에만 초점을 두고 급히 짠 로직이다보니 짜면서도 긴가민가했지만 이게 이렇게 오랜 시간이 걸리게 될 줄은 몰랐습니다.   
    
로직을 보면 알 수 있다시피 많은 데이터가 들어가있는 걸 stream을 통해 n회 돌게되는데 이때 또 그 반복문 안에서 중복된 id인지 검사를 하기 위해 n번을 돕니다. 이것만 봐도 굉장히 비효율적이라는 것을 알 수 있었습니다.

## 해결
이제 원인도 찾았으니, 해결만 하면 됩니다. 생각보다 다행스럽게도 간단한 문제였습니다. 무심코 가볍게 짜고 지나쳤던 로직이 너무 비효율적이었고 해당 문제는 중복 데이터를 추출해내는 로직이었는데 이를 모두 완전 탐색으로 풀어서 오랜 시간이 걸렸던 겁니다.   
    
그래서 저는 조회 시간을 줄여야 겠다는 생각에 Hash를 적용시키기로 합니다.
```java
HashSet<Long> uniqueStoreIds = new HashSet<>(); //조회 성능을 높이기 위해 HashSet으로 저장
HashSet<Long> duplicatedStoreIds = new HashSet<>();

for (StoreCertification storeCertification : allStoreCertifications) {
    Long storeId = storeCertification.getStore().getId();
    if (!uniqueStoreIds.add(storeId)) { //HashSet에 add를 했을 때 이미 존재하는 데이터면 false가 리턴되는 것을 활용
        duplicatedStoreIds.add(storeId);
    }
}
```

![3](/assets/img/reduce-api-response-time/3.png){: w="1000" h="600" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

위 코드처럼 구현을 하였고 Hash는 조회에 O(1)의 시간밖에 걸리지 않으므로 전체 데이터를 조회하는 가장 바깥 for loop n회만 돌아주면 됩니다.   
이렇게 구현하고 나서 다시 시간을 확인해보니 아래와 같이 약 0.4초가 걸리게 되었습니다.   
    
이로써 시간을 크게 단축할 수 있었는데 뭔가 더 비효율적인 로직이 있진 않을까 다시 한번 고민하다가 생각해보니 이 중복된 store id를 검사하는 로직을 api call마다 호출시킬 필요가 전혀 없다는 생각이 들었습니다. 왜냐하면, 서비스의 특성상 store data가 업데이트 되지 않기 때문에 Spring이 처음 뜬 상태 그대로 데이터가 유지되기 때문입니다. 그래서 이 반복적으로 호출되는 로직을 한번 더 Refactoring 하기로 결정합니다.

## 한번 더 Refactoring
해당 API call마다 반복적으로 중복된 store id를 찾아내던 로직을 해당 API가 사용하는 Service가 처음 생성되었을 때 1번만 중복된 store id들을 조회하고 그걸 Globally하게 사용할 수 있도록 수정해야겠다는 생각이 들었습니다. 그래서 Service 단에 아래와 같은 코드를 추가하였습니다.    

```java
@PostConstruct
public void init() {    //이 Service Bean이 생성된 이후에 한번만 중복된 storeId를 검사해서 Globally하게 저장
    List<StoreCertification> allStoreCertifications = storeCertificationRepository.findAll();   //중복된 id를 검사하기 위함

    HashSet<Long> uniqueStoreIds = new HashSet<>(); //조회 성능을 높이기 위해 HashSet으로 저장
    HashSet<Long> duplicatedIds = new HashSet<>();

    for (StoreCertification storeCertification : allStoreCertifications) {
        Long storeId = storeCertification.getStore().getId();
        if (!uniqueStoreIds.add(storeId)) { //HashSet에 add를 했을 때 이미 존재하는 데이터면 false가 리턴되는 것을 활용
            duplicatedIds.add(storeId);
        }
    }
    duplicatedStoreIds = new ArrayList<>(duplicatedIds);
}

public List<Long> getDuplicatedStoreIds() {
    return duplicatedStoreIds;
}
``` 

위처럼 Service단에 @PostConstruct 애너테이션을 활용해서 해당 Service Bean이 처음 생성된 직후에 한번만 중복된 storeId를 검사해서 필드로 담을 수 있게 구현하였습니다. 그 후 해당 리스트가 필요할 때마다 getter를 통해 조회할 수 있도록 하였습니다.   

![4](/assets/img/reduce-api-response-time/4.png){: w="1000" h="600" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

![5](/assets/img/reduce-api-response-time/5.png){: w="1000" h="600" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

이렇게 진행하였더니, 그 전에는 위 사진처럼 1294ms가 걸렸던 API call response time이 300ms로 더욱 줄게되었습니다.

## 마무리
이번 포스팅에서는 크게 총 두 번의 Refactoring 과정을 거쳐 약 21초가 걸리던 API response time을 0.3초로 줄여본 경험에 대해서 공유드렸습니다.   
    
사실 지금 정리하면서 돌이켜보면 처음 로직 자체가 너무 급히 짠 나머지 굉장히 좋지 못한 코드였고 시간 복잡도에 대한 고려가 전혀 되어있지 않았던 것 같습니다. 이렇게 글로 공유드리기조차 굉장히 부끄러운 로직이지만 급히 쳐내야하는 기능이라 할지라도 이렇게 큰 성능 이슈로 나중에 되돌아올 수 있으니 항상 성능에 대해 깊이있게 고민하고 저와 같은 또다른 피해자가 발생하지 않았음해서 정리해봤습니다. 감사합니다. 