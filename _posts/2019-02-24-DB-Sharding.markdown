---
layout: post
title:  "DB Sharding"
date:   2019-02-24
description: "이게 머조..?"
---

### DB 파티셔닝?

* 데이터를 분할하는 기술 - 퍼포먼스, 유용성, 관리성 향상을 위해
* 샤딩이 파티셔닝의 기술 중 하나
* 크게 vertical, horizotal 파티셔닝으로 나뉨
  * vertical
    * 데이터를 수직적으로 나누는 방식
    * 테이블의 칼럼을 분할해 여러 테이블로 나눈다
    * 데이터 보안상의 이유로 하는 경우도 있음
  * horizotal
    * 데이터를 수평적으로 나누는 방식
    * 동일한 테이블을 여러 곳에 두고 특정 기준으로 데이터를 분리하여 저장하는 방식
    * 이게 바로 `샤딩`


> Shard -> 조각

### 그래서 Sharding 이란?

* 관계형 데이터베이스 확장의 방법 중 수평 파티셔닝
* 샤딩의 다양한 기법이 있겠지만 대표적인 두 가지 Ranged Sharding과 Hashed Sharding

#### Ranged Sharding
* 이름에 느껴지는 것처럼 범위를 지정하고 범위에 해당하는 데이터를 그 범위의 DB에 저장하는 방식

![](https://docs.mongodb.com/manual/_images/sharded-cluster-ranged-distribution-good.bakedsvg.svg)

* 데이터 범위의 기준 - PK, 시간(연도), 사용자, 지역 등등
* 기존에 운영하던 서비스에서 DB의 수용량이 한계에 달했을 때 쉽게 샤드를 추가할 수 있음
* 지금까지 쌓인 데이터와 앞으로 쌓일 데이터 간의 연관관계가 적으면 적을 수록 효과적

BUT
* Work Load의 분산이 보장되지 않음
* 샤딩후 쌓이는 데이터와 이전 데이터 간 Join이 발생하면 아주 곤란할 수도 있다구..


#### Hashed Sharding
* 해시 함수를 통해서 데이터가 저장될 서버를 지정하는 방법

![](https://docs.mongodb.com/manual/_images/sharded-cluster-hashed-distribution.bakedsvg.svg)

* 해시 함수를 잘 골라야 함
* 데이터 편향이 적게 균등 분포가 쉽다


BUT
* 샤딩이 완료된 후 추가적인 샤드를 늘리는 게 어려움
  * 해시값의 범위가 달라져 대규모의 데이터 이동이나 해시함수의 아주 적절한 변경을 해야 하는데 어려움
  
* * *

* 그래서 서비스를 시작하기 전 샤딩을 염두에 두고 설계를 할 것 같음이


### DB 샤딩이 무조건 답은 아니다
* 위에서 말한 샤딩의 문제로 이미 샤딩을 진행한 db에 대해서는 성능이나 용량의 한계에서 scale-up을 고려하는 것이 일반적
* 대규모 데이터 이동이나 데이터를 재배치하는 기준을 만드는데 비용이 많이 들기 때문에 
* 샤딩된 db들 간의 join 문제
  * 보통 불가능하다고 보는가보다 
  * 이 문제에 대해서 완전한 해결책이 없음 
  * join이 필요한 데이터에 대해서는 중복적으로 저장하는 것이 일반적
  * 이 문제에 대해서 사전에 생각했을 때는 작은 단위의 쿼리를 여러 번 해서 해결 가능할 것이라고 생각했는데 작은 규모의 서비스에서 충분히 가능하겠지만 서비스의 몸집이 커지면 이 또한 많은 비용이 드는 방법인가 봄

* * *

### 그래서 구현은?
#### 메일 서비스
* 기존 1개의 DB에서 2개의 DB로 Hashed Sharding 진행
  * 데이터를 나누는 기준은 사용자!
  * 사용자를 기준으로 메일과 첨부파일에 해당하는 모든 데이터를 분할
    * A라는 사용자가 DB1에 샤딩되었다면 A사용자의 모든 데이터 읽는 활동은 DB1에서만 진행됨

* 샤딩을 적용하면서 가장 중요하게 생각한 부분 -> 원래의 구조를 바꾸지 말자 
  * 메일 보내기의 프로세스 - 여러사람에게 보낼때
    1. 메일을 받는 사람들을 보고 각 메일 데이터가 저장될 DB 서버를 구함
    2. 각각의 메일을 해당하는 DB에 저장 
    3. 저장하면서 각 메일 데이터의 PK를 받아옴
    4. 모든 메일 데이터의 저장이 끝나면 나온 메일의 PK들을 보고 첨부파일 데이터를 각각의 메일 수 만큼 저장 (중복저장 스펙입니다.)

    * 이 프로세스상에서 첨부파일 데이터를 저장할때 사용자의 정보가 없음으로 바로 해당하는 DB서버를 찾을 수 없는 문제가 발생
    * DB쿼리를 한 번 더해 타겟 DB를 찾는 것은 너무 비효율적이라 생각
  
  * 메일내용과 첨부파일 각각 테이블의 PK는 Auto increment 숫자 
  * 이 PK를 적절히 나눠서 PK만 보고도 타겟 DB를 알아낼 수 있으면 좋겠다
  * PK를 DB의 수에 맞게 나눔
  * 3대의 DB서버가 있을때 : 
    * DB1 : 1, 4, 7,...
    * DB2 : 2, 5, 8,...
    * DB3 : 3, 6, 9,...
  * 이 방법으로 각 데이터의 PK만으로 타겟 DB를 찾을 수 있음
  * 구현도 각 DB의 Auto Increment 설정의 숫자 2개를 바꾸는 것으로 쉽게 끝남

