# Bitbucket Pipelines 정리

> Atlassian **Bitbucket Cloud**에 내장된 CI/CD 서비스인 **Bitbucket Pipelines**를 분석·정리한 문서입니다.
> 최종 정리일: 2026-08-27 · 출처: [Bitbucket Pipelines 설정 레퍼런스](https://support.atlassian.com/bitbucket-cloud/docs/bitbucket-pipelines-configuration-reference/) 외

## 목차

1. [개요](#1-개요)
2. [동작 방식·핵심 개념](#2-동작-방식핵심-개념)
3. [`bitbucket-pipelines.yml` 기본 구조](#3-bitbucket-pipelinesyml-기본-구조)
4. [파이프라인 종류 (실행 조건)](#4-파이프라인-종류-실행-조건)
5. [step 주요 키](#5-step-주요-키)
6. [definitions (caches · services)](#6-definitions-caches--services)
7. [artifacts (스텝 간 파일 전달)](#7-artifacts-스텝-간-파일-전달)
8. [parallel · stage (병렬·단계)](#8-parallel--stage-병렬단계)
9. [변수·시크릿·배포 환경](#9-변수시크릿배포-환경)
10. [Pipes (재사용 통합)](#10-pipes-재사용-통합)
11. [Runners (self-hosted)](#11-runners-self-hosted)
12. [요금(빌드 분)·팁](#12-요금빌드-분팁)

---

## 1. 개요

**Bitbucket Pipelines**는 Bitbucket Cloud에 통합된 **CI/CD(지속적 통합·배포) 서비스**입니다. 저장소 루트의 **`bitbucket-pipelines.yml`** 한 파일에 빌드·테스트·배포 과정을 선언하면, 커밋/PR/스케줄 등 이벤트에 맞춰 자동 실행됩니다.

- **별도 CI 서버 불필요** — Bitbucket이 클라우드에서 실행
- **Docker 컨테이너 기반** — 각 스텝이 지정한 이미지의 컨테이너에서 실행
- **YAML 선언형** 구성, 저장소에 함께 버전 관리
- 과금은 **빌드 분(build minutes)** 기준

> GitHub Actions·GitLab CI/CD와 같은 범주의 도구이며, Bitbucket에 네이티브로 붙어 있는 것이 특징입니다.

---

## 2. 동작 방식·핵심 개념

| 개념 | 설명 |
| --- | --- |
| **Pipeline(파이프라인)** | 하나의 실행 단위. 여러 step으로 구성 |
| **Step(스텝)** | 파이프라인의 각 단계. **독립된 Docker 컨테이너**에서 실행, 기본 순차 진행 |
| **Image(이미지)** | 스텝이 실행되는 Docker 이미지 (기본 Atlassian 이미지, 커스텀 지정 가능) |
| **Script(스크립트)** | 스텝에서 실행할 셸 명령 목록 |
| **Cache(캐시)** | 의존성 등을 **실행 간 재사용**해 빌드 속도↑ |
| **Artifact(아티팩트)** | **같은 파이프라인 내 스텝 간** 파일 전달 |
| **Service(서비스)** | DB·큐 등을 **사이드카 컨테이너**로 함께 띄움 |
| **Deployment(배포 환경)** | test/staging/production 등 환경별 시크릿·이력 관리 |

> 각 스텝은 **격리된 컨테이너**라서, 스텝 간에 상태를 넘기려면 **artifacts**(파일)나 **cache**(재사용)를 씁니다.

---

## 3. `bitbucket-pipelines.yml` 기본 구조

저장소 **루트**에 위치하며, `pipelines:` 아래에 실행 조건별 파이프라인을 정의합니다.

```yaml
image: node:20                      # 전역 기본 Docker 이미지

pipelines:
  default:                          # 아래 조건에 안 걸리는 모든 push에서 실행
    - step:
        name: Build and Test
        caches:
          - node
        script:
          - npm install
          - npm test

  branches:                         # 특정 브랜치 push 시
    main:
      - step:
          name: Deploy to Production
          deployment: production
          script:
            - ./deploy.sh
```

- **`image`** (전역) — 모든 스텝의 기본 이미지. 스텝별로 재정의 가능
- **`pipelines`** — 실행 조건별 파이프라인 묶음
- **`definitions`** — 재사용할 cache·service 정의 (아래 6장)
- **`options`** — 전역 옵션(예: `max-time`, `docker`)
- **`clone`** — 클론 동작 제어(`depth`, `lfs`, `enabled` 등)

---

## 4. 파이프라인 종류 (실행 조건)

`pipelines:` 아래에 **어떤 이벤트에 무엇을 실행할지** 정의합니다.

| 키 | 실행 시점 |
| --- | --- |
| `default` | 다른 조건에 매칭되지 않는 **모든 push** |
| `branches:` | 특정 **브랜치**에 push (글롭 패턴 가능, 예: `feature/*`) |
| `tags:` | **태그** push 시 (`bookmarks:`는 Mercurial용) |
| `pull-requests:` | **PR**이 열려 있을 때 (대상 브랜치별 패턴 가능) |
| `custom:` | **수동 트리거** 전용 (UI/스케줄로만 실행, 자동 아님) |

```yaml
pipelines:
  default:
    - step: { script: ["echo default"] }

  branches:
    main:
      - step: { script: ["./deploy-prod.sh"] }
    'feature/*':
      - step: { script: ["npm test"] }

  tags:
    'v*':
      - step: { script: ["./release.sh"] }

  pull-requests:
    '**':                            # 모든 PR
      - step: { script: ["npm run lint && npm test"] }

  custom:                            # 수동/스케줄 전용
    nightly-security-scan:
      - step: { script: ["./scan.sh"] }
```

> **Scheduled pipelines(스케줄 실행)** — YAML이 아니라 **Bitbucket UI의 Pipelines → Schedules**에서 설정하며, 보통 `custom:` 파이프라인을 주기 실행합니다(cron 유사).

---

## 5. step 주요 키

| 키 | 설명 |
| --- | --- |
| `name` | 스텝 이름(UI 표시) |
| `image` | 이 스텝의 Docker 이미지(전역 재정의) |
| `script` | 실행할 셸 명령 목록 (**필수**) |
| `caches` | 사용할 캐시 목록 (`node`, `pip`, 커스텀 등) |
| `services` | 함께 띄울 서비스(사이드카) 목록 |
| `artifacts` | 이 스텝이 만들어 **다음 스텝에 넘길** 파일 글롭 |
| `after-script` | 스텝 성공/실패와 무관하게 마지막에 실행(정리·알림) |
| `size` | 컨테이너 크기 `1x`(기본)/`2x`/`4x`/`8x` (메모리·빌드분 배수) |
| `max-time` | 최대 실행 시간(분) |
| `trigger` | `manual`이면 **수동 승인** 후 진행 (게이트) |
| `deployment` | 이 스텝이 배포하는 환경(`test`/`staging`/`production` 등) |
| `condition` | 변경된 파일 경로 등 **조건부 실행**(`changesets`) |
| `runs-on` | **self-hosted runner** 라벨 지정 |

```yaml
- step:
    name: Deploy staging
    image: node:20
    size: 2x
    max-time: 30
    deployment: staging
    trigger: manual              # 수동 승인 게이트
    caches: [node]
    condition:
      changesets:
        includePaths: ["src/**"]
    script:
      - npm ci
      - ./deploy.sh
    after-script:
      - echo "done (성공/실패 무관 실행)"
```

---

## 6. definitions (caches · services)

`definitions:`에 **재사용 가능한 cache·service**를 정의하고, 각 스텝에서 이름으로 참조합니다.

```yaml
definitions:
  caches:
    my-gradle: ~/.gradle/caches      # 커스텀 캐시(디렉토리 지정)
  services:
    postgres:
      image: postgres:16
      variables:
        POSTGRES_DB: test
        POSTGRES_PASSWORD: secret

pipelines:
  default:
    - step:
        caches: [my-gradle]
        services: [postgres]         # 사이드카로 postgres 기동
        script:
          - ./gradlew test
```

- **내장 캐시**: `node`, `pip`, `composer`, `gradle`, `maven`, `docker` 등 — 이름만 쓰면 됨
- **커스텀 캐시**: 임의 디렉토리 지정. 재빌드 시 **50%+ 시간 단축**도 흔함
- **서비스**: DB(postgres/mysql)·redis 등을 사이드카 컨테이너로. `docker` 서비스로 DinD도 가능
- 캐시는 **실행 간 지속**, 아티팩트는 **한 실행 내 스텝 간 전달**(아래 7장)

---

## 7. artifacts (스텝 간 파일 전달)

한 스텝이 만든 산출물(빌드 결과 등)을 **같은 파이프라인의 다음 스텝**이 쓰게 합니다.

```yaml
pipelines:
  default:
    - step:
        name: Build
        script:
          - npm ci && npm run build
        artifacts:
          - dist/**                  # 다음 스텝으로 전달
    - step:
        name: Deploy
        script:
          - ./upload.sh dist/        # 이전 스텝의 dist/ 사용 가능
```

> **캐시 vs 아티팩트**: 캐시 = "실행 간 재사용(속도)", 아티팩트 = "한 실행 내 스텝 간 파일 이동(데이터)".

---

## 8. parallel · stage (병렬·단계)

**`parallel`** — 여러 스텝을 **동시에** 실행(테스트 분할 등)해 총 시간 단축.

```yaml
pipelines:
  default:
    - parallel:
        - step: { name: Unit,        script: ["npm run test:unit"] }
        - step: { name: Integration, script: ["npm run test:int"] }
    - step:
        name: Deploy
        script: ["./deploy.sh"]
```

**`stage`** — 여러 스텝을 논리 단계로 묶고, 단계 단위 배포 환경·수동 게이트를 적용(대규모 파이프라인 조직화).

---

## 9. 변수·시크릿·배포 환경

### 변수(Variables)

- **저장소/워크스페이스/배포 환경 변수** — Bitbucket UI에서 설정, 파이프라인 실행 시 주입
- **Secured(보안) 변수** — 값이 마스킹되어 로그에 노출 안 됨(토큰·비밀번호)
- **기본 제공 변수** — `BITBUCKET_BRANCH`, `BITBUCKET_COMMIT`, `BITBUCKET_BUILD_NUMBER`, `BITBUCKET_REPO_SLUG` 등

```yaml
- step:
    script:
      - echo "브랜치: $BITBUCKET_BRANCH, 빌드번호: $BITBUCKET_BUILD_NUMBER"
      - curl -H "Authorization: Bearer $DEPLOY_TOKEN" ...   # 보안 변수
```

### 배포 환경(Deployments)

`deployment: <환경>`으로 스텝을 특정 환경에 묶으면, **환경별 변수·시크릿 분리**와 **배포 이력/대시보드**를 제공합니다. 보통 `test → staging → production` 단계로 구성하고 production에 **수동 승인(`trigger: manual`)** 게이트를 둡니다.

---

## 10. Pipes (재사용 통합)

**Pipes**는 자주 쓰는 작업(배포·알림·스캔 등)을 **한 줄로 재사용**하는 사전 제작 통합입니다.

```yaml
- step:
    script:
      - pipe: atlassian/aws-s3-deploy:1.1.0
        variables:
          AWS_ACCESS_KEY_ID: $AWS_KEY
          AWS_SECRET_ACCESS_KEY: $AWS_SECRET
          S3_BUCKET: my-bucket
          LOCAL_PATH: dist
```

- Atlassian·서드파티가 제공하는 다양한 pipe(Kubernetes, AWS/GCP/Azure 배포, Slack 알림 등)
- 직접 pipe를 만들어 배포·공유도 가능

---

## 11. Runners (self-hosted)

기본은 Atlassian 클라우드에서 실행되지만, **self-hosted runner**로 **자체 인프라(사내 서버·특정 OS·GPU 등)** 에서 스텝을 돌릴 수 있습니다.

```yaml
- step:
    runs-on:
      - self.hosted
      - linux
    script:
      - ./build-on-our-infra.sh
```

- 사내 네트워크 리소스 접근, 특수 하드웨어, 규정 준수 격리 등에 활용
- Linux/Windows/macOS runner 지원

---

## 12. 요금(빌드 분)·팁

- **과금 단위**: 빌드 분(build minutes). `size: 2x`는 2배 소비. self-hosted runner는 클라우드 분을 쓰지 않음
- **속도 팁**:
  - **캐시 적극 활용**(node/pip/gradle 등) → 의존성 재설치 시간 절감
  - **parallel**로 테스트 분할
  - 무거운 스텝은 **condition(changesets)** 으로 변경 있을 때만 실행
  - 필요한 스텝만 실행되도록 브랜치/PR 파이프라인 분리
- **보안**: 시크릿은 **Secured 변수**로, production은 **배포 환경 + 수동 게이트**로 보호

---

> 본 문서는 Bitbucket Pipelines 공식 문서 구조와 확립된 사용법을 기반으로 정리했습니다. 키워드·기본 이미지·요금은 버전/플랜에 따라 다를 수 있으니 최신 정보는 공식 문서에서 확인하세요.
> 공식: [설정 레퍼런스](https://support.atlassian.com/bitbucket-cloud/docs/bitbucket-pipelines-configuration-reference/) · [시작하기](https://support.atlassian.com/bitbucket-cloud/docs/get-started-with-bitbucket-pipelines/) · [캐시/서비스 정의](https://support.atlassian.com/bitbucket-cloud/docs/cache-and-service-container-definitions/) · [실행 조건](https://support.atlassian.com/bitbucket-cloud/docs/pipeline-start-conditions/)
