---
title: '특정 DOM이 나타난 이후에 작업하기'
date: 2022-11-15 22:12:01
category: atWork
# thumbnail: { thumbnailSrc }
draft: false
---

오늘은 저희가 유통하고 있는 F5 솔루션을 사용하고있는 고객사에서 한번 로그인 후에는 방문 시 자동으로 로그인이 되도록 해달라는 요청을 받았습니다.

여러 보안상의 문제가 우려되지만 우선 가능한 방법이 있다면 적용해주는 것으로 내부 결정이 되었습니다.

F5 솔루션은 웹커스터마이징이라는 이름으로 자신들의 WebUI에 사용자 정의 스크립트를 붙일 수 있게 해줍니다. 이 웹커스터마이징을 이용해서 자바스크립트로 요구사항을 구현해야합니다.

우선, 엔지니어께서 F5 서버의 설정을 통해 이전에 입력하였던 사용자 ID/PW를 쿠키로 내려줄 수 있다고 합니다. 때문에 굳이 처음에 ID/PW를 자바스크립트로 가져와서 저장할 필요는 없고, 쿠키에서 읽어와서 입력 및 폼 제출만 자바스크립트로 해주면 됩니다.

로그인은 전형적인 `<form>` 제출로 이뤄지기 때문에, 절차는 간단하게 아래와 같습니다.

1. `cookie`에서 이전에 입력한 ID/PW 가져오기
2. `username`과 `password` `<input>`의 `value`에 넣기
3. `form.submit()`하기

문제는 `<input>`과 `<form>` 태그를 가져오려 했지만 `null`값이 반환되었다는 것입니다.

분명 개발자도구를 통해 봤을 때는 해당 요소들이 있었는데, 다시 `source`보기를 통해 확인해보니 F5 웹은 자체적인 SPA 프레임워크를 이용해 페이지를 렌더링하고 있었습니다. 즉 `React`나 `Vue`처럼 처음에는 빈 HTML에서 시작해서 자바스크립트로 각종 input과 form을 그리고 있었던 것입니다.

그리고 제가 삽입한 script는 렌더링이 되기전에 실행되어버려서 해당 태그를 찾을 수 없었던 것입니다.

```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=Edge">
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <meta name="robots" content="noindex,nofollow" />
    <link rel="stylesheet" href="/public/include/css/modern/framework.css?q=1649477949" />
    <script type="text/javascript" src="/public/include/js/modern/loader.js?q=1649477949"></script>
    <script type="text/javascript" src="/public/include/js/modern/main.js?q=1649477949"></script>
    <script type="text/javascript">
    function init(apmui) {
        var appLoader = new apmui.AppLoader();
        appLoader.configure({
    "pageType": "logon",
    "source": "/Common/modern",
    "styles": [],
    "scripts": [
        "/public/include/js/modern/user-logon.js?cg_code=22ed887969b58abee4cd97df8feddd5b&cg_name=n8h5bofZ1W9V6H1P4a0Auk_iaxJPaAJMZycBJq0xRtvVzCNQbEk-4zhhmrtx3xde"
    ],
    "logon": {
        "softToken": {
            "fieldName": "",
            "state": "",
            "newPin": ""
        },
        "form": {
            "id": "auth_form",
            "title": "F5 Networks \ubcf4\uc548 \ub85c\uadf8\uc628",
            "submitCaption": "\ub85c\uadf8\uc628",
            "savePassword": "\uc554\ud638 \uc800\uc7a5",
            "passwordVerifyDontMatch": "\uc785\ub825\ud558\uc2e0 \uc554\ud638\uac00 \uc77c\uce58\ud558\uc9c0 \uc54a\uc2b5\ub2c8\ub2e4.",
            "fields": [
                {
                    "type": "text",
                    "name": "username",
                    "caption": "\uc0ac\uc6a9\uc790 \uc774\ub984",
                    "value": "",
                    "disabled": false
                },
                {
                    "type": "password",
                    "name": "password",
                    "caption": "\uc554\ud638",
                    "value": "",
                    "disabled": false
                }
            ]
        },
    // ...생략
    </script>
</head>
<body onload="javascript: __run();">
</body>
</html>
```

