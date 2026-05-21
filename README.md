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
```

### Step 2. Select AI 메타데이터(어노테이션) 추가

**실행 계정: `WKSP_DEMO`**

LLM이 데이터베이스 스키마를 정확히 이해할 수 있도록 자연어 주석(Comment)과 JSON 어노테이션을 추가합니다. 한글 비즈니스 용어를 충분히 기재할수록 AI의 질의응답 정확도가 향상됩니다.

```sql
COMMENT ON TABLE PROJECT_BUDGETS IS '{"comment": "프로젝트별 예산 실적", "annotation": {"description": "각 프로젝트의 분기별 예산 편성액과 실제 집행 비용"}}';
COMMENT ON COLUMN PROJECT_BUDGETS.ACTUAL_SPENT IS '{"comment": "집행액", "annotation": {"logic": "집행액이 예산액보다 크면 예산 초과"}}';
```

### Step 3. Select AI Profile 생성

**실행 계정: `WKSP_DEMO`**

AI 공급자, 모델, 크레덴셜, 그리고 AI가 참조할 테이블 목록을 정의하여 프로파일을 생성합니다.

```sql
BEGIN
  DBMS_CLOUD_AI.CREATE_PROFILE(
    profile_name => 'OCI_AI_PROFILE',
    attributes   => '{
      "provider": "oci",
      "model": "meta.llama-3.1-405b-instruct",
      "credential_name": "OCI_KEY_CRED",
      "comments": true,
      "object_list": [
        {"owner": "WKSP_DEMO", "name": "PROJECT_BUDGETS"},
        {"owner": "WKSP_DEMO", "name": "PRODUCT_PERFORMANCE"},
        {"owner": "WKSP_DEMO", "name": "CASH_ASSETS"},
        {"owner": "WKSP_DEMO", "name": "ASSET_TRANSACTIONS"}
      ]
    }'
  );
END;
/

-- 생성된 프로파일 확인
SELECT PROFILE_NAME, STATUS, CREATED
FROM USER_CLOUD_AI_PROFILES;
```

### Step 4. 합성 데이터(Synthetic Data) 생성

**실행 계정: `WKSP_DEMO`**

LLM을 활용하여 각 테이블에 실습용 가상 데이터를 자동으로 채워 넣습니다. 프롬프트에 원하는 데이터의 특징을 지시할 수 있습니다.

```sql
EXEC DBMS_CLOUD_AI.SET_PROFILE('OCI_AI_PROFILE');

BEGIN
  DBMS_CLOUD_AI.GENERATE_SYNTHETIC_DATA(
    profile_name => 'OCI_AI_PROFILE',
    object_name  => 'PROJECT_BUDGETS',
    record_count => 10,
    user_prompt  => '2026년 Q4 데이터 포함, 예산 초과 사례 포함, 한글 프로젝트명 생성'
  );

  DBMS_CLOUD_AI.GENERATE_SYNTHETIC_DATA(
    profile_name => 'OCI_AI_PROFILE',
    object_name  => 'PRODUCT_PERFORMANCE',
    record_count => 8,
    user_prompt  => '보고월 2026-02, 한글 제품군, 다양한 이익률'
  );

  DBMS_CLOUD_AI.GENERATE_SYNTHETIC_DATA(
    profile_name => 'OCI_AI_PROFILE',
    object_name  => 'CASH_ASSETS',
    record_count => 5,
    user_prompt  => '국내 은행명 + 한글 계좌유형'
  );

  DBMS_CLOUD_AI.GENERATE_SYNTHETIC_DATA(
    profile_name => 'OCI_AI_PROFILE',
    object_name  => 'ASSET_TRANSACTIONS',
    record_count => 15,
    user_prompt  => '입금/출금 + 한글 적요(급여 지급, 법인세 납부 등)'
  );
END;
/
```

### Step 5. 자연어 질의(Text-to-SQL) 테스트

**실행 계정: `WKSP_DEMO`**

데이터베이스에 자연어로 질문을 던져 올바른 SQL이 생성되고 실행되는지 확인합니다.

```sql
EXEC DBMS_CLOUD_AI.SET_PROFILE('OCI_AI_PROFILE');

