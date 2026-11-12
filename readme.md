# Three.js 기반 3D 디지털 트윈 프로젝트 분석 및 실행 가이드

제공된 두 개의 HTML 파일(`smart_classroom_camera_stable.html`, `ThreeJS_korean_bus.html`)은 외부 3D 모델 파일(`.glb`, `.gltf`)이나 이미지 텍스처 없이 **순수 JavaScript 절차적 지오메트리(Procedural Geometry)와 HTML5 Canvas 텍스처**만으로 구현된 단일 파일(Single-File) 웹 애플리케이션입니다.

---

## 1. 프로젝트 개요 비교

| 구분 | 스마트 학교 교실 (`smart_classroom_camera_stable.html`) | 한국형 전기 시내버스 (`ThreeJS_korean_bus.html`) |
| :--- | :--- | :--- |
| **주제** | IoT 환경 센서 및 설비 제어 디지털 트윈 | 스마트팩토리 8단계 버스 조립 공정 디지털 트윈 |
| **Three.js 버전** | `v0.164.1` (jsDelivr CDN) | `v0.180.0` (jsDelivr CDN) |
| **레이아웃 구조** | 전체 화면 3D 캔버스 + 플로팅 글래스모피즘 패널 | CSS Grid 3단 레이아웃 (좌측 공정 / 중앙 3D / 우측 관제) |
| **핵심 기능** | • 실시간 가상 IoT 수치 연산 ($\text{CO}_2$, 온도, 조도)<br>• 카메라 벽면 자동 절개(Cutaway) 및 경계 제한<br>• JSON 스냅샷, CSV 로그, PNG 캡처 내보내기 | • 8단계 순차 조립 애니메이션 및 분해 보기(Explode)<br>• 외판 반투명화(X-RAY) 및 노선별 도색 변경<br>• Raycaster 기반 3D 부품 클릭 및 상세 정보 조회 |

---

## 2. 실행 방법

두 파일 모두 ES Module(`<script type="module">`)과 `importmap`을 사용하여 CDN에서 Three.js 라이브러리를 불러옵니다.

### ① 사전 요구 사항
* **인터넷 연결**: 초기 실행 시 `cdn.jsdelivr.net`에서 Three.js 코어 및 `OrbitControls` 모듈을 다운로드합니다.
* **최신 웹 브라우저**: WebGL 2.0 및 Import Maps를 지원하는 Chrome, Edge, Firefox, Safari 최신 버전(하드웨어 가속 활성화 권장).

### ② 실행 절차
1. **로컬 웹 서버 실행 (권장)**
   브라우저의 CORS(교차 출처 리소스 공유) 보안 정책으로 인해 로컬 파일을 직접 열면 일부 환경에서 모듈 로딩이 차단될 수 있습니다. 로컬 서버를 통해 실행하는 것이 가장 안전합니다.
   * **VS Code 사용 시**: `Live Server` 확장 프로그램 설치 후, HTML 파일 우클릭 $\rightarrow$ **Open with Live Server** 클릭.
   * **Python 사용 시**: 파일이 있는 폴더에서 터미널을 열고 아래 명령어 실행 후 브라우저에서 `http://localhost:8000` 접속.
     ```bash
     python -m http.server 8000
     ```
2. **직접 실행 (간편 방법)**
   * 최신 Chrome 또는 Edge 브라우저에서는 파일을 더블 클릭하거나 브라우저 창으로 드래그 앤 드롭하여 바로 실행할 수도 있습니다.

---

## 3. 전체적인 코드 및 문법 구조

두 파일은 외부 빌드 도구(Webpack, Vite 등) 없이 단일 HTML 파일 내부에 **HTML + CSS + JS**가 모두 통합된 구조를 따릅니다.

```text
├── <!DOCTYPE html>
├── <head>
│   ├── <meta> (뷰포트, 테마 컬러 설정)
│   ├── <style> (CSS 변수, 플로팅/그리드 레이아웃, 반응형 미디어 쿼리)
│   ├── <script type="importmap"> (Three.js CDN 경로 매핑)
│   └── <script type="application/json" id="sceneSpec"> (교실 파일 전용: 3D 씬 메타데이터)
└── <body>
    ├── UI 마크업 (헤더, 제어 패널, 카메라 전환 버튼, 로딩/에러 폴백 화면)
    ├── <script> (일반 스크립트: 모듈 로딩 실패 및 타임아웃 예외 처리)
    └── <script type="module"> (메인 Three.js 3D 엔진 및 상태 관리 로직)
```

---

## 4. 핵심 문법 및 아키텍처 상세 분석

