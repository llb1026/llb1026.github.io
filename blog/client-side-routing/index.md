---
order: 7
layout: post
title: "클라이언트 사이드 라우팅과 단점들"
subtitle: "React.js에서 CSR를 구현하며 겪은 삽질과 각 해결방법들"
tag: Lesson Learned
type: lesson-learned
blog: true
text: true
author: JIYUN LEE
post-header: true
header-img: img/01_1.png
next-link: "../backend-performance-test/"
prev-link: "../server-side-rendering/"
---

## 동작 방식

과거에는 유저가 특정 URL을 서버에 요청하면 서버에서 그 URL에 해당하는 결과 페이지를 넘겨주는 단순한 구조였다. 여기에 프론트엔드 프레임워크가 끼면서 요청된 URL에 대해 서버 측과 클라이언트 측 둘 다 어떻게 동작할지 알고 있어야 하게 되었다. 따라서 클라이언트 측에도 라우팅이 필요해졌고, React에서는 **React Router**를 이용해 클라이언트 사이드 라우팅을 할 수 있다.

서버 API에 필요한 데이터를 요청보내는 것을 제외하고, static 리소스 페이지에 대해 React Router는 다음과 같이 동작한다.

1. 맨 처음 요청은 항상 서버로 보내진다.
2. 서버로부터 React나 React Router 등을 로드하기 위한 필요한 스크립트를 받는다. 이제 React Router가 어느 URL에서 어떤 뷰를 렌더링할 것인지 결정할 수 있다.
3. 사용자가 버튼이나 링크를 통해 `http://example.com/about` 페이지에 접근하면 [History API](https://developer.mozilla.org/en-US/docs/Web/API/History_API#Adding_and_modifying_history_entries)를 통해 주소창의 URL이 *로컬상에서* 바뀐다. 여기서 중요한 점은 서버로 요청이 가는게 아니라, React가 해당 URL에 필요한 뷰를 렌더링해준다.

## 문제점

만약 위의 1, 2번 단계를 생략하고 바로 주소창에 `http://example.com/about`를 입력하면 React Router가 동작하기 전이므로, 해당 요청은 서버로 보내질 것이다. 그러면 서버에서는 `/about`에 대해 어떤 응답을 보낼지 알지 못하므로 404 에러가 발생한다.

`/about` URL은 React Router로 인해 클라이언트 사이드에서는 제대로 동작하지만, 서버 사이드에서는 서버에 `/about`에 대해 어떻게 동작할지 명시해주지 않는 이상 정상적으로 응답하지 못한다.

## 해결 방법

### Hash History

기존의 문제 상황에서는 다음과 같이 React Router를 사용하고 있었다.

```
import {BrowserRouter} from "react-router-dom";

...
    <BrowserRouter>
        </App>
    </BrowserRouter>
...
```

여기서 `BrowserRouter`를 `HashRouter`로 바꿔 적용하면 [BrowserHistory](https://github.com/jintoppy/react-training/blob/master/basic/node_modules/react-router/docs/guides/Histories.md#browserhistory) 대신 [HashHistory](https://github.com/jintoppy/react-training/blob/master/basic/node_modules/react-router/docs/guides/Histories.md#hashhistory)를 사용하게 되어 URL이 `http://example.com/#/about`과 같이 바뀐다. 가운데에 들어간 `#` 심볼 이후에 오는 URL 부분은 서버로 보내지지 않고, 서버는 `http://example.com/`까지만 보고 이에 해당하는 index 페이지를 응답으로 보내준다. 그리고 React Router가 `#/about` 부분에 따라 렌더링을 진행한다.

#### 단점

- URL이 못생겨진다.
- 서버 사이드 렌더링이 불가능해진다.
- production 모드에서는 권장하지 않는 방식이다.
- SEO 측면에서 최악이다. 

### Catch-all

BrowserHistory를 사용하되, 서버측에서 `/*` 요청에 대해 무조건 index 페이지를 리턴하도록 설정한다. URL이 깔끔해진다.

#### 단점

- Hash History 방법보다 설정하기가 까다롭다.
- 여전히 SEO 측면에서 좋지 않다.

### Hybrid

위의 Catch-all 방법을 좀 더 확장한 것으로, 특정 URL에 대해 어떤 페이지를 리턴할 것인지 정의해놓은 스크립트를 추가한다. 이렇게 할 경우 Googlebot이 최소한 해당 페이지에 뭐가 있는지는 파악할 수 있게 된다.

#### 단점

- 더 설정하기가 까다롭다.
- 스크립트에 정의해놓은 경로에 대해서만 SEO가 괜찮아진다.
- 페이지 리소스를 로드하는 데에 클라이언트와 서버에서 코드가 중복된다.

### Isomorphic

서버 사이드 렌더링을 하면 된다. [이전 블로그 글](../server-side-rendering/)에 자세한 내용을 적었다.

#### 단점

- 서버가 javascript를 구동할 수 있어야 한다.
- 서버 사이드에서 `window`를 사용하는 경우를 비롯해 개발하기 까다로운 부분이 다수 있다.
- 이제까지의 해결 방법 중에서 러닝 커브가 제일 크다.