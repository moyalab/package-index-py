# 자체 Package Index 만들기

`https://raw.githubusercontent.com/arduino/package-index-py/main/package-list.yaml`
같은 자체 MicroPython 패키지 인덱스를 직접 호스팅해서 IDE의 "Add Package"
다이얼로그에 노출시키는 방법.

## 1. 저장소 만들기

GitHub(또는 GitLab)에 **공개(Public) 저장소**를 새로 만든다 (예:
`myorg/my-mpy-packages`). raw URL의 CORS·익명 접근은 GitHub raw 도메인이 항상
허용하므로 추가 호스팅 설정이 필요 없다. 비공개 저장소면 `?token=...` 쿼리가
필요해 IDE에서 쓰기 곤란하므로 **반드시 public**으로 둔다.

## 2. YAML 파일 추가

저장소 루트(또는 원하는 경로)에 `package-list.yaml`을 만든다. 스키마는
[arduino/package-index-py](https://github.com/arduino/package-index-py/blob/main/package-list.yaml)
를 그대로 따르면 우리 IDE의 [package-installer.js](../ui/arduino/bridges/web/package-installer.js)
가 **수정 없이** 그대로 읽는다.

```yaml
packages:
  - name: My Sensor Lib
    url: https://github.com/myorg/my-sensor-lib       # github:owner/repo 도 가능
    author: Your Name
    description: One-line summary shown in the dialog
    tags: ["sensors", "i2c"]
    license: MIT
    docs: https://github.com/myorg/my-sensor-lib#readme   # 선택
  - name: Another Package
    url: github:myorg/another-pkg@v1.2.0   # @버전 고정 가능
    description: ...
    tags: ["util"]
```

필수: `name` + (`url` 또는 micropython-lib 인덱스 조회용 이름만). 사용 가능
필드는 `name / url / author / description / tags / license / docs`.

## 3. 패키지 측 요구사항 (`package.json`)

목록의 `url`이 가리키는 각 GitHub repo에는 [원본 micropython-lib 규약](https://github.com/micropython/micropython-lib)
을 따르는 **`package.json`**이 루트에 있어야 설치 가능하다:

```json
{
  "version": "1.0.0",
  "urls": [
    ["my_sensor_lib/__init__.py", "github:myorg/my-sensor-lib/my_sensor_lib/__init__.py"],
    ["my_sensor_lib/driver.py",   "github:myorg/my-sensor-lib/my_sensor_lib/driver.py"]
  ],
  "deps": [
    ["micropython-lib-name", "latest"]
  ]
}
```

`urls`의 두 번째 칸은 `github:...` / `gitlab:...` / `https://...` / 상대경로 모두
지원 ([package-installer.js:341-358](../ui/arduino/bridges/web/package-installer.js#L341-L358)).

## 4. raw URL 얻기

저장소가 `myorg/my-mpy-packages`, 브랜치가 `main`, 파일이 `package-list.yaml`
이라면:

```
https://raw.githubusercontent.com/myorg/my-mpy-packages/main/package-list.yaml
```

패턴 변환:

```
github.com/{owner}/{repo}/blob/{branch}/{path}
       ↓
raw.githubusercontent.com/{owner}/{repo}/{branch}/{path}
```

## 5. IDE에 연결

[package-installer.js:25-28](../ui/arduino/bridges/web/package-installer.js#L25-L28)의
`REGISTRY_URLS` 배열에 추가하면 된다:

```js
const REGISTRY_URLS = [
  'https://raw.githubusercontent.com/arduino/package-index-py/main/package-list.yaml',
  'https://raw.githubusercontent.com/arduino/package-index-py/main/micropython-lib.yaml',
  'https://raw.githubusercontent.com/myorg/my-mpy-packages/main/package-list.yaml',
]
```

여러 YAML이 OR로 합쳐지고 `name` 기준 정렬된다. arduino 목록 없이 자체 목록만
쓰려면 위 두 줄을 지우고 자기 URL만 남긴다.

## 6. 운영 시 알아둘 것

- **캐시**: GitHub raw는 응답에 `Cache-Control: max-age=300` 정도가 붙어 약 5분
  정도 옛 버전이 보일 수 있다. 즉시 반영이 필요하면 URL에 `?v=YYYYMMDDHHmm`
  같은 dummy 쿼리를 매번 바꿔서 호출. (우리 코드는 `cache: 'no-cache'`로 브라우저
  캐시는 우회한다.)
- **버전 핀**: `main` 대신 태그/커밋 SHA로 안정 버전을 고정할 수 있다 —
  `https://raw.githubusercontent.com/myorg/repo/v1/package-list.yaml`.
- **CDN 대안**: 트래픽이 많아지면 jsDelivr가 더 안정적 —
  `https://cdn.jsdelivr.net/gh/myorg/my-mpy-packages@main/package-list.yaml`
  (CORS·캐시 우호적, 버전 별 URL 자동 지원).
- **검증**: YAML이 잘못되면 다이얼로그 로딩 시 fetch는 성공하지만 파싱 단계에서
  throw → "패키지 레지스트리 불러오기 실패" 토스트가 뜬다. 푸시 전에 `js-yaml`
  등으로 로컬 검증 권장.
- **표시되는 정보**: `name`(굵게), `description`(부제), `url`(링크), `tags`(태그)
  4가지. 좋은 description과 일관된 tag 컨벤션이 검색·발견성을 좌우한다.

## 최소 시작 절차 요약

1. GitHub에 public repo 생성 (예: `myorg/my-mpy-packages`).
2. 루트에 `package-list.yaml` 추가, 위 스키마로 패키지 1개 작성하고 push.
3. `https://raw.githubusercontent.com/myorg/my-mpy-packages/main/package-list.yaml`
   이 브라우저에서 열리는지 확인.
4. 우리 IDE의 `REGISTRY_URLS`에 그 URL 추가, 새로고침.
5. "Add Package" 다이얼로그에서 본인 패키지가 검색되면 OK.
