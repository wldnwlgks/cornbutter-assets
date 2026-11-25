# qpdf WASM 빌드 — BUILD.md

콘버터 v3 B단계(PDF 암호 해제·설정)용. qpdf CLI를 Emscripten으로 WASM으로 빌드해서
Web Worker 안에서 `--decrypt`/`--encrypt`를 그대로 호출한다. 크립토는 qpdf 내장(native)
구현만 켜고(OpenSSL·GnuTLS 제외), 의존 라이브러리는 zlib·libjpeg(IJG)만 쓴다 —
라이선스는 전부 MIT/Apache-2.0보다 느슨한 permissive 라이선스.

## 버전

| 구성요소 | 버전 | 라이선스 | 출처 |
| --- | --- | --- | --- |
| qpdf | 12.4.2 (commit `4eba95899886e851cc41d76886483b347612f2a8`) | Apache-2.0 | https://github.com/qpdf/qpdf |
| zlib | 1.4 | zlib 라이선스 | Emscripten port(`emcc -sUSE_ZLIB=1`), https://github.com/madler/zlib |
| libjpeg | 9f | IJG 라이선스 | Emscripten port(`emcc -sUSE_LIBJPEG=1`), https://www.ijg.org |
| Emscripten | 7.0.1 (`ac0578743420bcbe9a4f77d0b6599c25f8d43702`) | — (빌드 도구일 뿐, 산출물에 라이선스 영향 없음) | emsdk |

## 빌드 환경

WSL2 Ubuntu 26.04, 16 코어. Windows에는 Emscripten 네이티브 빌드 체인이 없어서 WSL을 썼다.

```bash
# 1. 빌드 도구
sudo apt-get install -y build-essential cmake git python3 xz-utils

# 2. emsdk
git clone --depth 1 https://github.com/emscripten-core/emsdk.git
cd emsdk && ./emsdk install latest && ./emsdk activate latest
source ./emsdk_env.sh

# 3. qpdf 소스 — 정확히 이 태그로 고정
git clone https://github.com/qpdf/qpdf.git qpdf-src
cd qpdf-src && git switch --detach v12.4.2
```

## 설정 (CMake)

```bash
mkdir build-wasm && cd build-wasm
emcmake cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DBUILD_SHARED_LIBS=OFF \
  -DBUILD_STATIC_LIBS=ON \
  -DREQUIRE_CRYPTO_NATIVE=ON \
  -DUSE_IMPLICIT_CRYPTO=OFF \
  -DSTATIC_JPEG=ON \
  -DBUILD_DOC=OFF \
  -DINSTALL_MANUAL=OFF \
  -DCMAKE_C_FLAGS='-sUSE_ZLIB=1 -sUSE_LIBJPEG=1 -fwasm-exceptions' \
  -DCMAKE_CXX_FLAGS='-sUSE_ZLIB=1 -sUSE_LIBJPEG=1 -fwasm-exceptions' \
  -DCMAKE_EXE_LINKER_FLAGS='-fwasm-exceptions -sMODULARIZE=1 -sEXPORT_ES6=1 -sEXPORT_NAME=createQpdfModule -sENVIRONMENT=worker -sALLOW_MEMORY_GROWTH=1 -sINVOKE_RUN=0 -sEXIT_RUNTIME=0 -sEXPORTED_RUNTIME_METHODS=["callMain","FS"] -sFILESYSTEM=1 -sSTACK_SIZE=2097152 -sASSERTIONS=0'

cmake --build . --target qpdf -j 8
```

빌드 결과: `build-wasm/qpdf/qpdf.js`(ES 모듈, `createQpdfModule` 팩토리 함수 default export) +
`build-wasm/qpdf/qpdf.wasm`.

### 플래그가 왜 이런지

- `-DREQUIRE_CRYPTO_NATIVE=ON -DUSE_IMPLICIT_CRYPTO=OFF`: OpenSSL·GnuTLS를 빼고 qpdf 자체
  AES/MD5/SHA2/RC4 구현만 쓴다(지시문 요구사항 그대로). 외부 암호 라이브러리가 없어서
  바이너리 용량이 작고 라이선스도 깔끔해진다.
- `-sUSE_ZLIB=1 -sUSE_LIBJPEG=1`: zlib·libjpeg을 직접 크로스컴파일하지 않고 Emscripten
  공식 포트를 쓴다 — 포트가 WASM에 맞게 이미 검증된 빌드라 안정적이고, 라이선스도
  업스트림 그대로(zlib 라이선스, IJG) 유지된다.
