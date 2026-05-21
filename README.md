# Oracle Autonomous AI Database Select AI Agent 핸즈온

`SelectAIdemo.sql` 내용을 초보자 기준으로 다시 정리한 실습 가이드입니다.
순서가 섞여 있던 스크립트를 **처음부터 끝까지 따라 실행 가능한 순서**로 구성했습니다.

## 1. 실습 목표

이 실습을 완료하면 다음을 할 수 있습니다.

- `SELECT AI`로 자연어 질의 → SQL 조회 실행
- 재무 데이터를 위한 AI Profile/Tool/Task/Agent/Team 구성
- RAG(Object Storage 기반)로 규정 문서 검색
- 함수 기반 Tool로 비정상 거래 동결 처리
- Notification Tool로 이메일 보고 자동화

---

## 2. 준비사항

## 필수 환경

- Oracle Cloud Autonomous Database (Select AI / AI Agent 사용 가능 버전)
- 실습 계정 2개
1. `ADMIN` (권한 부여/ACL 설정용)
2. `WKSP_DEMO` (실습 실행 사용자)
- OCI Credential (예: `OCI_KEY_CRED`) 사전 생성
- (RAG 실습 시) Object Storage 버킷 + 규정 PDF 업로드
- (Email 실습 시) OCI Email Delivery SMTP 계정/비밀번호

## 권장 도구

- SQL Developer, SQLcl, Database Actions 중 하나

## 주의

- 원본 SQL에는 SMTP 사용자/비밀번호가 평문으로 포함되어 있습니다.
- GitHub 배포 시 **절대 실제 계정/비밀번호를 커밋하지 마세요**.
- 아래 문서에서는 모두 플레이스홀더(`<...>`)로 표기합니다.

---

## 3. 실습 흐름 한눈에 보기

1. 테이블 생성 + Select AI용 코멘트/어노테이션
2. Select AI Profile 생성
3. 합성 데이터 생성
4. Select AI 기본 질의 테스트
5. RAG Tool 구성
6. SQL / Freeze / Email Tool 구성
7. Task 생성
8. Agent 생성
9. Team 생성/활성화
10. 통합 시나리오 실행
11. 원복(선택)

---

## 4. Step-by-Step 실습

## Step 1) 재무 샘플 테이블 생성

`WKSP_DEMO` 계정에서 실행:

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

## Step 2) Select AI 이해도를 높이기 위한 주석/어노테이션 추가

핵심은 **테이블/컬럼 의미를 자연어로 설명**하는 것입니다.
원본 스크립트처럼 `COMMENT ON ...`과 JSON annotation을 추가합니다.

예시:

```sql
COMMENT ON TABLE PROJECT_BUDGETS IS '{"comment": "프로젝트별 예산 실적", "annotation": {"description": "각 프로젝트의 분기별 예산 편성액과 실제 집행 비용"}}';
COMMENT ON COLUMN PROJECT_BUDGETS.ACTUAL_SPENT IS '{"comment": "집행액", "annotation": {"logic": "집행액이 예산액보다 크면 예산 초과"}}';
```

팁:

- 한글 비즈니스 용어를 충분히 써주면 자연어 질의 정확도가 올라갑니다.
- `synonyms`, `example`, `logic`, `formula`를 적극 활용하세요.

## Step 3) Select AI Profile 생성

`WKSP_DEMO` 계정에서 실행:

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

SELECT PROFILE_NAME, STATUS, CREATED
FROM USER_CLOUD_AI_PROFILES;
```

## Step 4) 합성 데이터 생성

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

확인:

```sql
SELECT * FROM PROJECT_BUDGETS;
SELECT * FROM PRODUCT_PERFORMANCE;
SELECT * FROM CASH_ASSETS;
SELECT * FROM ASSET_TRANSACTIONS;
```

## Step 5) Select AI 기본 질의 테스트

```sql
EXEC DBMS_CLOUD_AI.SET_PROFILE('OCI_AI_PROFILE');

SELECT AI "현재 예산을 초과해서 사용하고 있는 프로젝트 리스트를 보여줘.";
SELECT AI "4분기 잔여 예산으로 집행 가능한 최대 금액이 얼마 남았지?";
SELECT AI narrate "지난달 제품군별 영업이익률 순위대로 정렬해줘.";
SELECT AI "현재 우리 회사의 현금성 자산 보유 현황을 계좌별로 요약해줘.";
```

---

## Step 6) RAG Tool 구성 (규정 PDF 검색)

### 6-1) Object Storage 파일 확인

```sql
SELECT object_name, bytes
FROM DBMS_CLOUD.LIST_OBJECTS(
  credential_name => 'OCI_KEY_CRED',
  location_uri    => 'https://swiftobjectstorage.<region>.oraclecloud.com/v1/<namespace>/<bucket>'
);
```

### 6-2) 임베딩/벡터 인덱스/프로파일 생성

```sql
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

