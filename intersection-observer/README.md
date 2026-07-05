# Intersection Observer

## 알고 있던 것

- threshold 기반으로 요소가 경계에 다다르면 콜백 함수를 실행한다.
- 무한스크롤, 이미지 최적화 등에 사용된다.

## 파고들며 궁금해진 것

- Intersection Observer는 왜 등장했을까?

## 직접 구현

1. **scroll 이벤트 + `getBoundingClientRect`로 직접 계산**
2. **`useIntersectionObserver` 커스텀 훅** — API를 훅으로 추상화
3. **`useInfiniteQuery` 미니 버전**

가장 처음 짠 1차 스케치 (버그 포함, 아래 "막혔던 부분" 참고):

```tsx
import { useState, useRef, useEffect } from "react";

const options = { rootMargin: "300px", threshold: 1.0 };

export default function App() {
  const targetRef = useRef<HTMLElement>();
  const [page, setPage] = useState<number>(1);

  useEffect(() => {
    const observer = new IntersectionObserver(() => {
      console.log("target detected!");
    }, options);

    observer.observe(targetRef.current);

    return () => {
      observer.disconnect(targetRef.current);
    };
  }, []);

  return (
    <div className="App">
      <div style={{ height: "2000px" }}></div>
      <div ref={targetRef} style={{ border: "1px solid red" }}>
        this is the target
      </div>
    </div>
  );
}
```

### 1단계 — scroll + gBCR (naive)

Intersection Observer 대신 직접 가시성 판단하는 코드를 작성해봄

```jsx
const targetRef = useRef(null);

useEffect(() => {
  const targetEl = targetRef.current;
  let isIntersected = false;

  const handleScroll = () => {
    if (isIntersected) return;

    const { top, bottom } = targetEl.getBoundingClientRect();
    const isInViewport = top < window.innerHeight && bottom > 0;

    if (isInViewport) {
      isIntersected = true;
      console.log("핸들러 실행");
    }
  };

  const throttledScroll = throttle(handleScroll, 200);

  window.addEventListener("scroll", throttledScroll); // 등록·제거는 같은 참조!
  handleScroll(); // 초기 1회 체크

  return () => {
    window.removeEventListener("scroll", throttledScroll);
  };
}, []);
```

가시성 판단식:

- `gBCR` 좌표는 **viewport 기준**(스크롤하면 값이 바뀜). 화면에 보이는 세로 구간은 `0` ~ `window.innerHeight`.
- `top < window.innerHeight && bottom > 0`
  - `top < innerHeight`: 요소 위쪽이 viewport 아래 끝보다 위 (아직 화면 밑으로 안 내려감)
  - `bottom > 0`: 요소 아래쪽이 viewport 위 끝보다 아래 (아직 화면 위로 안 사라짐)
- 이 `isIntersecting` ≈ 이게 intersection observer에서 쓰이는 `entry.isIntersecting`.

### 2단계 — useIntersectionObserver 훅

```jsx
const observer = new IntersectionObserver(
  (entries) => {
    entries.forEach((entry) => {
      if (entry.isIntersecting) {
        /* 여기서 기대 동작 실행 */
      }
    });
  },
  { threshold: 0.2, rootMargin: "0px", root: null },
);

observer.observe(targetEl); // 관찰 시작 (+ 초기 상태 1회 콜백 실행)
observer.unobserve(targetEl); // 특정 타겟만 관찰 중단
observer.disconnect(); // 모든 관찰 중단
```

- **options (트리거 설정)**: `threshold`, `rootMargin`, `root`
  - `threshold` = "이 비율 경계를 넘나들 때 콜백 불러줘". `0.2` → 20% 보일 때.
- **entries (결과)**: `entry.isIntersecting`(boolean), `entry.intersectionRatio`(실제 비율) 등
- **root vs target** (헷갈렸던 부분): `root`는 가시성 판단의 _기준 영역_(Intersection observer 미사용 구현에서 `window.innerHeight`와 비교했던 그 viewport)이고, *관찰 대상*은 `options.root`가 아니라 `observe(el)`로 넘긴다. ← 여기에 `ref.current`.
  - `root`가 대부분 `null`인 이유: `null` = 브라우저 viewport. lazy loading·무한스크롤·광고 노출은 다 "화면 기준"이라 viewport로 충분. 특정 요소로 주는 건 스크롤이 페이지가 아닌 내부 컨테이너(`overflow: scroll` 모달/패널 등)에서 일어날 때.