- `-fwasm-exceptions`(컴파일+링크 양쪽 다): **필수.** 이게 없으면 qpdf가 잘못된 비밀번호
  같은 "정상적인 실패"를 C++ 예외로 던질 때 Emscripten이 그 예외를 잡지 못하고
  `Aborted(Assertion failed: Exception thrown, but exception catching is not enabled)`로
  죽어버린다(실제로 처음 빌드했을 때 이 문제를 겪음 — 전체 .cc 재컴파일 필요했음).
  `-fwasm-exceptions`는 브라우저 네이티브 WASM 예외 처리라 레거시 JS 기반 예외 처리보다
  빠르고 가볍다(Chrome 95+·Firefox 100+·Safari 15.2+ 전부 지원, 이 프로젝트 타깃 브라우저
  범위 안).
- `-sEXIT_RUNTIME=0` + 모듈 생성 시 `{ noExitRuntime: true }`: qpdf의 `main()`은 끝나면
  `exit()`를 호출하는데, 기본 설정으로는 이게 `process.exit()`급으로 런타임을 통째로
  죽여버려서 재사용이 안 된다. 이 설정을 켜면 `exit()`가 깔끔한 JS 예외(`ExitStatus`)로만
  끝나고, `callMain()`이 정상적으로 exit code를 반환한다(확인됨: 틀린 비밀번호 →
  `callMain` 반환값 2, 예외 안 던짐).
- `-sENVIRONMENT=worker`: 이 모듈은 Web Worker 안에서만 쓴다. Node 환경 코드를 안 넣어서
  용량을 줄인다(테스트할 때만 일시적으로 `worker,node`로 바꿔서 썼다).
- `-sMODULARIZE=1 -sEXPORT_ES6=1 -sEXPORT_NAME=createQpdfModule`: `import createQpdfModule
  from "./qpdf-core.js"` 형태로 쓸 수 있는 평범한 ES 모듈 팩토리 함수를 만든다. 호출마다
  `await createQpdfModule({...})`로 새 인스턴스를 만들어 쓴다(상태 격리, 동시 처리 안전).
- `-sSTACK_SIZE=2097152`(2MB): 기본 스택(64KB)로는 중첩이 깊은 PDF 객체 트리에서
  스택 오버플로가 날 수 있어 넉넉히 늘렸다.

## 검증(이 빌드로 직접 확인한 것)

Node.js에서 `ENVIRONMENT=worker,node`로 임시 재빌드해 테스트(아래는 최종 `worker` 전용
빌드와 로직은 동일, 로딩 방식만 다름):

- 256비트 AES 암호화(`--encrypt user owner 256 --`) → 정상 생성, `--show-encryption`으로
  `stream/string/file encryption method: AESv3` 확인(진짜 AES-256, RC4/40비트 아님)
- 올바른 비밀번호로 `--decrypt` → 원본과 동일한 PDF 복원(`%PDF-1.7` 헤더 확인)
- **틀린 비밀번호** → `callMain` 반환값 `2`, stderr에 `invalid password` — 예외로 안 죽고
  깔끔하게 처리됨(위 `-fwasm-exceptions` 수정 후)
- 열기 암호 없이 권한 암호만 건 PDF(`--encrypt "" owner 256 --print=none --modify=none`)
  → 비밀번호 없이 `--decrypt` 성공, 권한 제한만 해제됨(지시문의 "권한 암호는 해제 가능"
  요구사항과 일치)
- `--is-encrypted` → 암호 있는 파일 0, 없는 파일 2(종료 코드로 깔끔하게 구분 — `not_encrypted`
  실패 코드 구현에 그대로 쓸 수 있음)

브라우저 Web Worker 환경에서의 최종 통합 테스트는 콘버터 레포 쪽 e2e(Playwright)에서
별도로 수행한다(이 레포 범위 밖).

## 파일 목록

- `qpdf.js`, `qpdf.wasm` — 빌드 결과물(콘버터가 받아 씀)
- `LICENSE-qpdf.txt` — qpdf Apache-2.0 라이선스 전문
- `LICENSE-zlib.txt` — zlib 라이선스 전문
- `LICENSE-libjpeg.txt` — IJG 라이선스 전문(libjpeg README 발췌)

## SHA256

```
qpdf.js:   2f4909ec7503d40821110954ae02b21c8513bb772d0ba3d5569906acfaac3aa5
qpdf.wasm: 43dbf19b9fdf0d5148a452889f9eb29a20c548cdb8aad54cd918b1314a1d135e
```

GitHub Release: `qpdf-v1` 태그에 `qpdf.js`·`qpdf.wasm`·세 라이선스 파일 첨부.
콘버터 쪽 `scripts/build-workers.mjs`가 빌드 시 이 Release 자산을 받아 SHA256 대조 후
`public/qpdf/`에 복사한다(기존 ONNX 모델과 같은 패턴).
