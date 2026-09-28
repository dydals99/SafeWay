# SafeWay

**살고 싶은 동네, 밤길은 안전할까?**
주소를 입력하면 주변 안전시설(CCTV·비상벨·보안등·가로등·민방위대피시설)을 기반으로 한 **안전 점수**, <br>
**근처 정류장**, 정류장까지의 **빠른 경로 / 안전 경로**를 지도 위에 보여주는 서비스입니다.

## 프로젝트 목적

이사할 집이나 자취방을 고를 때 교통·가격 정보는 쉽게 찾을 수 있지만, <br>
"퇴근길 정류장에서 집까지 걸어가는 길이 안전한가"는 직접 가보기 전에는 알기 어렵습니다. <br>

SafeWay는 서울시 공공데이터로 흩어져 있는 안전시설 위치 정보를 한 곳에 모아 <br>

- 특정 위치 주변이 얼마나 안전한지를 **0~100 점수**로 정량화하고, <br>
- 가장 가까운 길이 아니라 **안전시설이 많은 길로 돌아가는 경로**를 함께 제시해,

사용자가 거주지를 고르기 전에 주변의 야간 보행 안전성을 판단할 수 있도록 돕는 것을 목표로 합니다.

## 구현 화면

<!-- 실행 화면 GIF 추가 예정 -->

## 주요 성과

### 1. 서울시 공공 안전시설 데이터 통합 적재

CCTV, 가로등, 보안등, 안전비상벨, 민방위대피시설, 버스정류소, 지하철역 7종의 공공데이터를 PostgreSQL/PostGIS에 정규화해 적재했습니다. <br>
보안등은 자치구 25개 파일의 인코딩(UTF-8/CP949)과 컬럼 구성이 제각각이라 자동 감지 후 공통 스키마로 병합합니다.
  → [pipeline README](https://github.com/dydals99/pipeline#security_light-관련-특이사항)

### 2. 좌표 결측 데이터 지오코딩 백필

보안등 201,958건 중 44,565건(강남구 전체 등)은 원본에 위도/경도가 없어 지도와 점수 계산에서 빠지는 문제가 있었습니다. <br>
주소(도로명 → 지번 순)를 카카오 로컬 API로 좌표로 변환해 채우고, 좌표 출처(`raw`/`geocoded`/`missing`)를 컬럼으로 추적하며, <br>
결과를 캐시해 재실행 시 API를 다시 호출하지 않도록 했습니다. <br>
그 결과 결측 44,565건 중 **43,582건(97.8%)을 복구**해 보안등 좌표 보유율을 **77.9% → 99.5**%로 높였습니다. <br>
  → [좌표 결측 처리 (지오코딩 백필)](https://github.com/dydals99/pipeline#좌표-결측-처리-지오코딩-백필)

### 3. 안전 가중치 기반 경로 탐색

OSM 서울 보행 도로망(엣지 약 47만 개)을 pgRouting 그래프로 구축하고, 엣지마다 주변 안전시설 밀도가 낮을수록 커지는
`cost_safety` 비용을 계산해 최단 경로와 별도로 **안전 경로**를 탐색합니다.
  → [back-end 경로 탐색](https://github.com/dydals99/back-end#주요-기능)

### 4. 검색 API 성능 최적화

데이터가 쌓이면서 느려진 검색 API를 분석해 공간 인덱스 미사용(Seq Scan), 정류장별 N+1 쿼리, <br>
경로 탐색 시 전체 그래프 로딩 문제를 해결했습니다. 개선 전후 응답 결과가 동일함을 검증했습니다.

| API | 개선 전 | 개선 후 |
|---|---:|---:|
| 안전 점수 / 주변 시설 | 211~300 ms | 8~18 ms |
| 근처 정류장 + 정류장별 안전 점수 | 5.3~6.9 s | 46~99 ms (**최대 약 115배**) |
| 경로 탐색 (1~2km) | 1.4~1.5 s | 53~60 ms |

  → [성능 최적화 상세 (문제 분석 · 개선 내용 · 전체 벤치마크)](https://github.com/dydals99/back-end#성능-최적화)

## 기술 스택

| 영역 | 기술 |
|---|---|
| Front-end | Next.js, React, TypeScript, TanStack Query, Kakao Maps SDK |
| Back-end | NestJS, TypeORM, TypeScript |
| Database | PostgreSQL, PostGIS, pgRouting |
| Data Pipeline | Python, pandas, osmnx, Kakao Local API |

## 저장소 구성

이 저장소는 세 개의 서브모듈로 구성되어 있습니다. 실행 방법과 상세 내용은 각 저장소의 README를 참고해주세요.

| 디렉터리 | 역할 |
|---|---|
| [`pipeline/`](https://github.com/dydals99/pipeline) | 공공데이터 전처리, 지오코딩 백필, DB 적재, 보행 도로 그래프 구축 |
| [`back-end/`](https://github.com/dydals99/back-end) | 지오코딩, 안전 점수, 근처 정류장, 경로 탐색 API (NestJS) |
| [`front-end/`](https://github.com/dydals99/front-end) | 주소 검색 및 지도 결과 화면 (Next.js) |

```bash
git clone --recurse-submodules https://github.com/dydals99/SafeWay.git
```
