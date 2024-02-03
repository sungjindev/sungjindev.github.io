---
title: 지도 서비스에서 두 대각 좌표를 이용한 MBR 영역을 사용하면 안되는 이유!
categories: [Computer engineering, Backend engineering]
tags: [backend, spring, java, MBR, Minimum Bounding Rectangle, Map, 백엔드, 스프링, 자바, 최소 크기 사각형, 지도]
---

이번 포스팅에서는 두 (경도, 위도) 좌표로 만들어지는 MBR(Minimum Bounding Rectangle) 최소 크기 사각형 영역 내 존재하는 데이터들을 조회하다가 발생할 수 있는 이슈를 소개하려 합니다.

## MBR(Minimum Bounding Rectangle)이란?
MBR이란 MySQL 데이터베이스를 활용해 지도 서비스를 개발해봤다면 한번쯤 들어봤거나 사용해봤을 개념입니다. Minimum Bounding Rectangle로 주로 데이터베이스에서 공간 데이터를 조회할 때 영역을 지정하기 위해 사용합니다.   
두 개 이상의 좌표들을 받고 해당 좌표들을 모두 포함할 수 있는 가장 작은 크기의 사각형을 의미합니다.    
     
최근 프로젝트에서 저는 MBR을 활용해서 클라이언트가 요구하는 영역 내에 존재하는 가게 데이터들을 모두 리턴해주는 API를 개발하고 있었습니다. 그러던 중 아래와 같은 문제에 맞닥뜨리게 됩니다.

## 문제의 발단
문제 상황 자체를 한 줄에 모두 표현하기가 어려워서 포스팅의 제목을 결정하는 것부터가 쉽지 않았는데, 우선 무슨 이슈가 어떻게 발생하게 된 것인지 풀어서 설명드리겠습니다.   
    
다들 지도와 데이터들의 위치를 그 지도 위에 마커로 표시하는 기능을 구현하시다보면 MBR(Minimum Bounding Rectangle)이라는 것을 사용해서 많이들 필터링하실 것이라 생각이 듭니다.
저 또한 마찬가지로 처음에는 MySQL의 MBR 기능을 이용해서 필터링할 영역을 정해주고, 그걸 Query의 Where 절에 조건문으로 둬서 제가 원하는 영역 내 속하는 데이터들을 조회하여 사용했었습니다.   
    
이 과정에서 저는 Mobile Applicaiton Client로부터 좌상단 (경도, 위도) 좌표와 우하단 (경도, 위도) 좌표를 전달받고 이 두 좌표를 사용해서 만들어지는 MBR 영역 내 가게들을 리턴해주는 기능을 구현해야 했으며, Client에서 넘겨주는 좌상단, 우하단 두 좌표는 사용자가 보고 있는 지도 서비스 스크린의 대각선 끝 좌표입니다.    
     
![1](/assets/img/mbr-issues-with-two-coordinates/1.png){: w="300" h="80" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}
     
그렇게 만들어지는 MBR 사각형을 표현해보면 위와 같습니다. 이렇게 보면 별 문제가 없을 것 같은데, 문제는 사용자가 지도를 회전하게 되면 발생합니다.   
     
![2](/assets/img/mbr-issues-with-two-coordinates/2.png){: w="300" h="120" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}
     
만약 사용자가 위처럼 지도 자체를 위측으로 약 45도 정도 회전시켜서 봤다고 생각해봅시다. 그러면 위 사진처럼 경도, 위도 축도 함께 돌아가게 됩니다.   
이게 언제 문제가 되냐면 바로 아래 사진 처럼 회전된 지도에서 MBR 영역을 정하기 위해 좌상단, 우하단 두 좌표를 보냈을 때 발생합니다.

![3](/assets/img/mbr-issues-with-two-coordinates/3.png){: w="300" h="80" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}
     
이렇게 지도를 회전시키게 되면 그 방향에 맞춰서 절대적인 경도, 위도축도 함께 돌아가버리기 때문에 MBR을 구할 때 회전된 축을따라 똑같이 회전된 영역이 구해지게 됩니다. 그렇게 구해진 영역이 바로 위처럼 회전되어 구해진 영역인 것입니다.   
    
