# AbortController

## 알고 있던 것

- 진행 중인 요청을 중지하는 기능이다.

## 직접 구현

> 검색창: query 변경 → 이전 요청 abort 후 재요청 (DummyJSON search API)

```tsx
import { useState, useEffect } from "react";

const Search = () => {
  const [query, setQuery] = useState<string>("");
  const [isLoading, setIsLoading] = useState<boolean>(false);
  const [results, setResults] = useState([]);

  useEffect(() => {
    const controller = new AbortController();

    const request = async () => {
      if (query === "") return;
      setIsLoading(true);
      try {
        const data = await fetch(
          `https://dummyjson.com/products/search?q=${query}`,
          { signal: controller.signal },
        );
        const json = await data.json();
        setResults(json.products);
        setIsLoading(false);
      } catch (e) {
        if (e.name === "AbortError") {
          return;
        }
        console.error(e);
      }
    };

    request();

    return () => {
      controller.abort();
    };
  }, [query]);

  const handleInputChange = (e) => {
    setQuery(e.target.value);
  };

  return (
    <>
      <h1>Search</h1>
      <form>
        <input
          type="search"
          placeholder="검색어를 입력하세요"
          onChange={(e) => handleInputChange(e)}
        />
        <button>검색하기</button>
      </form>
      <div>current query: {query}</div>
      {isLoading ? (
        <div>Loading...</div>
      ) : (
        <ul>
          {results.map((el) => {
            return <li key={el.id}>{JSON.stringify(el)}</li>;
          })}
        </ul>
      )}
    </>
  );
};

export default Search;
```

## 구현하며 알게 된 것

- a→ab→abc를 빨리 치면 요청은 순서대로 나가도 응답은 순서대로 안 옴(네트워크 지연). 간혹 늦게 온 ab 응답이 abc 결과를 덮어쓰는 버그가 발생할 수 있다 (race condition)
- `const c = new AbortController()` → `c.signal`을 `fetch(url, { signal })`에 넘김, `c.abort()` 호출 시 그 fetch가 `AbortError`로 reject
- 한 번 abort된 signal은 영원히 aborted 상태 → 요청마다 controller를 새로 만들어야 함
- `useEffect([query])`에서 controller를 effect 지역변수로 두고 cleanup에서 abort → query 변경 시 React가 이전 cleanup을 먼저 돌려서 "새 요청 전 이전 요청 취소"가 자동(IntersectionObserver의 disconnect와 같은 자리)
- debounce = 요청 자체를 줄임(타이핑 멈출 때만 전송). AbortController = 나간 요청을 취소

## 막혔던 부분

1. `res.json()`에 await을 빼먹었었음
2. **`Uncaught (in promise) AbortError`**
   - React StrictMode로 useEffect가 두 번 실행됨. 첫 요청이 마운트 직후 abort되었음.
   - 이 때 1번 await 빼먹은 걸로 catch문에서 에러 처리가 안 되면서 노출됨
   - await 추가 + 에러 처리에서 aborterror 분기로 aborterror는 스킵

## AI 리뷰 결과

- 설계 흐름(signal→fetch, cleanup→abort, AbortError 분기, controller 일회용)을 완성
- 핵심 함정은 **비동기 에러가 try/catch를 빠져나가는 위치**였음 — await하는 것만 try가 잡는다. `return promise/await` 누락 주의!!
- 응답 래핑 확인 습관 — 배열이 바로 오는지, 감싸져 오는지.
- 개선 여지: debounce 결합, 로딩/에러 UI 상태.

## 실무에서 어디에 쓰나

- 검색·자동완성 — 이전 요청 취소로 stale 결과 방지
- 탭/라우트 전환 시 이전 페이지 요청 취소 (언마운트 fetch 정리)
- 파일 업로드/다운로드 중단 버튼
- React Query/SWR 내부도 AbortController로 요청 취소 처리
- 한 줄 요약: debounce로 요청 수를 줄이고, AbortController로 나간 요청 중 오래된 것을 끊어 race condition을 없앤다.