SELECT AI "현재 예산을 초과해서 사용하고 있는 프로젝트 리스트를 보여줘.";
SELECT AI "4분기 잔여 예산으로 집행 가능한 최대 금액이 얼마 남았지?";
SELECT AI narrate "지난달 제품군별 영업이익률 순위대로 정렬해줘.";
SELECT AI "현재 우리 회사의 현금성 자산 보유 현황을 계좌별로 요약해줘.";
```

### Step 6. RAG Tool 구성 (규정 PDF 검색)

**실행 계정: `WKSP_DEMO`**

Object Storage에 업로드된 문서를 기반으로 RAG(검색 증강 생성) 환경을 구성합니다. 플레이스홀더(`<...>`) 값을 본인 환경에 맞게 수정하세요.

```sql
-- 1. 임베딩 프로파일 생성
BEGIN
  DBMS_CLOUD_AI.CREATE_PROFILE(
    profile_name => 'EMBED_PROFILE',
    attributes   => '{
      "provider": "oci",
      "credential_name": "OCI_KEY_CRED",
      "embedding_model": "cohere.embed-v4.0"
    }'
  );
END;
/

-- 2. 벡터 인덱스 생성 (버킷 경로 수정 필수)
BEGIN
  DBMS_CLOUD_AI.CREATE_VECTOR_INDEX(
    index_name => 'RAG_INDEX',
    attributes => '{
      "vector_db_provider": "oracle",
      "location": "https://swiftobjectstorage.<region>[.oraclecloud.com/v1/](https://.oraclecloud.com/v1/)<namespace>/<bucket>",
      "object_storage_credential_name": "OCI_KEY_CRED",
      "profile_name": "EMBED_PROFILE",
      "vector_dimension": 1536,
      "vector_distance_metric": "cosine",
      "chunk_overlap": 128,
      "chunk_size": 1024
    }'
  );
END;
/

-- 3. RAG 텍스트 생성 프로파일 생성 (Compartment OCID 수정 필수)
BEGIN
  DBMS_CLOUD_AI.CREATE_PROFILE(
    profile_name => 'RAG_PROFILE',
    attributes   => '{
      "provider": "oci",
      "credential_name": "OCI_KEY_CRED",
      "vector_index_name": "RAG_INDEX",
      "oci_compartment_id": "<your_compartment_ocid>",
      "temperature": 0.2,
      "max_tokens": 3000
    }'
  );
END;
/

-- 4. 에이전트용 RAG 도구 생성
BEGIN
  DBMS_CLOUD_AI_AGENT.CREATE_TOOL(
    tool_name  => 'RAG_TOOL',
    attributes => '{
      "tool_type": "RAG",
      "tool_params": {"profile_name": "RAG_PROFILE"}
    }'
  );
END;
/
```

### Step 7. 권한 부여 및 유틸리티 설정

**실행 계정: `ADMIN` 및 `WKSP_DEMO`**

Agent 실행 및 이메일 발송에 필요한 권한과 네트워크 설정을 구성합니다.

```sql
-- [ADMIN 계정] AI Agent 패키지 실행 권한 부여
GRANT EXECUTE ON C##CLOUD$SERVICE.DBMS_CLOUD_AI_AGENT TO WKSP_DEMO;
GRANT EXECUTE ON C##CLOUD$SERVICE.DBMS_CLOUD_AI TO WKSP_DEMO;
GRANT EXECUTE ON DBMS_LOB TO WKSP_DEMO;
GRANT SELECT ON DBA_AI_AGENT_TASKS TO WKSP_DEMO;
GRANT SELECT ON DBA_AI_AGENT_TASK_ATTRIBUTES TO WKSP_DEMO;
GRANT SELECT ON DBA_AI_AGENT_TOOLS TO WKSP_DEMO;
GRANT SELECT ON DBA_AI_AGENT_TOOL_ATTRIBUTES TO WKSP_DEMO;

-- [ADMIN 계정] 이메일 발송을 위한 SMTP 네트워크 ACL 설정 (리전 수정 필수)
BEGIN
  DBMS_NETWORK_ACL_ADMIN.APPEND_HOST_ACE(
    host       => 'smtp.email.<region>.oci.oraclecloud.com',
    lower_port => 587,
    upper_port => 587,
    ace        => xs$ace_type(
      privilege_list => xs$name_list('connect'),
      principal_name => 'WKSP_DEMO',
      principal_type => xs_acl.ptype_db
    )
  );
END;
/

-- [WKSP_DEMO 계정] 편리한 Team 호출을 위한 유틸리티 함수 생성
CREATE OR REPLACE FUNCTION run_team_clob (
  p_team_name   IN VARCHAR2,
  p_user_prompt IN VARCHAR2,
  p_params      IN CLOB
) RETURN CLOB
AS
  PRAGMA AUTONOMOUS_TRANSACTION;
  l_answer CLOB;
BEGIN
  l_answer := DBMS_CLOUD_AI_AGENT.RUN_TEAM(
    team_name   => p_team_name,
    user_prompt => p_user_prompt,
    params      => p_params
  );
  COMMIT;
  RETURN l_answer;