최종 구현:

```tsx
import { useEffect, useState, useRef } from "react";

export const useIntersectionObserver = ({
  threshold = 0,
  rootMargin = "0px",
  root = null,
}: IntersectionObserverInit = {}) => {
  const ref = useRef<HTMLElement>(null);
  const [isIntersecting, setIsIntersecting] = useState(false);

  useEffect(() => {
    if (!ref.current) return;
    const observer = new IntersectionObserver(
      ([entry]) => {
        setIsIntersecting(entry.isIntersecting);
      },
      { threshold, rootMargin, root },
    );

    observer.observe(ref.current);
    return () => observer.disconnect();
  }, [threshold, rootMargin, root]);

  return [ref, isIntersecting] as const;
};
```

설계 결정:

- **반환은 `[ref, isIntersecting]`** — 쓰는 쪽이 `<div ref={ref}>`로 붙이고 상태로 반응. observe/정리는 훅이 감춘다.
- **`isIntersecting`은 `useState`** 로 관리해야 값 변경 시 리렌더 가능
- **콜백 주입 여부 (설계 A vs B)**
  - A: 상태 반환형 — 훅 내부에서 `setIsIntersecting`만. lazy loading에 적합.
  - B: 콜백 주입형 — "보이면 뭘 할지"가 매번 다른 무한스크롤 등에 적합 (`onIntersect`).
  - 실제 `react-intersection-observer`는 `[ref, inView, entry]` 반환 + `onChange` 콜백을 둘 다 지원함

### 3단계 — useInfiniteQuery 미니 버전

캐싱은 제외하고 intersection observer가 어떤 식으로 사용되는지 확인해보기 위해 구현

무한스크롤 훅이 들고 있어야 할 상태:

- `data` — 페이지 **누적** (`[...이전, ...새 페이지]`)
- `isFetching` — 로딩 중 여부. `if (isFetching) return` 가드로 중복 요청 방지.
- `hasNextPage` — 다음 페이지 존재 여부.
- `pageParam` — 다음에 부를 페이지 식별자(번호 또는 cursor).

트리거 흐름:

```
바닥 센티넬 <div ref={ref} /> → isIntersecting: false→true
  → (그리고 !isFetching && hasNextPage) → fetchNextPage()
```

```jsx
useEffect(() => {
  if (isIntersecting && !isFetching && hasNextPage) fetchNextPage();
}, [isIntersecting, isFetching, hasNextPage]); // 셋 다 deps에 (stale closure 방지)
```

- deps에서 `isFetching`/`hasNextPage`가 빠지면 stale closure → 낡은 값으로 잘못 판단.

**연쇄 로딩(chain loading)**: fetch 완료로 `isFetching`이 `true`에서 `false`로 바뀌며 effect가 재실행됨. 이때 타겟 요소가 아직 화면에 보이면(한 페이지가 화면을 못 채움) 스크롤 없이 또 fetch되며 화면에서 벗어날 때까지 반복할 수 있음.  
대체로 정상 동작이지만, pageSize가 너무 작거나 `hasNextPage` 판단이 틀리면 폭주할 수 있음.

설계 결정:

- 요청 성공 후 page 증가
- 트리거 위치
  - tanstack-query는 `useInfiniteQuery`가 `{ data, isFetching, hasNextPage, fetchNextPage }`만 반환하고, 센티넬+`isIntersecting` 트리거는 사용처에서 담당
- `hasNextPage`를 훅이 직접 계산하지 않는다.
  - 백엔드 페이지네이션 모양을 모르므로 사용처에서 `getNextPageParam(lastPage, allPages)`를 주입한다. 반환이 `undefined`면 다음 없음 → `hasNextPage = getNextPageParam(...) !== undefined`.

  ```js
  // offset: (lastPage, allPages) => lastPage.hasMore ? allPages.length : undefined
  // cursor: (lastPage) => lastPage.nextCursor ?? undefined
  ```

  **`TODO`: 구현 예정**

## 구현하며 알게 된 것

