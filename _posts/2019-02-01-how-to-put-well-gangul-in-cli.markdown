---
layout: post
title:  "Shell환경에서 한글을 줄맞춰 찍어보자"
date:   2019-02-01
description: "쉬움 주의"
---

> 구현한 메일 서비스를 CLI로 가즈아~


### 고민
#### 메일 리스트를 shell 환경에서 이쁘게 찍으려면
* shell 환경에서 한글은 2칸 나머지는 1칸 - OK!
* 표 형태로 보여주고 싶은데
* Java의 String.format()에서는 한글이든 뭐든 다 동일하게 인덱스 하나씩
  * java에서 format 형식 그대로 넘겨주면 줄이 안 맞아 삐뚤빼뚤 표를 이룰 수가 없음

### 생각
* 문자열에서 한글만 따로 카운드 할 수 있다면..
  * 정규표현식으로 가능한가?
* 뭔가 계산이 복잡할것 같은데..

### 해결
* 한글 갯수 세기
  * Character.getType() 사용
    * 문자 하나 넣으면 해당 문자의 종류에 맞는 int 값 return
    * 한글은 5
* java에서 해당 문자열에 format할 문자개수 = 
  * shell 상에서 문자열이 차지할 칸 수 - 문자열에 포함된 한글의 개수


#### 코드
```java
String string;
int stringSpaceInShell = 40;
int stringLengthInShell = 0;
int numHangul = 0;

for (int i = 0; i < string.length(); i++) {
    if (stringLengthInShell > stringSpaceInShell-1){
        //공간 넘어가면 문자열 자름
        string = string.substring(0,i);
        break;
    }
    //한글일때
    if(Character.getType(mailTitle.charAt(i)) == 5) {
        stringLengthInShell += 2;
        numHangul++;
    }
    else 
        stringLengthInShell++; 
}
// 문자열이 차지할 칸
String = formatType = "| %-" + (stringSpaceInShell-numHangul) + "s ";
```

#### 실제 적용 예
![](https://raw.githubusercontent.com/tanker0212/tanker0212.github.io/master/assets/img/shellprintlist.PNG)

### 남은 생각
* 문자열을 하나씩 for 문으로 돌려보는 부분 비효율적으로 보임
* 칸 수 계산 부분 더 똑똑하게 할 수 있을 것 같음
