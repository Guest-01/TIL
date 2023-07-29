---
title: '프론트엔드에서 파일 입출력 다루기'
date: 2023-07-25 22:12:01
category: atWork
# thumbnail: { thumbnailSrc }
draft: false
---

회사 솔루션에서 백오피스 특성상, CSV 가져오기/내보내기를 구현할 일이 있었습니다. 사용자가 본인이 가진 기기의 MAC주소를 등록하는 화면이 있었는데 이때 UI에서는 한개씩만 등록할 수 있게 되어있었기 때문이죠.

UI상에서 여러개를 등록할 수 있게 개선할 수도 있지만, 그보다는 엑셀형태(정확히는 CSV)로 관리하는 방식을 관리자들이 선호했기 때문에 대신 CSV파일을 업로드하여 여러개의 MAC 주소를 한번에 입력할 수 있게 해달라고 했습니다.

이를 구현하기 위해서는 먼저 사용자가 우리 웹에 파일을 업로드할 수 있게 해주어야 했죠. 하지만 제가 백엔드 개발자로부터 받은 API는 파일(Binary)을 받는 것이 아니라 MAC주소의 배열이 담긴 JSON을 받는 API였습니다. 그렇다면 파일을 그대로 서버로 전송하는 것이 아닌, **제가 직접 파일을 파싱한 후 JSON으로 만들어서 보내야하는 상황**이었습니다.

### 고전적인 파일 전송 방법

html에는 이미 오래전부터 파일 입출력을 받을 수 있는 방법이 있었습니다. 바로 `<input type="file" />`과 `<a download />`이죠. 하지만 이것들은 html이기 때문에 어떤 로직을 추가하고 싶거나 파일을 핸들링하고 싶다면 자바스크립트로 해당 태그(HTMLElement)를 먼저 가져오는 등, DOM 조작을 필요로 했습니다. 즉, 지금처럼 백엔드에 그대로 보내는 것이 아닌 파싱 같은 작업을 해야할 때는 약간 번거로운 추가 과정이 필요해집니다. 그래서 저는 자바스크립트에서 바로 파일 선택창을 띄울 수 있는 메소드는 없는지 찾아보았습니다.

### 최신 File System Access API

현대 브라우저들은 이런 필요에 의해 [File System Access API](https://developer.mozilla.org/en-US/docs/Web/API/File_System_Access_API)를 만들게 되었습니다. 이 API를 이용하면 자바스크립트에서 파일 선택창, 혹은 저장창을 바로 띄워줄 수 있습니다. 그리고 변수에 바로 파일을 담을 수 있죠. 이것은 더이상 사용자가 input을 클릭하길 기다릴 필요가 없다는 뜻입니다. 개발자가 원하는 타이밍이나 이벤트에 파일 업로드/다운로드를 바로 진행시킬 수 있다는 뜻입니다.

현재 제가하려는 것은 파일 업로드이므로 아래와 같이 간단하게 할 수 있습니다.

```js
const [fileHandle] = await window.showOpenFilePicker();
const file = await fileHandle.getFile();
const text = await file.text();
```

위 함수를 실행하면 사용자에게 파일 선택 모달을 띄우고 파일이 선택된 경우 그 내용을 string으로 가져옵니다. 사용자가 파일을 선택하길 기다려야하기 때문에 비동기로 되어있는 점을 참고해주세요.

### 그러나... 호환성은?

고전적인 input 대신 파일 선택창을 열어주는 메소드인 `showOpenFilePicker()`는 크롬과 엣지에서 2020년 10월 경부터 지원하는 메소드입니다. 하지만 Firefox나 Safari, 그리고 모바일용 브라우저에서는 전부 지원되지 않아서 다소 호환성이 떨어지므로 주의할 필요가 있습니다.

파일 선택 후 얻어지는 `FileSystemFileHandle` 인터페이스의 경우는 현재 모든 모던 브라우저에서 호환이 되지만, Firefox는 비교적 최근인 2023-03-14 버전부터 지원되기 시작했으므로 역시나 완벽하게 안심할 수는 없습니다.

그리고 `fileHandle.getFile()` 로 얻어지는 `File` 인터페이스 객체는 오래전부터 Blob의 자식 클래스로 제공되고 있었던 것이므로 이 부분만은 문제가 없겠네요.

### Polyfill (정확히는 Ponyfill)

따라서 호환성을 위해서는 여전히 고전적인 input/a 태그 방식을 사용할 필요가 있겠습니다. 하지만 이런 고민을 우리가 처음한 것이 아닐 것이란 생각에 조금 검색해보니, 무려 크롬개발자들이 만든 라이브러리가 있었습니다.

- https://github.com/GoogleChromeLabs/browser-fs-access

이 `browser-fs-access`라는 이름의 라이브러리는 파일 입출력에 대한 추상화된 함수를 제공합니다. 예를 들어 여기에서 제공하는 `fileOpen` 함수를 이용하면, 지원되는 최신 브라우저에서는 위와 같은 `showOpenFilePicker()`를 사용하고, 지원되지 않는 브라우저에서는 알아서 기존의 input 태그 방식으로 fallback시킵니다.

> This module allows you to easily use the File System Access API on supporting browsers, with a transparent fallback to the `<input type="file">` and `<a download>` legacy methods. This library is a ponyfill.

(라이브러리의 README.md 첫 문장)

앞으로 서버(API)에서 파일 업로드/다운로드를 직접 지원하지 않는 다면 위 라이브러리가 큰 도움이될 것 같습니다.