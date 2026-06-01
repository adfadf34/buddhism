# KCI OpenAPI 연동 및 개발 가이드

이 문서는 발급받으신 KCI(한국학술지인용색인) API 키를 활용하여 기존 대시보드를 정적 데이터(`kci_data.json`) 방식에서 **실시간 동적 데이터 연동 방식**으로 업그레이드하기 위한 가이드 및 계획서입니다.

## 🔑 API 인증 정보
- **API Key**: `74737035`
- **제공처**: 한국연구재단 KCI (Korea Citation Index)
- **주요 활용 목적**: 실시간 논문 검색, 피인용 횟수 조회, 저자 및 학술지 정보 동적 로딩

---

## 🛠 1. 기본 API 호출 구조 (Endpoint)

KCI OpenAPI는 RESTful 기반으로 XML 형태의 데이터를 반환합니다. 발급받으신 키를 사용하여 다음과 같은 형태로 요청을 보낼 수 있습니다.

### 논문 검색 API 예시 (논문명 조회)
```http
https://open.kci.go.kr/po/openapi/openApiSearch.kci?apiCode=articleSearch&key=74737035&title=불교&displayCount=10
```

### 필수 및 주요 파라미터
* `apiCode`: 사용할 API 종류 (예: `articleSearch` 논문검색)
* `key`: 발급받은 API 키 (`74737035`)
* `title` / `author` / `keyword`: 검색하려는 조건
* `displayCount`: 한 번에 가져올 결과 수 (최대 100건 권장)

---

## 💻 2. 자바스크립트 연동 예제 (Fetch API)

프론트엔드 환경(현재 대시보드)에서 API를 직접 호출하려면 CORS(교차 출처 리소스 공유) 문제가 발생할 수 있습니다. 이를 해결하기 위한 클라이언트 사이드 호출 예제입니다.

```javascript
// KCI API 호출 함수 예제
async function searchKCIPapers(keyword) {
    const API_KEY = "74737035";
    // KCI는 기본적으로 XML을 반환하므로 주의가 필요합니다.
    const url = `https://open.kci.go.kr/po/openapi/openApiSearch.kci?apiCode=articleSearch&key=${API_KEY}&keyword=${encodeURIComponent(keyword)}&displayCount=20`;

    try {
        const response = await fetch(url);
        const xmlText = await response.text();
        
        // XML을 파싱하여 사용하기
        const parser = new DOMParser();
        const xmlDoc = parser.parseFromString(xmlText, "text/xml");
        
        // 논문 아이템 추출
        const items = xmlDoc.getElementsByTagName("item");
        const results = Array.from(items).map(item => ({
            id: item.getElementsByTagName("articleInfo")[0]?.getAttribute("article-id"),
            title: item.getElementsByTagName("title-group")[0]?.getElementsByTagName("article-title")[0]?.textContent,
            author: item.getElementsByTagName("author-group")[0]?.getElementsByTagName("author")[0]?.textContent,
            year: item.getElementsByTagName("pub-year")[0]?.textContent,
            url: item.getElementsByTagName("url")[0]?.textContent
        }));
        
        console.log("검색 결과:", results);
        return results;
    } catch (error) {
        console.error("KCI API 호출 중 오류 발생:", error);
    }
}
```

---

## 🚀 3. 대시보드 연동 적용 방안 (Next Steps)

현재 대시보드 구조에 API를 적용하기 위한 3단계 전략입니다.

1. **기존 데이터 확장 (Hybrid 방식)**
   * 초기 로딩은 기존처럼 속도가 빠른 `kci_data.json`을 사용하여 트리를 구성합니다.
   * 사용자가 특정 노드(키워드)를 클릭하거나 방금 만든 '통합 검색창'에서 검색할 때만 KCI API를 호출하여 **최신 논문 정보를 실시간으로 추가**합니다.

2. **CORS 우회 및 프록시 설정**
   * 웹 브라우저에서 KCI 서버로 직접 요청 시 보안 문제(CORS)로 막힐 수 있습니다.
   * `index.html` 단일 파일 빌드 환경을 유지하기 위해, 퍼블릭 CORS 프록시(예: `cors-anywhere`)를 사용하거나, 추후 서버(Node.js 등)를 작게 띄우는 방식을 고려해야 합니다.

3. **XML to JSON 변환 파이프라인 구축**
   * KCI 데이터는 XML 구조이므로, 대시보드의 D3.js나 React 컴포넌트가 이해하기 쉽도록 실시간으로 JSON으로 파싱(Parsing)해주는 유틸리티 함수를 기존 자바스크립트 코드에 추가합니다.

---
> **Tip:** 앞으로 검색바에서 키워드를 칠 때 이 API 키를 사용하도록 `search_library.html` 코드를 수정할 수 있습니다!