BEGIN
  DBMS_CLOUD_AI.CREATE_VECTOR_INDEX(
    index_name => 'RAG_INDEX',
    attributes => '{
      "vector_db_provider": "oracle",
      "location": "https://swiftobjectstorage.<region>.oraclecloud.com/v1/<namespace>/<bucket>",
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

---

## Step 7) 권한/네트워크/유틸 함수 준비

### 7-1) 권한 부여 (`ADMIN` 계정)

```sql
GRANT EXECUTE ON C##CLOUD$SERVICE.DBMS_CLOUD_AI_AGENT TO WKSP_DEMO;
GRANT EXECUTE ON C##CLOUD$SERVICE.DBMS_CLOUD_AI TO WKSP_DEMO;
GRANT EXECUTE ON DBMS_LOB TO WKSP_DEMO;
GRANT SELECT ON DBA_AI_AGENT_TASKS TO WKSP_DEMO;
GRANT SELECT ON DBA_AI_AGENT_TASK_ATTRIBUTES TO WKSP_DEMO;
GRANT SELECT ON DBA_AI_AGENT_TOOLS TO WKSP_DEMO;
GRANT SELECT ON DBA_AI_AGENT_TOOL_ATTRIBUTES TO WKSP_DEMO;
```

### 7-2) SMTP ACL 설정 (`ADMIN` 계정)

```sql
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
```

### 7-3) `run_team_clob` 함수 생성 (`WKSP_DEMO`)

```sql
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

---

## Step 8) Tool 생성

### 8-1) SQL Tool

```sql
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
```

### 8-2) 동결 함수 + Tool

```sql
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
```

### 8-3) Email Notification Tool

1) SMTP Credential 생성:

```sql
BEGIN
  DBMS_CLOUD.CREATE_CREDENTIAL(
    credential_name => 'OCI_SMTP_CRED',
    username        => '<smtp_username>',
    password        => '<smtp_password>'
  );
END;
/
```

2) Notification Tool 생성:

```sql
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

---

## Step 9) Task 생성

원본 스크립트에는 Task를 여러 번 재생성하는 버전이 있습니다.
초보자 실습에서는 아래처럼 **최종 지침 1개**만 사용하면 됩니다.

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

---

## Step 10) Agent + Team 생성 및 활성화

```sql
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

---

## Step 11) 통합 시나리오 테스트

### 시나리오 질문 예시

1. 데이터 탐지:

```text
최근 일주일간 거래 내역 중 적요에 '미등록' 또는 '테스트'가 포함된 건을 보여줘.
```

2. 규정 조회:

```text
사내 부정거래 방지 규정에서 비정상 거래 발견 시 지급 동결 절차가 어떻게 돼?
```

3. 동결 실행:

```text
방금 찾은 비정상 거래를 규정대로 동결 처리해줘.
```

4. 이메일 보고:

```text
동결 처리 내역을 요약해서 매니저에게 메일로 보내줘.
```

### `RUN_TEAM` 호출 예시

```sql
SET SERVEROUTPUT ON SIZE UNLIMITED;
DECLARE
  v_response CLOB;
  v_conv_id  VARCHAR2(4000);
BEGIN
  v_conv_id := DBMS_CLOUD_AI.CREATE_CONVERSATION();

  v_response := DBMS_CLOUD_AI_AGENT.RUN_TEAM(
    team_name   => 'Finance_Audit_Team',
    user_prompt => 'Send an email notification saying: Oracle Select AI Agent test successful.',
    params      => '{"conversation_id": "' || v_conv_id || '"}'
  );

  DBMS_OUTPUT.PUT_LINE('Conversation ID: ' || v_conv_id);
  DBMS_OUTPUT.PUT_LINE('Response: ' || DBMS_LOB.SUBSTR(v_response, 4000, 1));
END;
/
```

---

## 12. 실습 후 원복(선택)

동결 테스트로 변경된 데이터를 원복하려면:

```sql
UPDATE WKSP_DEMO.ASSET_TRANSACTIONS
SET STATUS = 'NORMAL', UPDATED_BY = NULL
WHERE TRANS_ID IN (<id1>, <id2>, <id3>);

COMMIT;
```

---

## 13. 자주 발생하는 오류와 해결

- `ORA-... insufficient privileges`
1. `ADMIN`으로 권한 부여 구문 재확인

- RAG 검색 실패
1. Object Storage URL/credential/bucket 권한 확인
2. Vector Index 생성 성공 여부 확인

- Email 전송 실패
1. ACL host/port 확인
2. SMTP credential 정확성 확인
3. sender/recipient 형식 확인

---

## 14. GitHub 배포 체크리스트

- 실 계정정보/비밀번호/OCID는 전부 플레이스홀더로 치환
- 실행 계정(`ADMIN`, `WKSP_DEMO`)을 단계마다 명시
- 실패 시 확인 쿼리와 복구 절차 포함
- 데모 질문(한국어 자연어 질의) 예시 포함

---

필요하면 다음 단계로, 제가 이 문서를 기반으로 `README.md`까지 바로 정리해드릴 수 있습니다.