### ① ES Modules & Import Map
번들러 없이 브라우저에서 직접 `three` 패키지를 임포트하기 위해 `<script type="importmap">`을 정의합니다.
```html
<script type="importmap">
{
  "imports": {
    "three": "https://cdn.jsdelivr.net/npm/three@0.164.1/build/three.module.js",
    "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.164.1/examples/jsm/"
  }
}
</script>
<script type="module">
  import * as THREE from 'three';
  import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
</script>
```

### ② 헬퍼 함수를 활용한 절차적 3D 모델링 (Procedural Modeling)
외부 3D 에셋을 로드하지 않고 기본 기하학 도형(`BoxGeometry`, `CylinderGeometry`, `SphereGeometry`, `TubeGeometry`)을 조립하기 위해 재사용 가능한 헬퍼 함수를 정의하여 사용합니다.
* **교실 파일 (`smart_classroom_camera_stable.html`)**: `group()`, `bx()`, `plane()`, `cyl()`, `sphere()`, `rod()` 함수를 통해 책상 24세트, 학생 아바타, 창문, 에어컨 등을 계층 구조(`THREE.Group`)로 생성합니다.
* **버스 파일 (`ThreeJS_korean_bus.html`)**: `G()`, `bx()`, `cyl()`, `sph()`, `tube()` 함수를 정의하고, `part()` 래퍼(Wrapper) 함수를 통해 각 부품을 8단계 공정(`stage: 0~7`) 및 분해 좌표(`explode`)와 매핑하여 관리합니다.

### ③ 동적 캔버스 텍스처 (Procedural Canvas Texture)
이미지 파일 대신 자바스크립트로 `<canvas>` 요소에 2D 그래픽과 텍스트를 그린 뒤, `THREE.CanvasTexture`로 변환하여 3D 재질(`map`)에 입히는 기법을 사용합니다.
* **교실 파일**: 나무 바닥 결(`floorTex`), 벽면 질감(`wallTex`), 실시간 데이터가 갱신되는 전자칠판(`boardTex`), 3D 공간 내 센서 라벨(`THREE.Sprite`)에 활용됩니다.
* **버스 파일**: 공장 안전 표지판 및 버스 행선지 전광판(`701 서울 시내버스`, `친환경 전기버스`) 생성에 사용됩니다.

### ④ 렌더링 최적화 및 카메라 제어 문법
* **렌더러 설정 (`THREE.WebGLRenderer`)**:
  * `preserveDrawingBuffer: true`: 사용자가 3D 화면을 PNG 이미지로 다운로드(`toBlob` / `toDataURL`)할 수 있도록 버퍼를 유지합니다.
  * `toneMapping: THREE.ACESFilmicToneMapping`: 물리 기반 조명(PBR)의 명암을 자연스럽게 보정합니다.
* **카메라 트위닝 (Camera Tweening)**:
  * 시점 전환 버튼 클릭 시 카메라가 순간 이동하지 않고 부드럽게 이동하도록 `Vector3.lerpVectors()`(교실) 또는 `Vector3.lerp()`(버스) 보간법을 애니메이션 루프 내에서 실행합니다.
* **벽면 자동 절개 (교실 파일 전용)**:
  * `updateWallCutaway()` 함수가 매 프레임 카메라 위치를 감시하여, 사용자가 교실 밖으로 시점을 회전할 때 시야를 가리는 앞벽(`frontWallOccluders`)과 창가 벽(`windows`)의 `visible` 속성을 자동으로 `false`로 전환합니다.
* **레이캐스팅 (버스 파일 전용)**:
  * `THREE.Raycaster`와 `pointerdown`/`pointerup` 이벤트를 결합하여, 드래그가 아닌 단순 클릭(이동 거리 $6\text{px}$ 이하)일 때만 마우스 좌표의 3D 부품을 판별(`intersectObjects`)해 상세 설명을 표시합니다.

### ⑤ 방어적 예외 처리 (Fallback & Context Loss)
네트워크 단절이나 GPU 메모리 부족으로 3D 렌더링이 중단될 때 빈 화면이 나오는 것을 방지하는 2중 안전장치가 구현되어 있습니다.
1. **로딩 타임아웃 감지**: 일반 `<script>` 블록에서 `setTimeout`($12\sim 14$초)을 설정하여, 메인 모듈이 정상 실행 플래그(`__SMART_CLASSROOM_READY__` 또는 `__busReady`)를 켜지 못하면 안내 메시지와 새로고침 버튼을 표시합니다.
2. **WebGL 컨텍스트 손실 복구 (교실 파일)**: `webglcontextlost` 및 `webglcontextrestored` 이벤트를 리스닝하여 그래픽 드라이버 재시작 시 텍스처와 그림자 맵을 자동으로 재업로드합니다.