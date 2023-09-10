---
title: 'Lodash를 사용할 때 번들 사이즈 줄이기'
date: 2023-9-10 22:12:01
category: atWork
# thumbnail: { thumbnailSrc }
draft: false
---

### 배경 (약간의 TMI)

자바스크립트에는 꼭 있어야할 것 같은데 없는 함수나 메소드가 종종 있습니다. 대표적으로는, 두 오브젝트가 동일한 지 확인하는 방법입니다.

```javascript
const obj = { 'a': 1 };
const other = { 'a': 1 };
 
obj === other; // false!
```

자바스크립트에서는 두 객체의 내용이 동일하더라도 별도로 선언된 것이라면 다른 객체로 봅니다. 이런 경우에는 `Object` 프로토타입에서 뭔가 비교할 수 있는 내장 메서드를 제공하는 것이 당연할 것 같지만 그런 것이 없습니다.

그래서 이걸 굳이 구현하자면 `JSON`을 이용하는 방법이 있습니다.

```javascript
// 문자열로 변환 후 비교
JSON.stringify(obj) === JSON.stringify(other)
```

다행히 누군가가 이런 불편함을 위해 먼저 다양한 유틸함수들을 구현해두었는데, 이런 대표적인 유틸 함수 라이브러리가 바로 `lodash`입니다.

```javascript
import _ from "lodash";

_.isEqual(object, other); // true
```

하지만 `lodash`는 그냥 `import`해서 사용하면 번들 사이즈를 크게 잡아먹습니다.

> ![lodash-bundle-size](https://yrnana.dev/_astro/lodash.15dc1d67_uuW15.webp)

왜냐하면 lodash에는 내가 사용하고 싶은 함수 외에도 수많은 유틸 함수들이 구현되어 있기 때문인데, 굳이 사용하지 않을 함수까지 포함하여 번들 사이즈를 크게 만들 이유가 없죠.

프론트엔드 개발자에게 번들 사이즈란, 사용자가 보는 로딩 속도와 연관이 있으므로 관리하는 것이 필수입니다. 그렇다면 번들 사이즈를 줄이는 방법은 무엇이 있을까요?

### Tree Shaking

현대 프론트엔드 개발에서는 작성한 JS 코드가 서버에 그대로 올라가는 경우는 드뭅니다. 보통 트랜스파일, 번들링 과정을 거쳐서 Minified된 버전이 최종적으로 서버에 올라갑니다. 따라서 이 과정에서 불필요한 코드(죽은 코드)를 제거할 수 있습니다. 이 과정을 `Tree Shaking`이라고 부릅니다.

일반적으로 트리 쉐이킹은 우리가 흔히 사용하는 Webpack(Babel)이나 Vite와 같은 번들러가 알아서 해주기 때문에 우리가 사용할 함수만 명시적으로 `import`를 해주면 됩니다.

```javascript
import _ from "lodash"; // (x) lodash 전부를 불러오고 있음
import { isEqual } from "lodash"; // 일반적으로는 옳은 방법 그러나...
```

하지만 lodash는 **CommonJS**라는 방식의 모듈 시스템을 이용하여 개발되었기 때문에, 두번째 방법으로 import하더라도 번들러가 트리 세이킹을 하기 어렵습니다. 이런 경우에 해결할 수 있는 방법 세가지가 있습니다.

### 1) babel-plugin-lodash

이 방법은 import문을 require로 변환하는 babel에 lodash를 올바르게 tree-shaking할 수 있도록 도와주는 플러그인을 설치하는 것입니다. 자세한 설명은 https://github.com/lodash/babel-plugin-lodash를 참고하시기 바랍니다. 하지만 babel 설정을 직접 건드려야하기 때문에 아래 소개할 더 간편한 방법을 사용하는 것이 낫습니다.

### 2) cherry picking

애초에 from 절에 참조할 경로를 좁히는 방법입니다.

```javascript
import isEqual from "lodash/isEqual";
```

이렇게 하면 lodash 전체가 아닌 하위의 `isEqual`이 정의된 파일만 포함하므로 번들 사이즈가 줄어듭니다. 별도로 설치해야할 것이 없지만, 만일 불러올 함수가 많다면 **일일이 한줄씩** 불러와야하므로 역시 저는 선호하지 않습니다.

### 3) lodash-es

그래서 오늘의 핵심인 `lodash-es` 라이브러리를 소개합니다. 이 라이브러리는 lodash 대신 설치하여 사용하면 트리 쉐이킹을 가능하게 해줍니다.

```javascript
import { isEqual } from "lodash-es";
```

여러 함수를 불러온다고 해도 한줄에 다 쓸 수 있는 장점이 있고, 특별히 어떤 설정이 필요하지도 않습니다.

앞으로 lodash를 종종 이용하게 될 것 같은데, `lodash-es`를 설치해서 사용하는 것이 번들 사이즈를 줄일 수 있겠습니다.