- **`unobserve` vs `disconnect`**: `unobserve`는 지정된 타겟 하나만, `disconnect`는 모든 타겟의 관찰을 그만둔다.
- **`([entry]) =>` vs `forEach`**: 타겟 1개만 observe하면 entries 길이가 항상 1 → 첫 요소만 구조분해하면 된다. 여러 타겟을 한 observer로 볼 때만 `forEach`가 필요.
- **`isIntersecting`을 true일 때만 넣으면** 화면 밖으로 나가도 false로 안 돌아온다 → one-shot이면 OK, 범용이면 `setIsIntersecting(entry.isIntersecting)`로 양방향 반영.
- **의존성 배열 함정**:
  - `ref`는 `useRef`가 준 객체라 참조가 안 바뀜 → `ref.current` 변경은 deps로 안 잡힌다.
  - `options`를 객체째 deps에 넣으면 호출부의 객체 리터럴이 매 렌더 새 참조 → observer가 계속 재생성된다.
  - `useMemo(() => options, [options])`로는 못 고친다(입력이 이미 불안정). → **원시값(`threshold`, `rootMargin`)을 deps에 직접** 넣으면 useMemo도 필요 없다. 옵션 객체는 effect 안에서 만든다(지역 변수라 무관).
- **cleanup은 언제 실행되나**: unmount될 때, 그리고 **다음 effect 실행 직전**. 이걸 빼먹고 있었다.
- 타입: 옵션 객체 타입은 `IntersectionObserver`(인스턴스)가 아니라 `IntersectionObserverInit`. 반환 튜플은 `as const`로 고정해야 구조분해 타입이 정확.
- naive 버전에서: `window`엔 React 합성 이벤트(`onScroll` prop)를 못 붙인다(합성 이벤트는 JSX 엘리먼트에만). `useEffect`에서 `addEventListener`로 직접 등록이 정석. scroll 이벤트는 버블링도 안 한다.

## 막혔던 부분

- **`observer.observe()` 인자로 `ref.current`가 아니라 `ref` 자체를 넘겨버림.**
- **observer를 컴포넌트 내부에 만들었다가 `useEffect`로 옮김.** Intersection Observer 자체는 한 번만 만들어져도 된다.
- **왜 위로 스크롤할 때에도 콜백이 실행됐지?** → Intersection Observer는 교차 _상태가 변경될 때_ 실행되기 때문. `true → false`도, `false → true`도 둘 다 실행된다. 그래서 콜백에서 `isIntersecting`일 때를 체크하는 것.
- **cleanup에서 `disconnect`가 아니라 `unobserve`를 했더니 오류.**
  - 처음엔 "cleanup 시점엔 이미 ref가 정리돼서?"라고 짐작 → 정확히는 `ref.current`를 잘못 참조하고 있었기 때문. cleanup 시점엔 `ref.current`가 이미 없을 수 있다. `const target = ref.current`로 변수에 담아뒀으면 괜찮다.
- **초기 상태 누락**: 마운트 시 이미 화면 안에 있으면 scroll이 한 번 일어나야 감지됨 → 등록 직후 `handleScroll()` 1회 호출로 보완. (Intersection Observer는 `observe` 시 초기 1회 콜백을 호출해줌)

개념적으로 아직 확실히 정리 안 된 것:

- scroll 이벤트는 메인 스레드에서 처리된다. 반면 Intersection Observer의 콜백은 비동기로 메인 스레드와 분리돼 실행된다 — 이 부분을 이벤트 루프와 연결지어 더 볼 수 있음.

## 다음 단계

- **훅을 콜백 주입형(설계 B)으로 확장하면 생기는 질문들** :
  - `callback`이 바뀌면? — `page`를 참조하는 콜백은 `page`가 바뀔 때마다 새로 생성된다. observer를 다시 만들까, 기존 걸 재사용할까?
  - `options`가 바뀌면? (`threshold: 0.5 → 1`) observer를 새로 만들까?
  - `target`이 바뀌면? 무한스크롤은 target이 고정이라 문제없지만, 리스트 아이템 감시처럼 `oldTarget → newTarget`으로 바뀌면 `unobserve(oldTarget)` / `observe(newTarget)`를 어떻게 처리할까?
- `rootMargin`, `threshold`, `scrollMargin`의 정확한 차이 정리.

## 실무에서 어디에 쓰나

- **무한스크롤** — 바닥 센티넬 감지 → 다음 페이지 fetch
- **이미지 lazy loading** — 화면 진입 시 `src` 로드 (다수 요소에 observer 달아놓고 경계 잡히는지 체크)
- **광고/콘텐츠 노출 측정** — viewport에 실제로 보였는지
- **코드 스플리팅** — 화면에 보일 때 dynamic import