즉, 개발자는 당연히 사용자가 보고 있는 스크린의 방향과 일치하게 영역에 데이터들을 띄워줄 것이라 생각했지만 만약 사용자가 회전 시킨채로 영역을 구하기 위한 두 좌표를 보내주게 되면 위 두 번째 사진처럼 사용자 스크린 전체에 가득 차는 영역이 아닌 회전되어 일부 영역은 포함되지 않은 채로 데이터들이 조회되게 됩니다.   
     
```java
Query query = em.createNativeQuery("SELECT sc.* " + "FROM store_certification AS sc " + "JOIN store AS s ON sc.store_id = s.store_id " + "JOIN certification AS c ON sc.certification_id = c.certification_id " + "WHERE MBRCONTAINS(ST_LINESTRINGFROMTEXT(" + pointFormat + "), s.location)", StoreCertification.class);
```     
     
참고로 이때 제가 사용했던 쿼리는 위와 같습니다.

## 해결
처음에 이 문제를 인지하거나 이 문제가 발생하는 원인에 대해서 파악하는 것이 오래걸리지 해결하는 법을 떠올리는 것은 그리 어렵지 않을 것입니다.   
    
지금 이런 문제가 생기는 이유는 MBR을 단 두 좌표값만을 사용해서 만들었기 때문입니다. 두 좌표를 이용해서 사각형을 만든다는 것은 적은 정보로 더 고차원의 정보를 얻어내는 과정이며, 그러다보니 제가 생략한 나머지 두 좌표에 대해서는 MBR의 기준에 따라 자동으로 계산되어 채워지기 때문에 위처럼 제가 의도하지 않은대로 동작하게 되는 것입니다.   
    
좀 더 자세히 적어보면, 단 두 좌표로 MBR을 구할 때 로직은 두 좌표의 최소, 최대 경도, 위도값을 각각 구하게 되고 이를 통해 영역의 네 꼭짓점을 구하는 방식이기 때문에 기준이 되는 경도, 위도 두 축이 회전하게 되면 당연히 그 좌표계 위에서 구해지는 영역도 함께 회전되어 계산될 수 밖에 없을 것입니다.   
    
따라서 이 문제를 해결하기 위해서는 네 좌표를 사용해야합니다. 이 과정에서 MySQL의 POLYGON이라는 데이터 타입을 사용하였습니다. 다음 쿼리가 이 문제를 해결한 쿼리입니다.   
    
```java
Query query = em.createNativeQuery("SELECT sc.* " + "FROM store_certification AS sc " + "JOIN store AS s ON sc.store_id = s.store_id " + "JOIN certification AS c ON sc.certification_id = c.certification_id " + "WHERE ST_CONTAINS(ST_POLYGONFROMTEXT(" + pointFormat + "), s.location)", StoreCertification.class);
```    
     
이전에 작성했던 쿼리와 현재 문제를 해결한 쿼리를 비교해보면 크게 달라진 부분이 없는 것 같지만 기존에는 MBRCONTAINS를 사용했던 부분이 ST_CONTAINS로 바뀌었고 기존에는 두 좌표만을 사용했기에 ST_LINE을 사용했지만 이번에는 네 좌표를 사용했기 때문에 ST_POLYGON을 사용했습니다. 이렇게 하면 사용자가 지도를 회전시킨 채로 영역을 조회하더라도 사용자가 보낸 네 꼭짓점을 통해 올바른 영역 사각형이 바로 특정되기 때문에 문제를 해결할 수 있습니다.

## 마무리
이번 포스팅에서는 지도 서비스를 개발할 때 영역을 통해 데이터를 조회하면서 경도, 위도 축이 회전 상태로 좌표가 넘어오게 되면 어떤 문제가 발생할 수 있는지 알아보고 해결 방법까지 기록해봤습니다.    
    
어떻게 보면 굉장히 사소한 문제이고 당연한 문제이기도하나 무심코 넘어가기 쉬운 오류라 이렇게 기록으로 남겨 공유드립니다. 이와 같은 문제로 어려움을 겪으시는 분이 계시다면 작게나마 도움이 되었으면 좋겠습니다. 감사합니다.