END;
/
```

### Step 8. Agent 도구(Tool) 추가 구성

**실행 계정: `WKSP_DEMO`**

SQL 분석 도구, 비정상 거래 동결 처리 도구, 이메일 통보 도구를 각각 정의합니다.

```sql
-- 1. SQL 조회 도구 생성
EXEC DBMS_CLOUD_AI.SET_PROFILE('OCI_AI_PROFILE');

BEGIN
  DBMS_CLOUD_AI_AGENT.CREATE_TOOL(
    tool_name   => 'SQL_Analysis_Tool',
    attributes  => '{
      "tool_type": "SQL",
      "tool_params": {"profile_name": "OCI_AI_PROFILE"}
    }',
    description => '재무 데이터 조회/분석용 SQL 도구'
  );
END;
/

-- 2. 비정상 거래 동결 함수 및 도구 생성
CREATE OR REPLACE FUNCTION F_FREEZE_TRANSACTIONS (
  p_keyword IN VARCHAR2 DEFAULT '미등록'
) RETURN VARCHAR2 IS
  v_count NUMBER;
  v_result_msg VARCHAR2(4000);
BEGIN
  UPDATE ASSET_TRANSACTIONS
  SET STATUS = 'FROZEN',
      UPDATED_BY = 'SELECT_AI_AGENT',
      DESCRIPTION = DESCRIPTION || ' [규정 제4조 의거 동결 - ' || TO_CHAR(SYSDATE, 'YYYY-MM-DD HH24:MI') || ']'
  WHERE (DESCRIPTION LIKE '%' || p_keyword || '%')
    AND STATUS = 'NORMAL';

  v_count := SQL%ROWCOUNT;
  COMMIT;

  IF v_count > 0 THEN
    v_result_msg := '성공적으로 ' || v_count || '건 동결 처리';
  ELSE
    v_result_msg := '동결 대상 없음';
  END IF;

  RETURN v_result_msg;
EXCEPTION
  WHEN OTHERS THEN
    ROLLBACK;
    RETURN '오류: ' || SQLERRM;
END;
/

BEGIN
  DBMS_CLOUD_AI_AGENT.CREATE_TOOL(
    tool_name   => 'Freeze_Financial_Transactions',
    attributes  => '{
      "instruction": "적요 키워드 기반 비정상 거래 동결",
      "function": "F_FREEZE_TRANSACTIONS"
    }',
    description => '비정상 거래 일괄 동결 처리 도구'
  );
END;
/

-- 3. 이메일 통보 도구 생성 (SMTP 계정 정보 수정 필수)
BEGIN
  DBMS_CLOUD.CREATE_CREDENTIAL(
    credential_name => 'OCI_SMTP_CRED',
    username        => '<smtp_username>',
    password        => '<smtp_password>'
  );
END;
/

BEGIN
  DBMS_CLOUD_AI_AGENT.CREATE_TOOL(
    tool_name  => 'EMAIL_NOTIFY',
    attributes => '{
      "tool_type": "NOTIFICATION",
      "tool_params": {
        "notification_type": "EMAIL",
        "credential_name": "OCI_SMTP_CRED",
        "smtp_host": "smtp.email.<region>.oci.oraclecloud.com",
        "smtp_port": 587,
        "sender": "<sender_email>",
        "recipient": "<recipient_email>",
        "encryption": "STARTTLS"
      }
    }',
    description => '조치 결과 이메일 통보 도구'
  );
END;
/
```

### Step 9. Task 구성

**실행 계정: `WKSP_DEMO`**

Agent가 수행할 역할과 사용할 도구를 묶어 지침(Instruction)을 부여합니다.

```sql
BEGIN
  DBMS_CLOUD_AI_AGENT.DROP_TASK('Finance_Audit_Task');
EXCEPTION WHEN OTHERS THEN NULL;
END;
/

DECLARE
  v_attributes CLOB;
BEGIN
  v_attributes := JSON_OBJECT(
    'instruction' VALUE
      '당신은 데이터 기반 재무 감사 에이전트입니다. ' ||
      '1) 조회는 SQL_ANALYSIS_TOOL로 수행하고 결과를 Markdown 표로 제시하세요. ' ||
      '2) 규정 질문은 RAG_TOOL로 답변하세요. ' ||
      '3) FREEZE_FINANCIAL_TRANSACTIONS와 EMAIL_NOTIFY는 사용자가 명시적으로 요청한 경우에만 실행하세요.',
    'tools' VALUE JSON_ARRAY('SQL_ANALYSIS_TOOL', 'RAG_TOOL', 'FREEZE_FINANCIAL_TRANSACTIONS', 'EMAIL_NOTIFY'),
    'enable_human_tool' VALUE true
  );

  DBMS_CLOUD_AI_AGENT.CREATE_TASK(
    task_name  => 'Finance_Audit_Task',
    attributes => v_attributes
  );
