# Oracle Autonomous Database Select AI Agent Hands-on Lab

> **Description:** Hands-on guide for building a financial audit agent using Oracle Autonomous Database Select AI.

이 리포지토리는 Oracle Autonomous Database의 **Select AI**와 **AI Agent** 기능을 활용하여 '재무 감사 AI 에이전트'를 직접 구축해 보는 단계별 실습(Hands-on) 가이드입니다. 

자연어로 데이터베이스를 조회하는 Text-to-SQL부터, 사내 규정 PDF 문서를 참조하는 RAG, 그리고 비정상 거래 동결 및 이메일 자동 통보까지 이어지는 통합 시나리오를 경험할 수 있습니다.

---

## 1. 실습 목표

이 실습을 완료하면 다음 항목들을 이해하고 구현할 수 있습니다.
* **Text-to-SQL**: `SELECT AI`를 활용한 자연어 기반 재무 데이터 조회
* **AI Components**: AI Profile, Tool, Task, Agent, Team 구성 방법
* **RAG (Retrieval-Augmented Generation)**: OCI Object Storage 기반 사내 규정 문서 검색
* **Action Tool**: PL/SQL 함수 기반 Tool을 통한 비정상 거래 자동 동결 처리
* **Notification Tool**: OCI Email Delivery를 활용한 조치 결과 이메일 자동화

---

## 2. 필수 준비사항

* **Oracle Cloud Autonomous Database**: Select AI 및 AI Agent 기능이 지원되는 버전
* **실습용 데이터베이스 계정 (2개)**: 권한 제어용 `ADMIN` 계정과 실제 스크립트를 실행할 `WKSP_DEMO` 계정
* **OCI Credential**: Object Storage 등에 접근하기 위한 크레덴셜 (예: `OCI_KEY_CRED`) 사전 생성 완료
* **Object Storage 및 파일**: (RAG 실습용) 버킷 생성 및 사내 규정 PDF 파일 업로드 완료
* **SMTP 정보**: (이메일 실습용) OCI Email Delivery SMTP 계정 및 비밀번호
* **클라이언트 도구**: SQL Developer, SQLcl, Database Actions 중 택일

> **⚠️ 주의사항**
> GitHub 등 공개 저장소에 코드를 업로드할 때는 **절대 실제 계정, 비밀번호, OCID를 포함하지 마세요.**
> 본 문서의 코드 블록 내 `<...>` 로 표시된 부분(플레이스홀더)은 본인의 실제 환경 값으로 치환하여 실행해야 합니다.

---

## 3. 실습 흐름 요약

1. 재무 샘플 테이블 생성 및 어노테이션 추가
2. Select AI Profile 생성 및 합성 데이터(Synthetic Data) 생성
3. Select AI 기본 질의(Text-to-SQL) 테스트
4. RAG 기반 검색 Tool 구성
5. 권한, 네트워크 ACL, 유틸리티 함수 설정
6. SQL 분석 / 거래 동결 / 이메일 발송 Tool 구성
7. Task, Agent, Team 생성 및 활성화
8. 통합 시나리오 테스트

---

## 4. 단계별 실습 가이드

### Step 1. 재무 샘플 테이블 생성

**실행 계정: `WKSP_DEMO`**

프로젝트 예산, 제품 실적, 현금성 자산, 자산 거래 내역을 관리할 4개의 테이블을 생성합니다.

```sql
CREATE TABLE PROJECT_BUDGETS (
    PROJECT_ID     NUMBER PRIMARY KEY,
    PROJECT_NAME   VARCHAR2(100),
    BUDGET_YEAR    NUMBER,
    QUARTER        VARCHAR2(10),
    BUDGET_AMOUNT  NUMBER,
    ACTUAL_SPENT   NUMBER
);

CREATE TABLE PRODUCT_PERFORMANCE (
    PERF_ID        NUMBER PRIMARY KEY,
    GROUP_NAME     VARCHAR2(50),
    REPORT_MONTH   VARCHAR2(7),
    REVENUE        NUMBER,
    OPERATING_COST NUMBER
);

CREATE TABLE CASH_ASSETS (
    ASSET_ID       NUMBER PRIMARY KEY,
    BANK_NAME      VARCHAR2(50),
    ACCOUNT_TYPE   VARCHAR2(50),
    BALANCE        NUMBER,
    CURRENCY       VARCHAR2(10)
);

CREATE TABLE ASSET_TRANSACTIONS (
    TRANS_ID     NUMBER PRIMARY KEY,
    ASSET_ID     NUMBER,
    TRANS_DATE   DATE,
    TRANS_TYPE   VARCHAR2(20),
    AMOUNT       NUMBER,
    DESCRIPTION  VARCHAR2(200),
    STATUS       VARCHAR2(20) DEFAULT 'NORMAL',
    UPDATED_BY   VARCHAR2(50),
    CONSTRAINT FK_ASSET FOREIGN KEY (ASSET_ID) REFERENCES CASH_ASSETS(ASSET_ID)
);
