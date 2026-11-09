# 8주차 워크플로우 실행 최적화 리포트

- 레포: https://github.com/pckdou/workflow
- 대상: `frontend`, `backend` 샘플 Node 프로젝트
- 적용 기법: Matrix, Composite Action, Reusable Workflow, npm 캐시, 브랜치/PR 조건, 변경 파일 감지(paths-filter)

---

## 1. 구조 개선 (중복 제거)

| 파일 | 역할 |
|---|---|
| `.github/workflows/01-baseline.yml` | 최적화 전 (frontend/backend job 중복 작성) |
| `.github/actions/setup-node-deps/action.yml` | Composite Action (Node 설치 + 캐시 + `npm ci`) |
| `.github/workflows/_reusable-test.yml` | Reusable Workflow + Matrix |
| `.github/workflows/ci.yml` | 최적화 후 (Reusable Workflow 호출) |
| `.github/workflows/deploy.yml` | 조건부 배포 |
| `.github/workflows/02-cache-benchmark.yml` | 캐시 효과 측정용 격리 워크플로우 |

| 항목 | Before | After |
|---|---|---|
| 워크플로우 YAML 줄 수 | 29줄 (`01-baseline.yml`) | 87줄 (`ci.yml` + `_reusable-test.yml` + `action.yml`) |
| 중복된 job 정의 | 2개 (frontend, backend 복붙) | 0개 (Reusable Workflow 1개를 호출) |
| 테스트 환경 | Ubuntu + Node 20 (1조합) | Ubuntu 3개 + Windows 2개 = 5조합 × 2프로젝트 = 10 job |
| 새 프로젝트 추가 시 | job 약 12줄 복사 | 호출 job 약 4줄 추가 |

줄 수 계산 명령은 다음과 같습니다.

```powershell
(Get-Content .github/workflows/01-baseline.yml).Count
(Get-Content .github/workflows/ci.yml).Count + (Get-Content .github/workflows/_reusable-test.yml).Count + (Get-Content .github/actions/setup-node-deps/action.yml).Count
```

**해석:** After의 총 줄 수는 Matrix와 재사용 구조가 추가되어 Before보다 많을 수 있다. 개선의 핵심은 줄 수 감소가 아니라 **중복 제거와 테스트 범위 확대(1조합 → 5조합)**, 그리고 **수정 지점의 단일화**다. 설치 로직을 바꿀 때 Composite Action 한 곳만 고치면 된다.

