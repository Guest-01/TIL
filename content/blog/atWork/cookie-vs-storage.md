---
title: 'access token은 어디에 저장하는 것이 좋을까'
date: 2022-12-21 22:12:01
category: atWork
# thumbnail: { thumbnailSrc }
draft: true
---

우리 회사는 자체 솔루션을 개발하여 판매하고 있기도 하지만, 해외 유수 보안 솔루션의 벤더사로써도 영업하고 있습니다.

최근에는 우리가 벤더사로써 판매하고 있는 P사의 솔루션이 제공하는 대시보드를 다시 만들어달라는 요구사항이 있었습니다.

P사의 대시보드 화면은 실제 관리자가 보고 싶어하는 데이터 외에 너무 많은 데이터가 존재하고, 관리자라 하더라도 보아서는 안되는 민감한 개인정보들도 함께 있었기 때문입니다. 그래서 P사의 솔루션에서 필요한 정보들만 추려서 새로운 대시보드를 만들어달라는 미션이 들어왔습니다.

그래서 열심히 대시보드를 만들던 중, 로그인 정보를 어디에 담아야하는지 몰라 고민한 내용을 여기 공유합니다.

## 레퍼런스 체크

먼저 기존 P사의 대시보드가 어떻게 구성되어있는지 확인해보았습니다.

기존 P사 대시보드는 `ASP`로 제작되어 있는 전통적인 서버사이드 웹페이지였으며, 때문에 서버에서 `set-cookie`를 이용해 브라우저에 로그인 정보를 저장하고 있었습니다. 세션쿠키를 사용하는 방식은 예전에는 표준으로 여겨지던 방식이며, 사용자에 대한 민감정보를 서버에 저장해놓고 브라우저에는 세션 키값만 쿠키로 저장하기 때문에 보안상 안전하나 서버 자원을 소모하며 "스케일 아웃"을 하기 어렵다는 단점이 있습니다.

세션쿠키로 동작하는 로그인 방식의 특징으로는, 새탭이나 새창을 열어도 로그인이 유지된다는 점이 있습니다. 쿠키는 사이트마다 저장되고 브라우저 전체로 공유가 되기 때문입니다.

처음에는 이런 특징을 파악하지 못하고 `sessionStorage`를 사용하여서 아래와 같이 사용자경험이 달라지는 문제가 있었습니다.

## Session Storage를 사용한 첫번째 버전

처음에 세션 스토리지를 사용한 것은, 막연히 세션 스토리지가 로그인에 쓰기 좋다는 이야기를 어렴풋이 들었기 때문입니다. 세션 스토리지는 세션(개별 브라우저 탭 또는 창)이 종료되면 자동으로 사라지는 특징을 가지고 있습니다. 따라서 브라우저를 닫을 시 로그인 정보가 날아가야하는 경우에 쓰기 좋습니다. 하지만 제가 간과했던 것은, 이 세션의 기준이 개별 탭 또는 창이라는 것이었습니다.

기존의 대시보드에서는 쿠키를 사용했기 때문에 별도의 탭이나 창을 띄워도 세션이 동일하게 공유되었습니다. 특히나 대시보드와 같이 여러 정보를 볼 수 있는 페이지는 여러 메뉴를 여러 탭이나 창에 띄워놓고 동시에 보는 경우가 많은데, 세션 스토리지를 사용하게 되면 탭 또는 창마다 로그인을 해주어야하는 문제가 생기죠.

이렇게 되면 로그아웃 할때도 각각의 탭이나 창마다 로그아웃을 해줘야한다는 문제가 생깁니다. 물론 세션 스토리지가 그렇게 쓰라고 만든 것이므로 그게 정상이라고 우길 수도 있겠지만, 기존의 쿠키 방식과는 사용자경험이 달라지기 때문에 다른 방법을 찾아야했습니다.

## Local Storage를 사용한 두번째 버전

위의 문제를 해결하기 위해서 로컬 스토리지로 저장소를 옮겨보았습니다. 로컬 스토리지는 세션 스토리지와는 달리, 탭이나 창 간에 공유가 되는 저장소입니다. 로컬 스토리지에 액세스 토큰을 넣어놓으면, 사용자가 한번 로그인을 하고 새 탭이나 창에서 대시보드를 켜도 동일한 계정으로 로그인이 되어있습니다.

이렇게하여 모든게 해결된 것처럼 보였으나, 한가지 아쉬운 점은 로컬 스토리지는 브라우저를 종료하더라도 남아있는다는 점이었습니다. 기존의 쿠키 방식은 창을 닫으면 삭제되어 (expire가 없는 세션 쿠키의 경우) 브라우저를 켜면 다시 로그인을 해야했으나 로컬 스토리지는 자동으로 삭제되지 않아 로그인이 유지되는 차이가 있었습니다.

물론 서버로부터 받아온 토큰이 만료된 상태라면 로그인 화면이 다시 나오기 때문에 큰 상관은 없지만 역시 기존과는 약간 다른 사용자 경험이므로 찝찝한 점은 지울 수 없었습니다.

그러나 저는 일단 이 방식으로 하기로 마음을 먹었습니다. 쿠키는 아무래도 서버쪽에서 클라이언트에 데이터를 저장하고 싶을 때 사용하는 곳이라는 느낌이 강했고 프론트엔드에서 쿠키를 파싱하는 작업을 하고 싶지 않았기 때문입니다. 그리고 애초에 쿠키를 파싱하고 관리하는 API도 빈약해서 자바스크립트단에서 사용하지 말라는 의미처럼 느껴졌습니다. 예를 들어, 로컬/세션 스토리지 API는 `localStorage.getItem()`과 같이 키값을 기준으로 값을 꺼내거나 넣는 API가 직관적으로 제공되는데 비해, 쿠키는 `document.cookie`라는 값을 이용해서 문자열로 직접 `key=value`를 집어넣어야하는 식입니다.

웹 API에서 굳이 쿠키 API를 이렇게 불편하게 만들어놓는 이유가 아마 서버쪽에서 관리하라는 의미가 아닐까 싶습니다.

## 결론

결론은 프론트엔드에서 자바스크립트로 무언가를 저장하고 싶다면 로컬/세션 스토리지를 사용하는 것이 좋습니다. 쿠키는 서버에서 사용하는 곳으로 생각하는게 맞는 것 같습니다.