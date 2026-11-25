# cornbutter-assets

콘버터(CornButter)가 빌드 시 셀프 호스팅용으로 내려받는 대용량 바이너리(ML 모델 등)를
Release 에셋으로 올려두는 저장소입니다. 소스 코드는 별도 비공개 저장소(cornbutter)에
있습니다.

## Releases

- `models-v1` — 배경 제거(누끼) 기능용 IS-Net(DIS) int8 양자화 ONNX 모델. 출처·라이선스는
  각 Release 설명을 참고하세요.
- `qpdf-v1` — PDF 암호 해제·설정 기능용 qpdf WASM 빌드(qpdf 12.4.2, Apache-2.0). 빌드
  과정·플래그는 [`qpdf/BUILD.md`](qpdf/BUILD.md) 참고.
