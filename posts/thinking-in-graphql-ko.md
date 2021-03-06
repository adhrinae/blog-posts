---
title: (번역) Thinking in GraphQL
author: Hyeseong Kim
date: 2019-04-08T05:30:00
tags:
  - graphql
  - relay
  - react
---

## 역주: 들어가며

이 글은 Facebook의 데이터 기반 리액트 프레임워크인 [Relay](https://facebook.github.io/relay)의 문서 [Thinking in GraphQL](https://facebook.github.io/relay/docs/en/thinking-in-graphql.html)를 번역한 글입니다.

저는 릴레이라는 프레임워크를 사용하진 않지만 GraphQL을 만든 페이스북의 문제해결 방식을 엿볼 수 있는 좋은 글이라 생각되어 번역해보았습니다.

GraphQL는 (서버와 클라이언트 양측 모두) 인메모리 캐시를 적극적으로 활용하여 기존 REST API에 있던 문제점을 해결하며 데이터를 현대적인 뷰(React) 구조에 맞게 제공합니다.

캐시를 통한 데이터 정규화 과정은 GraphQL을 다루는 서버와 클라이언트 모두 알아야 하는 핵심적인 부분임에도 GraphQL을 소개할 때 잘 언급되지 않으며 REST API에 비해 비교적 쉬운 진입장벽만 강조되어 섣불리 GraphQL을 도입했다가 어려움을 겪는 과정을 몇 번 목격했습니다.

온갖 프레임워크가 쏟아지는 오늘날 어떤 프레임워크가 더 예쁘고 편한 API를 제공하느냐를 아는 것 만큼 그 이면에 숨겨진 철학과 기술적 접근방식을 이해하는 것이 중요합니다. 이 번역을 통해 국내 커뮤니티에서 GraphQL에 대한 (올바른 측면의) 이해도가 개선될 수 있기를 희망합니다.

----

GraphQL은 클라이언트 애플리케이션에서 개발자가 필요한 데이터에 불러오는데 초점을 맞춰, 뷰에 필요한 데이터를 정확하게 지정해서 한 번의 요청으로 통해 불러올 수 있는 새로운 방법을 제시합니다. GraphQL은 리소스 기반의 REST 같은 기존 접근 방법에 비해 더 효과적으로 데이터를 불러오고, 서버 측에서 중복된 로직이 반복되거나, 이를 피하기 위해 커스텀 엔드포인트를 추가하는 일을 작업을 방지하며, 더 나아가서 제품 코드를 서버의 로직으로부터 분리(Decouple)하는 것을 돕습니다. 예를 들면, 관련 서버 엔드포인트의 변경 없이 더 많거나 적은 정보를 가져오게끔 변경할 수 있습니다.

이 글에서는 GraphQL 클라이언트 프레임워크를 구성하는 요소들을 살펴보고 기존 REST 기반 시스템과 어떤 차이점이 있는지 알아보겠습니다.

## 데이터 불러오기

스토리 목록을 불러오고, 각 스토리별 추가 디테일을 불러오는 간단한 애플리케이션을 상상해보세요. 리소스 기반 REST API를 사용하는 코드는 다음과 같을 겁니다.

```js
// 스토리 ID 목록을 가져옵니다. 상세정보는 포함되어 있지 않습니다.
rest.get('/stories').then(stories =>
  // This resolves to a list of items with linked resources:
  // 여기서 링크된 리소스로부터 아이템 목록을 가져옵니다.
  // `[ { href: "http://.../story/1" }, ... ]`
  Promise.all(stories.map(story =>
    rest.get(story.href) // 링크 따라가기
  ))
).then(stories => {
  // 스토리 아이템 목록을 가져왔습니다.
  // `[ { id: "...", text: "..." } ]`
  console.log(stories);
});
```

이 방법은 n+1회 요청을 필요로 합니다. 1은 목록을 불러오고, n은 각 항목의 상세를 불러오는거죠. GraphQL을 사용하면 같은 데이터를 한 번의 네트워크 요청으로 가져올 수 있습니다. (관리해야 할 커스텀 엔드포인트를 추가하지 않고도 말이죠)

```js
graphql.get(`query { stories { id, text } }`).then(
  stories => {
    // 스토리 아이템 목록을 가져왔습니다.
    // `[ { id: "...", text: "..." } ]`
    console.log(stories);
  }
);
```

이 코드는 GraphQL을 사용한 것만으로 일반적인 REST 접근법 보다 더 효율적인 버전이 됐습니다.

- 모든 데이터를 단 한 번 네트워크 왕복으로 불러왔습니다.
- 클라이언트는 필요한 데이터가 있는 서버 엔드포인트들에 의존하는 대신 필요한 데이터를 표현하게 됐습니다.  
  (서버 클라이언트 디커플링)

간단한 애플리케이션에서도 벌써 많은 개선이 보입니다.

## 클라이언트 캐싱

서버로부터 반복적으로 정보를 다시 불러오는 것은 앱을 상당히 느리게 만들 수 있습니다. 예를 들면, 스토리 목록에서 특정 항목으로, 그리도 다시 목록으로 돌아오면 전체 목록을 다시 한 번 불러오게 됩니다. 이를 해결하는 표준적인 방법이 바로 *캐싱*입니다.

리소스 기반의 REST 시스템에선, **응답 캐시**를 URI에 기반해서 관리합니다.

```js
var _cache = new Map();
rest.get = uri => {
  if (!_cache.has(uri)) {
    _cache.set(uri, fetch(uri));
  }
  return _cache.get(uri);
};
```

GraphQL도 마찬가지로 응답 캐시를 사용합니다. 기본적인 접근법은 REST 비슷하지만, 캐시의 키 값으로써 쿼리 자체가 사용된다는 것이 다릅니다.

```js
var _cache = new Map();
graphql.get = queryText => {
  if (!_cache.has(queryText)) {
    _cache.set(queryText, fetchGraphQL(queryText));
  }
  return _cache.get(queryText);
};
```

이제 이전에 캐싱 된 데이터를 통해 네트워크 요청 없이 바로 응답을 만들 수 있습니다. 캐시는 애플리케이션의 유의미한 성능 향상을 위한 실용적인 방법이지만, 데이터 일관성 문제를 유발할 수 있습니다.

## 캐시 일관성

GraphQL을 사용하면 여러 쿼리 결과가 겹치는 일이 매우 일반적입니다. 하지만 이전 섹션에서 설명한 캐시 방식은 이러한 중첩을 처리하고 있지 않습니다. — 각각이 고유한 쿼리를 기반으로 캐싱되고 있습니다. 예를 들어 스토리 목록을 가져오는 경우:

```graphql
query { stories { id, text, likeCount } }
```

그리고 `likeCount`가 하나 증가된 후에 개별 항목을 가져오는 경우:

```graphql
query { story(id: "123") { id, text, likeCount } }
```

이제 스토리를 어디서 보느냐에 따라 서로 다른 `likeCount` 값을 보게 됩니다. 첫 번째 쿼리를 사용하는 뷰는 지난 값을 사용하고 있고, 두 번째 쿼리를 사용하는 뷰는 새로운 값을 사용하고 있습니다.

### 그래프 캐싱하기

GraphQL이 이 문제를 해결하는 방법은 여러 계층의 레코드 응답을 단일 계층의 컬렉션으로 정규화(Nomalize)하는 것입니다.

릴레이는 이를 레코드의 ID를 기반으로 하는 맵으로 구현합니다. 여기서 각 레코드는 필드의 이름과 값을 기반으로 하는 맵입니다. 레코드들은 다른 레코드(순환 그래프를 허용)에 링크될 수 있고, 링크들은 최상위 맵으로 다시 참조되는 특수한 타입으로 저장됩니다.

이 방식을 통해 각 서버 레코드는 어느 화면으로부터 불리던 관계 없이 *한 차례*만 저장됩니다.

스토리의 텍스트와 작성자의 이름을 가져오는 쿼리 예제입니다.

```graphql
query {
  story(id: "1") {
    text,
    author {
      name
    }
  }
}
```

그리고 그 응답입니다.

```js
query: {
  story: {
     text: "Relay is open-source!",
     author: {
       name: "Jan"
     }
  }
}
```

이런 계층적인 응답이 주어지면 릴레이는 모든 레코드를 다음과 같이 병합합니다.

```js
Map {
  // `story(id: "1")`
  1: Map {
    text: 'Relay is open-source!',
    author: Link(2),
  },
  // `story.author`
  2: Map {
    name: 'Jan',
  },
};
```

이 건 아주 단순한 예시일 뿐입니다. 실제 캐시는 one-to-many 관계과 페이지네이션을 처리해야하기 때문에 더 복잡합니다.

### 캐시 사용하기

그래서 이 캐시는 어떻게 사용될까요? 관련된 두 가지 작업에 대해 알아보겠습니다.

1. 응답을 받고 캐시에 저장하는 작업
2. 캐시를 읽고, 쿼리가 로컬에서 전부 처리될 수 있는지를 결정하는 작업  
  (위에서 구현한 `_cache.has(key)`와 거의 동일하지만, 그래프 버전)

### 캐시 채우기

캐시를 쓰는 작업에는 계층적인 응답으로부터 정규화된 레코드를 생성하거나, 정규화된 캐시 레코드를 업데이트하는 일이 포함됩니다.

처음에는 응답 자체만 처리해도 될 수 있지만, 이는 아주 간단한 쿼리들만 해당되는 얘기입니다. `user(id: "456") { photo(size: 32) { uri } }` 쿼리를 생각해보세요. — 어떻게 `photo`를 저장해야 할까요? `photo`라는 이름만 사용하면 다른 인자 값을 사용해서 쿼리하는 경우(예시: `photo(size: 64) {...}`)에는 제대로 동작하지 않습니다. 비슷한 이슈로 페이지네이션을 처리할 때 `stories(first: 10, offset: 10)` 이라는 쿼리로 11번 째부터 20번 째 스토리 목록을 불러온다면 새로운 결과들은 기존 목록 뒤에 추가되어야 합니다.

따라서 GraphQL에서 응답 캐시 정규화 작업은 페이로드와 쿼리 구문을 함께 처리해줘야 합니다. 예시로 들었던 `photo` 필드는 이름과 인자 값을 구분하기 위해 `photo_size(32)` 같은 고유한 필드를 생성해서 캐싱합니다.

### 캐시에서 데이터 읽기

캐시로부터 각 필드의 값을 불러오려면 쿼리를 읽고 각 필드를 Resolve 하면 됩니다. 그런데 잠깐, 이건 GraphQL 서버가 쿼리를 처리할 때 하는 일과 똑같네요?

그렇습니다! 캐시를 읽는 것은 마치 특수한 경우의 쿼리 처리기 처럼 동작합니다.

1. 모든 결과가 고정된 데이터 구조에서 나오며 커스텀 필드 리졸버가 필요하지 않고
2. 결과가 항상 동기화 되어 있는 경우

— 캐시 된 데이터는 있을 수도 없을 수도 있는데 말이죠.

릴레이는 쿼리문 탐색(캐시나 응답 페이로드의 데이터와 함께 쿼리를 처리하는 작업)의 몇 가지 변형을 구현하고 있습니다. 예를 들면, 쿼리를 수행할 때 릴레이는 "diff" 탐색을 실행해서 어떤 필드들이 누락되어 있는지(리액트가 가상DOM 트리를 diff 하는 과정과 매우 흡사함)를 판단합니다. 이렇게 하면 평균적으로 가져오는 데이터 양을 줄일 수 있고, 쿼리가 완전히 캐시되어 있다고 판단되면 릴레이는 아예 네트워크 요청을 하지 않게 됩니다.

### 캐시 업데이트

정규화된 구조를 통해 중첩된 결과를 중복 없이 캐시하는 방법을 알아보았습니다. 이제 예제로 돌아가 이 캐싱 기법이  기존 시나리오의 일관성 문제를 어떻게 해결하는지 확인해봅시다.

첫 번째로 스토리 목록을 불러오는 쿼리가 있습니다.

```graphql
query { stories { id, text, likeCount } }
```

정규화된 응답 캐시에는 목록의 모든 스토리들에 대한 레코드가 생성됩니다. 그리고 `stories` 필드에 레코드들에 대한 링크가 저장됩니다.

하나의 스토리 정보를 불러오기 위해 다시 다음 쿼리가 수행되는 경우

```graphql
query { story(id: "123") { id, text, likeCount } }
```

이 쿼리의 응답이 정규화 되면, 캐시는 해당 `id` 필드를 가진 기존 데이터와 겹치는 것을 감지하고 새 레코드를 만드는 대신 기존 `123` 레코드를 업데이트 합니다. 이제 이 두 쿼리, 뿐만 아니라 스토리를 참조하는 모든 쿼리가 새로운 `likeCount` 값을 참조하게 되었습니다.

## 데이터/뷰 일관성

캐시 정규화를 통해 캐시의 일관성을 보장했습니다. 하지만 뷰는 어떨까요? 이상적으로는, 리액트 뷰는 항상 캐시의 현재 정보를 반영해야합니다.

스토리의 텍스트와 작성자 정보와 커멘트를 함께 렌더링하는 경우를 고려해보세요. 쿼리는 다음과 같을겁니다.

```graphql
query {
  story(id: "1") {
    text,
    author { name, photo },
    comments {
      text,
      author { name, photo }
    }
  }
}
```

초기에 스토리를 불러오면 다음과 같이 캐시 됩니다. 여기서 스토리와 댓글은 동일한 `author` 레코드를 참조합니다.

```js
// Note: 이건 `Map` 초기화의 구조를 좀 더 분명하게 설명하기 위한 의사코드입니다.

Map {
  // `story(id: "1")`
  1: Map {
    text: 'got GraphQL?',
    author: Link(2),
    comments: [Link(3)],
  },
  // `story.author`
  2: Map {
    name: 'Yuzhi',
    photo: 'http://.../photo1.jpg',
  },
  // `story.comments[0]`
  3: Map {
    text: 'Here\'s how to get one!',
    author: Link(2),
  },
}
```

스토리의 저자가 댓글도 등록했습니다 — 꽤 일반적인 상황이죠 :p 이제 다른 뷰에서 작성자에 대한 추가 정보와 그녀의 변경된 프로필 사진 정보를 가져온다고 가정합시다. 이 경우, 캐시 된 데이터 중 오직 *한 부분만*이 바뀝니다.

```js
Map {
  ...
  2: Map {
    ...
    photo: 'http://.../photo2.jpg',
  },
}
```

`photo` 필드의 값이 변경 되었기 때문에 2번 레코드가 변경되었습니다. 이게 전부입니다. 캐시에서 그 이외 부분은 영향을 받지 않습니다. 하지만 뷰는 작성자와 스토리 UI에 새로운 사진을 표시하기 위해 전체적으로 변경될 필요가 있습니다.

가능한 흔한 해결책은 "그냥 불변(Immutable) 객체를 쓰자" 입니다. — 그러면 어떤 일이 일어나는지 확인 해보죠.

```js
ImmutableMap {
  1: ImmutableMap {/* 이전과 같음 */}
  2: ImmutableMap {
    ... // 다른 필드는 변경되지 않았음
    photo: 'http://.../photo2.jpg',
  },
  3: ImmutableMap {/* 이전과 같음 */}
}
```

`2`번을 새로운 불변 레코드로 교체하면, 캐시 객체도 새로운 불변 인스턴스를 얻게 됩니다. 하지만 레코드 `1`과 `3`은 변경되지 않았습니다. 데이터가 정규화 되어 있기 때문에 스토리 레코드만 보면 스토리의 컨텐츠 변경을 알 수가 없습니다.

### 뷰 일관성 보장하기

정규화된 캐시에서 뷰의 일관성을 유지하는 다양한 솔루션이 있습니다. 그 중에서 릴레이가 사용하는 방법은 각 UI 뷰의 데이터 매핑을 참조하는 ID들의 세트로써 관리하는 것입니다.

위 예제에서 스토리 뷰는 `1` 스토리, `2` 작성자, `3`을 포함한 댓글들의 변경내용을 각각 구독합니다. 캐시에 데이터를 쓸 때, Relay는 영향을 받는 ID들을 추적해서 뷰에 통보합니다. 더 나은 성능을 위해 영향을 받는 뷰만 다시 렌더링되고, 영향을 받지 않는 뷰는 무시할 수 있습니다. (릴레이는 안전하고 효율적인 기본 `shouldComponentUpdate` 메서드를 제공합니다) 이 전략이 없다면 모든 뷰가 아주 작은 변경사항에도 매번 다시 렌더링 될 것입니다.

이 솔루션은 쓰기 작업들에도 동일하게 적용됩니다.

## 뮤테이션

데이터를 쿼리해오며 어떻게 최신 상태를 유지하는지 알아보았습니다. 이번엔 쓰기 과정을 알아봅시다. GraphQL에서의 쓰기 과정은 **Mutation**이라고 불립니다. 이 것은 쿼리와 사이드이펙트가 혼합된 형태로 볼 수 있습니다.

스토리에 좋아요 표시를 하는 뮤테이션을 호출하는 예시입니다.

```graphql
# 읽기 좋은 이름과 입력 타입을 함께 정의합니다.
# 여기서는 좋아요 표시할 스토리의 아이디
mutation StoryLike($storyID: String) {
   # 뮤테이션 필드를 호출해서 사이드 이펙트를 트리거 합니다.
   storyLike(storyID: $storyID) {
     # 뮤테이션이 완료된 후 다시 가져올 필드들을 정의합니다.
     likeCount
   }
}
```

뮤테이션 호출로 (아마도) 변경된 데이터를 쿼리했습니다. 여기서 분명 왜 서버가 변경된 것만 알아서 알려줄 수 없는지 궁금할 수 있습니다. 그건 좀 복잡한 문제입니다.

GraphQL은 *모든 종류*의 데이터 저장 레이어(또는 여러 소스의 집합)를 추상화하며, 어떤 프로그래밍 언어와도 함께 동작합니다. 더 나아가 GraphQL의 목적은 제품 개발자가 뷰를 만드는데 유용한 폼으로 데이터를 제공하는 것입니다.

우리는 GraphQL 스키마가 실제 디스크에 저장되는 데이터의 형태와는 조금 아니 거의 대체로 다르다는 것을 발견했습니다. 간단히 말하자면, *제품 내* 데이터 구조(GraphQL)와 데이터 저장소(디스크) 레이어의 데이터는 항상 1:1로 대응하진 않습니다. 그 예시로 개인정보의 경우 `age` 같은 개인정보 필드를 접근할 수 있는지 판단하기 위해 데이터 저장소 레이어에서 수 많은 레코드에 접근해야 될 수 있습니다. (친구인지, 공유된 정보인지, 블록되지 않았는지 등등)

이런 현실적인 제약사항들을 감안한 GraphQL의 접근법은 뮤테이션 이 후 결과들이 변경될 수 있는 것들만 쿼리하는 것입니다. 그럼 쿼리에는 정확이 어떤 것들이 들어가나요? 릴레이를 개발하면서 우리는 몇 가지 아이디어를 탐색했습니다. — 릴레이의 접근 방식을 이해하기 위해 몇 가지 다른 접근 방식들과 함께 살펴봅시다.

- 옵션 1: 앱에서 쿼리한 적 있는 것들을 모조리 다시 불러옵니다. 데이터 중 아주 작은 부분만 변경되더라도 서버가 전체 쿼리를 수행하고, 결과를 다운로드하고, 그게 다시 후처리 되는 것을 기다려야 합니다. 이 방식은 매우 비효율적입니다.

- 옵션 2: 현재 뷰에서 보여지는 것들만 다시 불러옵니다. 이 방식은 옵션 1 보다는 조금 개선되었습니다. 그러나 캐시된 데이터 중 현재 뷰에 보여지지 않는 부분들은 업데이트 되지 않습니다. 이 데이터들이 어떤식으로든 만료된 데이터로 표시되지 않는 한, 이후 쿼리는 오래된 정보를 읽게 될 것입니다.

- 옵션 3: 뮤테이션 이후 변경될 수 있는 고정 필드 목록을 다시 불러옵니다. 우리는 이걸 **팻 쿼리**라고 부릅니다. 실제로는 팻 쿼리 중에 일부만 렌더링하기 때문에 이 방법 또한 비효율적이긴 하지만 제대로 데이터를 불러오긴 할 것입니다.

- 옵션 4: 변경될 수 있는 부분(팻 쿼리)과 캐시 된 데이터의 겹치는 부분만 다시 불러옵니다. 릴레이는 데이터 캐시에 추가로 각 항목을 가져오는 데 사용된 쿼리를 기억하고 있습니다. (역주: 캐시의 키 값으로 저장되어 있음) 이 것을 **추적된 쿼리**라고 부릅니다. 이 추적된 쿼리와 팻 쿼리를 교차시킴으로써 릴레이는 앱에서 업데이트 할 필요가 있는 정보만을 정확히 쿼리할 수 있습니다.

## 데이터 패칭 API

지금까지 우리는 데이터 패칭의 저레벨 측면과 다양한 기존 컨셉들이 어떻게 GraphQL에 적용되었는지 살펴보았습니다.

이제 뒤로 한 번 물러나 제품 개발자가 데이터 패칭과 관련해서 자주 접하는 고레벨 관점의 문제들을 살펴봅시다.

- 뷰 계층구조에 따라 데이터 불러오기
- 비동기 상태 전이와 요청 동시성 관리하기
- 에러 관리하기
- 실패한 요청 재시도하기
- 쿼리/뮤테이션 응답에 따라 로컬 캐시 업데이트하기
- 경쟁상태를 피하기 위해 뮤테이션을 큐에 넣기
- 서버의 뮤테이션 응답을 기다리는 동안 UI를 낙관적(Optimistically) 업데이트하기

우리는 필수 API를 사용하는 데이터 수집에 대한 전형적인 접근방식들이 개발자들에게 불필요한 복잡성을 너무 많이 다루도록 강요한다는 것을 알게됐습니다.

서버 응답을 기다리는 동안 사용자에게 피드백을 주는 방법인 낙관적 UI 업데이트가 그 예시입니다. 무엇을 해야하는지 로직은 꽤 명확합니다. 사용자가 "좋아요"를 클릭하면, 스토리를 좋아하는 것으로 표시하고 서버에 요청을 보내야 합니다. 하지만 구현은 종종 훨씬 복잡해집니다. 명령형 접근방식을 사용하면  UI에서 버튼을 가져와 토글하고, 네트워크 요청을 보내고, 필요하면 재시도하고, 실패하면 에러를 보여주고 (버튼 토글을 취소하고) 기타 등등의 절차들을 모두 구현해야합니다. 데이터 패칭도 마찬가지입니다. 필요한 데이터가 *무엇인지* 지정하기에 따라 데이터를 가져오는 *방법*과 *시점*이 달라지는 경우가 많습니다. 다음 글에서 릴레이가 이런 문제들을 어떻게 해결하는지 알아보겠습니다.

----

## 역주: 마무리하며

목차 상 다음 문서는 [Thinking in Relay](http://facebook.github.io/relay/docs/en/thinking-in-relay.html)입니다. 이어서 번역할 계획은 없지만 이 글에서 설명한 내용들이 리액트와 어떻게 통합되는지를 설명한 비교적 짧은 글이기에 한 번 읽어볼 것을 추천드립니다.

문맥을 매끄럽게 하기 위한 의역과 내용을 정확히 전달하기 위한 직역이 뒤죽박죽 섞여있습니다. 더 나은 번역에 대한 피드백은 언제든시 환영합니다.

원 글의 저작권은 MIT 라이센스에 따라 Facebook에 있으며 번역에 대한 저작권은 명시하지 않습니다. (아래 CC는 무시)