증빙: [PR #1 Checks (Matrix 10개 job)](https://github.com/pckdou/workflow/pull/1/checks)

---

## 2. 캐싱 전후 측정

### 2-1. 측정 방법
- 같은 워크플로우를 `use-cache` 입력으로 켜고 끄며 비교했다. (Composite Action의 `use-cache` 토글)
- 캐시를 처음 만드는 run(miss)은 측정에서 제외했다.
- 지표는 **Total**(run 전체 소요 시간)과 **Dep**(`setup-node-deps` 단계 시간, 캐시 복원 + `npm ci`)을 사용했다.
- 캐시 동작 확인: 모든 조합에서 로그에 `Cache restored successfully`와 `Cache hit occurred on the primary key ..., not saving cache`가 출력됨.

### 2-2. 1차 측정: 5조합 CI (소규모 의존성)

| 조건 | 1회 | 2회 | 3회 | 평균 |
|---|---|---|---|---|
| Baseline | 30 | 78 | 29 | 45.7초 |
| CI 캐시 OFF | 51 | 60 | 48 | 53.0초 |
| CI 캐시 ON | 76 | 71 | 80 | 75.7초 |

**결과:** 캐시 ON이 오히려 더 느리게 나왔다.
**원인 분석:**
- 같은 조합의 의존성 단계가 5초와 39초, 17초와 71초로 3~10배씩 요동쳤다. 러너/네트워크 편차가 캐시 효과보다 훨씬 컸다.
- 전체 시간은 가장 느린 job(Windows 러너)이 결정한다.
- 로그 타임스탬프상 캐시 복원은 약 1초, `npm ci`는 3~4초로, 캐시로 줄일 수 있는 시간 자체가 몇 초뿐이었다.
- 이 편차 요인은 로그와 단계별 시간으로 확인한 부분과 추정한 부분이 섞여 있다. 90초급 이상치의 정확한 원인은 로그로 확정하지 못했다.

**조치:** 편차를 줄이기 위해 Ubuntu 1개, Node 20 단일 조건의 격리 워크플로우(`02-cache-benchmark.yml`)를 만들어 재측정했다.

### 2-3. 2차 측정: 격리 벤치마크 (소규모 의존성, 각 5회)

| 지표 | 캐시 OFF | 캐시 ON | 개선율 |
|---|---|---|---|
| 평균 | 32.8초 | 47.8초 | -45.7% (이상치 2건 영향) |
| 중앙값 | 34초 | 32초 | 5.9% |
| 이상치(90초, 58초) 제외 평균 | 32.8초 | 30.3초 | 7.6% |

- OFF: 34, 30, 35, 29, 36 / ON: 32, 90, 29, 58, 30
- 평균은 ON의 이상치 2건이 왜곡했으므로 중앙값을 기준으로 해석한다.

### 2-4. 3차 측정: 의존성 확장 후 (최종 결과)

의존성이 작아 캐시 이득이 작다고 판단해 패키지를 확장했다.
(frontend: next, @mui/material, antd, vite, sass 등 / backend: mongoose, aws-sdk, rxjs, typescript 등. 로컬 `npm ci` 기준 frontend 약 3분, backend 약 48초, 패키지 수 961개 / 537개)

OFF/ON을 번갈아 5쌍 실행했다.

| 지표 | 캐시 OFF | 캐시 ON | 개선율 |
|---|---|---|---|
| **Dep 중앙값** | 29.0초 | 25.0초 | **13.8%** |
| Dep 평균 | 25.8초 | 23.4초 | 9.3% |
| Total 중앙값 | 38.0초 | 34.0초 | 10.5% |
| Total 평균 | 35.4초 | 32.8초 | 7.3% |

- Dep OFF: 23, 30, 18, 29, 29 / ON: 22, 17, 25, 28, 25
- Total OFF: 32, 41, 28, 38, 38 / ON: 30, 25, 34, 40, 35

### 2-5. 종합

| 구분 | 의존성 | 캐시 OFF | 캐시 ON | 개선율 (중앙값) |
|---|---|---|---|---|
| 2차 Total | 소규모 | 34초 | 32초 | 5.9% |
| 3차 Total | 확장 | 38.0초 | 34.0초 | 10.5% |
| 3차 Dep | 확장 | 29.0초 | 25.0초 | 13.8% |

- 의존성이 커지자 개선율이 5.9%에서 10.5%(전체), 13.8%(의존성 단계)로 커지는 **경향**이 확인된다.
- **한계:** 표본이 각 5회뿐이고 값 범위가 겹친다(Dep OFF 18~30초, ON 17~28초). 개선 "경향"은 보이지만 통계적으로 확정된 결과라고 단정하지 않는다.
- **효과가 작은 이유:** 캐시는 패키지 다운로드만 줄인다. `npm ci`는 매번 `node_modules`를 지우고 다시 만들기 때문에 그 시간은 그대로이고, GitHub 러너의 npm 레지스트리 접속이 빨라 다운로드 시간이 원래 짧다.

### 2-6. Baseline 재측정 (의존성 확장 후)

| 회차 | 1 | 2 | 3 | 평균 | 중앙값 |
|---|---|---|---|---|---|
| Baseline | 26초 | 31초 | 33초 | 30.0초 | 31초 |

Baseline(Ubuntu 1개, 두 프로젝트 병렬 job)과 벤치마크(한 job에서 순차 설치)는 조건이 달라 전체 시간을 직접 비교하지 않는다. 구조 개선의 비교 지표는 1장(중복 제거, 테스트 조합 수)이다.

### 2-7. 증빙 링크 (모두 Actions run)

| 측정 | run |
|---|---|
| Baseline 1차 | [42566119190](https://github.com/pckdou/workflow/actions/runs/42566119190), [42566689254](https://github.com/pckdou/workflow/actions/runs/42566689254), [42566836070](https://github.com/pckdou/workflow/actions/runs/42566836070) |
| CI 캐시 OFF 1차 | [42566896696](https://github.com/pckdou/workflow/actions/runs/42566896696), [42566993141](https://github.com/pckdou/workflow/actions/runs/42566993141), [42567114111](https://github.com/pckdou/workflow/actions/runs/42567114111) |
| CI 캐시 ON 1차 | [42567200156](https://github.com/pckdou/workflow/actions/runs/42567200156), [42567337802](https://github.com/pckdou/workflow/actions/runs/42567337802), [42567459455](https://github.com/pckdou/workflow/actions/runs/42567459455) |
| 벤치마크 OFF (소규모) | [42568441729](https://github.com/pckdou/workflow/actions/runs/42568441729), [42568508140](https://github.com/pckdou/workflow/actions/runs/42568508140), [42568568872](https://github.com/pckdou/workflow/actions/runs/42568568872), [42568631501](https://github.com/pckdou/workflow/actions/runs/42568631501), [42568688257](https://github.com/pckdou/workflow/actions/runs/42568688257) |
| 벤치마크 ON (소규모) | [42568756341](https://github.com/pckdou/workflow/actions/runs/42568756341), [42568820859](https://github.com/pckdou/workflow/actions/runs/42568820859), [42568966632](https://github.com/pckdou/workflow/actions/runs/42568966632), [42569023612](https://github.com/pckdou/workflow/actions/runs/42569023612), [42569125966](https://github.com/pckdou/workflow/actions/runs/42569125966) |
| 벤치마크 OFF (확장) | [42598474922](https://github.com/pckdou/workflow/actions/runs/42598474922), [42598647290](https://github.com/pckdou/workflow/actions/runs/42598647290), [42598816831](https://github.com/pckdou/workflow/actions/runs/42598816831), [42598982973](https://github.com/pckdou/workflow/actions/runs/42598982973), [42599171928](https://github.com/pckdou/workflow/actions/runs/42599171928) |
| 벤치마크 ON (확장) | [42598562131](https://github.com/pckdou/workflow/actions/runs/42598562131), [42598753747](https://github.com/pckdou/workflow/actions/runs/42598753747), [42598895669](https://github.com/pckdou/workflow/actions/runs/42598895669), [42599069934](https://github.com/pckdou/workflow/actions/runs/42599069934), [42599257287](https://github.com/pckdou/workflow/actions/runs/42599257287) |
| Baseline (확장 후) | [42599702682](https://github.com/pckdou/workflow/actions/runs/42599702682), [42599772714](https://github.com/pckdou/workflow/actions/runs/42599772714), [42599853251](https://github.com/pckdou/workflow/actions/runs/42599853251) |

캐시 hit 로그는 1차 ON run [42567459455](https://github.com/pckdou/workflow/actions/runs/42567459455)의 `Run actions/setup-node@v4` 단계에서 `Cache restored successfully`로 확인할 수 있다.

---

## 3. 조건부 배포 파이프라인

### 3-1. 규칙 (`deploy.yml`)
- `main` push는 `production`, `develop` push는 `staging` 환경으로 배포
- PR 이벤트에서는 배포하지 않음 (`github.event_name == 'push'` 조건)
- `dorny/paths-filter`로 변경된 폴더(`frontend/**`, `backend/**`)만 배포
- `ci.yml`은 `.md`, `docs/**`만 바뀐 경우 실행하지 않음 (`paths-ignore`)

### 3-2. 검증 시나리오

| # | 시나리오 | 이벤트 | 기대 결과 | 실제 결과 | 증빙 |
|---|---|---|---|---|---|
| 1 | frontend만 수정 | PR → develop | 배포 job 모두 skipped | 일치 | [PR #1](https://github.com/pckdou/workflow/pull/1/checks) |
| 2 | frontend만 수정한 PR merge | push develop | frontend만 staging 배포 | 일치 | [run](https://github.com/pckdou/workflow/actions/runs/42602686154) |
| 3 | backend만 수정 (수정 전) | push develop | backend만 배포 | **불일치** (둘 다 배포) | [run](https://github.com/pckdou/workflow/actions/runs/42603503732) |
| 3 | backend만 수정 (수정 후) | push develop | backend만 배포 | 일치 | [PR #3](https://github.com/pckdou/workflow/pull/3), [run](https://github.com/pckdou/workflow/actions/runs/42604759724) |
| 4 | docs만 수정 | PR/push develop | CI 미실행, 배포 skipped | 일치 | [PR #4](https://github.com/pckdou/workflow/pull/4/checks), [run](https://github.com/pckdou/workflow/actions/runs/42605150528) |
| 5 | develop → main | push main | frontend, backend 모두 production 배포 | 일치 | [PR #5](https://github.com/pckdou/workflow/pull/5), [run](https://github.com/pckdou/workflow/actions/runs/42605604902) |

- 시나리오 1: PR의 Checks는 13개(Selective Deploy 3 + CI matrix 10)였고, 배포 job은 skipped였다.
- 시나리오 4: PR의 Checks가 3개뿐이었다(앞선 PR은 13개). `CI (after)`가 실행되지 않았다는 증거다.
- 시나리오 5: 변경 폴더가 둘 다 감지되어 두 job이 모두 실행됐다. 의도한 동작이다.

### 3-3. 문제 발견과 수정 (시나리오 3)

**문제:** backend만 수정했는데 `deploy-frontend`까지 실행됐다.

**원인:** `changes` job 로그에 `Changes will be detected between main and develop`가 출력됐다. `paths-filter`에 `base`를 지정하지 않으면 push 이벤트에서도 기본 브랜치(main)와 현재 브랜치의 **누적 차이**를 비교한다. 시나리오 2에서 develop에 들어간 frontend 변경이 아직 main에 없어서 계속 감지된 것이다. (로그: `Detected 2 changed files`, `Filter frontend = true`, `Filter backend = true`)

**수정:** `deploy.yml`에 push 직전 커밋을 비교 기준으로 지정했다.

```yaml
- uses: dorny/paths-filter@v3
  id: filter
  with:
    base: ${{ github.event.before }}
    filters: |
      frontend:
        - 'frontend/**'
      backend:
        - 'backend/**'
```

**검증:** 수정 후 같은 방식으로 backend만 수정하자 `deploy-backend`만 실행되고 `deploy-frontend`는 skipped였다 (시나리오 3 수정 후 run). 시나리오 4도 이번 push의 변경분(`docs/`)만 비교해 배포가 skipped되었다.

### 3-4. 참고 사항
- 시나리오 1의 PR을 처음 만들었을 때 Checks가 표시되지 않아, 빈 변경 커밋(`chore: retrigger PR checks`)을 push해 이벤트를 다시 발생시켰다. 원인은 확정하지 못했고, 이후 PR #2~#5에서는 재현되지 않았다.
- 배포는 실제 배포 대신 `echo`로 시뮬레이션했다.

---

## 4. 결론

1. **구조:** Composite Action과 Reusable Workflow로 중복 job을 제거했고, Matrix로 테스트 범위를 1조합에서 5조합(2프로젝트 기준 10 job)으로 넓혔다.
2. **캐싱:** 의존성이 큰 환경에서 의존성 단계 중앙값 기준 **13.8%**, 전체 시간 중앙값 기준 **10.5%** 단축되었다. 의존성 규모가 작을 때는 효과가 5.9% 수준이고 러너 편차에 묻혔다. 표본이 작아 경향 수준으로 해석한다.
3. **측정 방법:** 처음의 5조합 측정은 러너 편차가 커서 결론을 내릴 수 없었다. 조건을 단일화한 벤치마크, 번갈아 실행, 평균 대신 중앙값 사용으로 신뢰도를 높였다.
4. **조건부 배포:** 변경 파일 감지의 기본 비교 기준(main과의 누적 차이)이 의도와 다르다는 점을 로그로 발견하고 `base` 지정으로 해결했다.
5. **한계:** 캐시 이득은 프로젝트 규모와 네트워크 상태에 크게 좌우되며, 이번 실험은 표본 수가 적다.