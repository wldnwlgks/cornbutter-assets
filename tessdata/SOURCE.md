# Tesseract 언어 데이터 — SOURCE.md

콘버터 v4 ⑤ 이미지 텍스트 추출(OCR)용. Tesseract LSTM 엔진의 "fast" 학습 데이터(한국어·영어)를
그대로 재배포한다.

## 출처

| 파일 | 언어 | 버전 | 라이선스 | 출처 |
| --- | --- | --- | --- | --- |
| `kor.traineddata.gz` | 한국어 | tessdata_fast 4.0.0 | Apache-2.0 | https://github.com/naptha/tessdata (gh-pages 브랜치, `4.0.0_fast/kor.traineddata.gz`) — tesseract-ocr/tessdata_fast 공식 릴리즈를 그대로 미러링한 저장소 |
| `eng.traineddata.gz` | 영어 | tessdata_fast 4.0.0 | Apache-2.0 | 위와 동일 저장소, `4.0.0_fast/eng.traineddata.gz` |

원본을 변형하지 않고 그대로 가져왔다(바이트 동일 — sha256은 콘버터 레포의
`scripts/build-workers.mjs`에 박아둔 값과 대조). `naptha/tessdata`는 tesseract.js가
기본으로 쓰는 CDN(`tessdata.projectnaptha.com`)의 소스 저장소이기도 하다 — 콘버터는
"처리는 전부 브라우저 안에서, 외부 서버 호출 없이"라는 원칙을 지키려고 이 CDN에 직접
요청하지 않고, 같은 파일을 이 저장소(cornbutter-assets)의 Release로 옮겨 자체 호스팅한다.

## 왜 "fast"고 "best"가 아닌가

"best"(정확도 우선, 느림)보다 "fast"(gzip 후 1~2MB대, 빠름)를 골랐다 — 브라우저에서
실시간으로 돌리는 용도라 속도가 더 중요하다고 판단했고, fast 모델도 인쇄된 문서·화면
캡처 텍스트에는 충분한 정확도를 보인다(결정 로그 참고: cornbutter 레포
`docs/overnight-decisions.md`).

## 라이선스

Apache License 2.0 — `LICENSE.txt` 참고(원본 저장소에서 그대로 가져옴).