END;
/
```

### Step 10. Agent 및 Team 활성화

**실행 계정: `WKSP_DEMO`**

생성한 Task를 담당할 Agent를 만들고, 이를 Team 단위로 묶어 활성화합니다.

```sql
-- Agent 생성
BEGIN
  BEGIN
    DBMS_CLOUD_AI_AGENT.DROP_AGENT(agent_name => 'Finance_Audit_Agent');
  EXCEPTION WHEN OTHERS THEN NULL;
  END;

  DBMS_CLOUD_AI_AGENT.CREATE_AGENT(
    agent_name => 'Finance_Audit_Agent',
    attributes => JSON_OBJECT(
      'profile_name' VALUE 'OCI_AI_PROFILE',
      'role' VALUE '당신은 기업 재무 감사 전문가입니다. 데이터와 규정 근거로 답변하세요.'
    )
  );
END;
/

-- Team 생성 및 활성화
BEGIN
  BEGIN
    DBMS_CLOUD_AI_AGENT.DROP_TEAM(team_name => 'Finance_Audit_Team');
  EXCEPTION WHEN OTHERS THEN NULL;
  END;

  DBMS_CLOUD_AI_AGENT.CREATE_TEAM(
    team_name  => 'Finance_Audit_Team',
    attributes => JSON_OBJECT(
      'agents' VALUE JSON_ARRAY(
        JSON_OBJECT('name' VALUE 'Finance_Audit_Agent', 'task' VALUE 'Finance_Audit_Task')
      ),
      'process' VALUE 'sequential'
    )
  );
END;
/

EXEC DBMS_CLOUD_AI_AGENT.SET_TEAM('Finance_Audit_Team');
SELECT DBMS_CLOUD_AI_AGENT.GET_TEAM() AS ACTIVE_TEAM FROM DUAL;
```

### Step 11. 통합 시나리오 테스트 실행

모든 준비가 완료되었습니다. 아래의 시나리오 질문들을 활용하여 Agent와 상호작용해 보세요.

**시나리오 질문 예시 (자연어)**

1. **데이터 탐지:** `최근 일주일간 거래 내역 중 적요에 '미등록' 또는 '테스트'가 포함된 건을 보여줘.`
2. **규정 조회:** `사내 부정거래 방지 규정에서 비정상 거래 발견 시 지급 동결 절차가 어떻게 돼?`
3. **동결 실행:** `방금 찾은 비정상 거래를 규정대로 동결 처리해줘.`
4. **이메일 보고:** `동결 처리 내역을 요약해서 매니저에게 메일로 보내줘.`

**호출 쿼리 예시**

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED;
DECLARE
  v_response CLOB;
  v_conv_id  VARCHAR2(4000);
BEGIN
  v_conv_id := DBMS_CLOUD_AI.CREATE_CONVERSATION();

  -- 프롬프트 영역을 원하는 시나리오 질문으로 변경하여 실행하세요.
  v_response := DBMS_CLOUD_AI_AGENT.RUN_TEAM(
    team_name   => 'Finance_Audit_Team',
    user_prompt => '최근 일주일간 거래 내역 중 적요에 미등록이 포함된 건을 보여줘.',
    params      => '{"conversation_id": "' || v_conv_id || '"}'
  );

  DBMS_OUTPUT.PUT_LINE('Conversation ID: ' || v_conv_id);
  DBMS_OUTPUT.PUT_LINE('Response: ' || DBMS_LOB.SUBSTR(v_response, 4000, 1));
END;
/
```

---

## 5. 실습 데이터 원복 (선택 사항)

테스트 과정에서 변경(동결)된 데이터를 원래 상태로 되돌리려면 아래 쿼리를 실행합니다.

```sql
UPDATE WKSP_DEMO.ASSET_TRANSACTIONS
SET STATUS = 'NORMAL', UPDATED_BY = NULL
WHERE STATUS = 'FROZEN';

COMMIT;
```

---

## 6. 자주 발생하는 오류 및 트러블슈팅

* **`ORA-... insufficient privileges` 발생**: `ADMIN` 계정으로 접속하여 권한 부여(GRANT) 구문이 모두 정상적으로 실행되었는지 재확인하세요.
* **RAG 검색 실패**: Object Storage URL, Credential 정보, 버킷 접근 권한을 확인하고, Vector Index 생성이 완료되었는지 상태를 점검하세요.
* **이메일 전송 실패**: ACL Host/Port 설정, SMTP Credential 사용자명 및 비밀번호, 그리고 송수신자(Sender/Recipient) 이메일 주소 형식을 확인하세요.
