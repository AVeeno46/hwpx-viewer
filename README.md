# hwpx 편집기 (보관용)

한글 `.hwpx` 파일을 **브라우저에서 직접** 열어 본문 텍스트를 고치고 다시
저장하는 코드다. rag-docgen 클라이언트에서 쓰다가 hwpx 열람을 서버 변환 PDF
방식으로 바꾸면서 화면에서 연결이 끊겼고, 쓰지 않게 되어 여기로 옮겼다.

원래 위치: `rag-docgen/apps/client/my-ai-app-mvvm/`
분리 시점: 2026-08-05

## 들어 있는 것

| 파일 | 원래 경로 | 줄 수 | 설명 |
|---|---|---|---|
| `src/models/hwpxModel.js` | `src/models/hwpxModel.js` | 604 | 편집 로직 전부 |
| `src/models/hwpxModel.test.js` | `src/models/hwpxModel.test.js` | 174 | 위 모델의 테스트 |
| `src/components/HwpxDocView.jsx` | `src/features/note/center/HwpxDocView.jsx` | 462 | hwpx를 DOM으로 그리는 뷰어 |

## hwpxModel.js — 무엇을 하나

`.hwpx`는 실제로는 **XML이 든 zip**이다. JSZip으로 열어 `Contents/section*.xml`을
읽고, 고친 뒤 다시 zip으로 묶는다.

- `openHwpx(arrayBuffer)` — zip을 열어 섹션 XML들을 파싱
- `extractSegments(xml, sectionIndex)` — 화면에 보여줄 텍스트 조각(run) 단위로 추출
- `parseDocumentStructure(...)` — 문단·글자 속성까지 포함한 구조 파싱
- `injectEmptyRuns(xml)` — 빈 문단에도 편집 가능한 run을 하나 넣어준다
  (원본에 run이 없으면 사용자가 그 줄에 글자를 넣을 자리가 없다)
- `extractIngestSegments({ sections })` — 검색/임베딩용 텍스트만 뽑기
- `saveHwpx({ zip, sections }, editsBySection, outputType)` — 고친 내용을 반영해
  다시 `.hwpx`로 묶기

### 핵심 설계: "런 단위 교체"

문서 전체를 다시 만들지 않고 **바뀐 run의 텍스트 노드만 갈아끼운다.** hwpx에는
글꼴·자간·표·도형 같은 정보가 잔뜩 들어 있어서, 전체를 재생성하면 손대지 않은
서식까지 망가진다. 그래서 원본 XML을 최대한 그대로 두고 바뀐 부분만 교체한다.

## HwpxDocView.jsx — 무엇을 하나

파싱한 구조를 브라우저 DOM으로 그린다. HWPUNIT(1/7200인치)을 96dpi 기준
px로 환산하고(`÷75`), 문단·표·도형을 배치한다.

한계가 주석에 남아 있다 — **쪽 나눔은 재현하지 못한다.** 한글의 페이지 구분은
파일에 저장돼 있지 않고 레이아웃 엔진이 계산하는 값이라서다. 도형도 위치
근사치가 부정확한 종류(그룹·다각형·글맵시·OLE)는 아예 그리지 않는다. 본문 위에
지저분하게 겹쳐 보이기 때문이다.

## 왜 안 쓰게 됐나

hwpx **열람**을 브라우저 변환에서 **서버 변환 PDF**로 바꿨다(2026-07-29).
브라우저에서 변환하면 원본과 쪽 구성이 달라지고 표가 페이지 경계에서 잘렸다.
지금은 서버 `GET /file/get-pdf/{file_id}`가 변환한 PDF를 받아서 보여준다.

열람이 PDF로 가면서 **편집기를 붙일 자리도 같이 없어졌다.** PDF는 고칠 수
없으니, 편집을 되살리려면 별도의 편집 화면을 다시 만들어야 한다.

## 되살리려면

`jszip` 의존성이 필요하다(`npm i jszip`). 테스트는 vitest 기준이다.
편집 화면(파일 열기 → 텍스트 수정 → 저장)은 이 저장소에 없다 — 모델과
뷰어만 있고, 그 둘을 엮던 화면 컴포넌트는 이미 지워진 뒤였다.