> 위처럼 `body`가 `onload`되었을 때 자바스크립트 함수를 실행하면서 렌더링이 되고 있습니다.

그래서 처음에는 `DOMContentLoaded`라는 이벤트에 리스너를 달아서 시도해보려고 했으나, 해당 이벤트는 그냥 빈 `body`태그가 생겼을 때 발생하는 것으로 확인되었습니다. 또한 제가 삽입한 스크립트 실행시점에는 호출되지 않는 것으로 보아, 해당 이벤트가 이미 발생한 후에 스크립트가 실행되는 것 같았습니다.

이와 같이 SPA로 된 웹페이지에서 특정 DOM이 렌더링 된 이후에 작업을 하고 싶다면 어떻게 해야할까요?

이럴 때는, `MutationObserver`를 사용하면 됩니다.

- [MutationObserver MDN 설명](https://developer.mozilla.org/ko/docs/Web/API/MutationObserver)
- [MutationObserver CanIUse](https://caniuse.com/?search=MutationObserver)

`MutationObserver`는 초기 생성자에서 mutation이 일어났을 때 무엇을 할지에 대한 함수를 받습니다. 그리고 `.observe()` 메소드를 통해 어떤 노드를 감시할 지 설정해줍니다.

이를 우리의 목적대로 특정 DOM이 있을 때 사용하기 위해 사용하려면 아래와 같이 코드를 짜면 됩니다.

```javascript
// Wait for HTMLElement to appear
function waitForElm(selector) {
    return new Promise(resolve => {
        if (document.querySelector(selector)) {
            return resolve(document.querySelector(selector));
        }

        const observer = new MutationObserver(mutations => {
            if (document.querySelector(selector)) {
                resolve(document.querySelector(selector));
                observer.disconnect();
            }
        });

        observer.observe(document.body, {
            childList: true,
            subtree: true
        });
    });
}
```

`waitForElm`이라는 함수는 Promise를 반환하는 비동기 함수로 내부에 `MutationObserver`를 생성하고 활용하고 있습니다. `document.body`를 감시하면서 변화가 생겼을 때 특정 `selector` 요소가 있다면 `resolve`를 반환합니다.

그럼 이제 이 함수를 이용해서 우리가 원하는 작업을 아래와 같이 하면 됩니다.

```javascript
// get auth from cookie and submit automatically, after form appear
waitForElm('#auth_form').then((form) => {
    console.log("form ready");

    var usernameCookie = getCookie("cookiename");
    if (usernameCookie === "") {
        console.log("No cookie found");
        return;
    }
    var pwCookie = getCookie("cookiepassword");

    var usernameInput = document.getElementById("username");
    var pwInput = document.getElementById("password");

    usernameInput.value = usernameCookie;
    pwInput.value = pwCookie;

    form.submit();
});
```

이렇게 코드를 짜면 `#auth_form`이라는 `id`의 `<form>` 태그가 화면에 존재할 경우에 `.then`이 실행됩니다. 여기서 우리가 원래하려고 했던, 쿠키로부터 ID/PW를 가져와서 input에 넣고 `form.submit()`을 제출하면 됩니다.

### 결론

오늘은 SPA 프레임워크 등을 사용해서 DOM이 늦게 렌더링 될 경우 특정 DOM이 나타난 이후에 작업을 하고 싶다면 어떻게 해야할지에 대해 알아보았습니다. `MutationObserver`라는 내장 함수가 있는지 처음 알게 되었고 간단하게 사용법도 알 수 있었습니다.

이를 이용하면 제가 작성하지 않은 SPA 웹사이트에서 DOM 작업을 하기 수월해집니다.
