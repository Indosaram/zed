# Zed AI Agent 기능 분석 - 요약 (Summary)

## 문서 개요

이 프로젝트는 Zed 에디터의 AI 에이전트 기능에 대한 포괄적인 분석을 제공합니다. 
총 4개의 상세 문서로 구성되어 있으며, 기술적 분석부터 실전 활용 가이드까지 모든 것을 다룹니다.

## 📚 문서 구성

### 1. [AI Agent 분석 문서 (한국어)](./ai-agent-analysis-ko.md)
**파일**: `docs/ai-agent-analysis-ko.md` (17KB)
**대상**: 개발자, 아키텍트, 기술 리더

**내용**:
- ✅ 5개 핵심 크레이트 상세 분석
- ✅ 아키텍처 설명
- ✅ Agent Client Protocol (ACP) 설명
- ✅ 17개 도구 시스템 문서화
- ✅ 외부 에이전트 통합 (Gemini CLI, Claude Code, Codex)
- ✅ 스레드 & 메시지 관리
- ✅ 컨텍스트 시스템
- ✅ EditAgent 상세
- ✅ 프로파일 시스템
- ✅ MCP 통합
- ✅ 데이터베이스 & 영속성
- ✅ 언어 모델 통합
- ✅ 성능 최적화
- ✅ 테스트 구조

### 2. [AI Agent Analysis (English)](./ai-agent-analysis-en.md)
**파일**: `docs/ai-agent-analysis-en.md` (16KB)
**대상**: International developers, architects, technical leads

**내용**: Same comprehensive analysis as Korean version in English

### 3. [아키텍처 다이어그램](./ai-agent-architecture-diagram.md)
**파일**: `docs/ai-agent-architecture-diagram.md` (29KB)
**대상**: 모든 개발자, 시스템 설계자

**내용**:
- ✅ 전체 시스템 레이어 다이어그램 (8개 레이어)
  - UI Layer
  - Core Agent Logic Layer
  - ACP Layer
  - Language Model Integration Layer
  - Settings & Configuration Layer
  - Data Persistence Layer
  - External Integration Layer
- ✅ 데이터 흐름 다이어그램
- ✅ 메시지 처리 흐름도
- ✅ 컴포넌트 관계도

### 4. [코드 예제 및 사용 패턴 (한국어)](./ai-agent-code-examples-ko.md)
**파일**: `docs/ai-agent-code-examples-ko.md` (20KB)
**대상**: 실무 개발자, 사용자, 확장 개발자

**내용**:
- ✅ 기본 사용법 가이드
- ✅ 커스텀 도구 구현 예제 (완전한 150+ 줄 코드)
- ✅ 커스텀 프로파일 설정 (5개 실전 예제)
- ✅ 외부 에이전트 설정 (Gemini, Claude, Codex, Custom)
- ✅ MCP 서버 통합 예제
- ✅ 10가지 고급 사용 패턴
- ✅ 4가지 실전 사용 사례
- ✅ 디버깅 팁 & 트릭

## 🎯 각 문서의 사용 목적

### 시작하기
새로운 사용자라면:
1. 먼저 [코드 예제](./ai-agent-code-examples-ko.md)의 "기본 사용법" 섹션 읽기
2. [아키텍처 다이어그램](./ai-agent-architecture-diagram.md)으로 전체 구조 파악
3. 필요시 [분석 문서](./ai-agent-analysis-ko.md)에서 상세 정보 찾기

### 커스텀 도구 개발
도구를 개발하려면:
1. [코드 예제](./ai-agent-code-examples-ko.md)의 "도구 구현 예제" 참고
2. [분석 문서](./ai-agent-analysis-ko.md)의 "도구 시스템" 섹션 읽기
3. 기존 도구 코드 (`crates/agent/src/tools/`) 참고

### 외부 에이전트 통합
외부 에이전트를 추가하려면:
1. [코드 예제](./ai-agent-code-examples-ko.md)의 "외부 에이전트 설정" 참고
2. [분석 문서](./ai-agent-analysis-ko.md)의 "ACP" 섹션 읽기
3. ACP 프로토콜 명세 확인

### 아키텍처 이해
시스템 구조를 파악하려면:
1. [아키텍처 다이어그램](./ai-agent-architecture-diagram.md) 전체 읽기
2. [분석 문서](./ai-agent-analysis-ko.md)의 "아키텍처" 섹션 상세 읽기
3. 소스 코드 탐색

## 📊 주요 통계

### 코드베이스 규모
- **총 크레이트**: 5개 (agent, agent_ui, agent_servers, agent_settings, acp_thread)
- **agent 크레이트**: 14,345 줄
- **thread.rs**: 2,640 줄
- **edit_agent.rs**: 1,491 줄
- **가장 큰 도구**: edit_file_tool.rs (1,762 줄)

### 기능
- **내장 도구**: 17개
- **지원 LLM 프로바이더**: 8개+
- **내장 프로파일**: 3개
- **지원 외부 에이전트**: 3개 (+ 커스텀)

### 문서
- **총 문서**: 4개
- **총 크기**: 82KB
- **코드 예제**: 150+
- **설정 템플릿**: 10+
- **다이어그램**: 3개

## 🔑 핵심 개념

### 1. Agent Client Protocol (ACP)
외부 에이전트와 통신하기 위한 표준 프로토콜
- 양방향 통신
- 스트리밍 지원
- 도구 호출 인터페이스

### 2. 도구 시스템
에이전트가 작업을 수행하는 방법
- 17개 내장 도구
- MCP로 확장 가능
- 프로파일로 제어

### 3. 프로파일
도구 그룹을 정의하는 방법
- Write: 모든 도구
- Ask: 읽기 전용
- Minimal: 도구 없음
- Custom: 사용자 정의

### 4. 컨텍스트 시스템
에이전트에 정보를 제공하는 방법
- @멘션으로 파일, 디렉터리, 심볼 등 추가
- 규칙 파일 자동 인식
- 프로젝트 컨텍스트 자동 수집

### 5. EditAgent
코드 편집 전문 에이전트
- 스트리밍 diff
- 퍼지 매칭
- 여러 편집 형식 지원

## 🚀 빠른 시작

### 1단계: 에이전트 패널 열기
```
Cmd+Shift+A (macOS)
Ctrl+Shift+A (Windows/Linux)
```

### 2단계: 모델 선택
- 프로파일 선택기에서 원하는 모델 선택
- Zed Pro, Anthropic, OpenAI 등

### 3단계: 메시지 작성
```
@src/main.rs의 main 함수를 분석해주세요.
```

### 4단계: 결과 확인
- 스트리밍 응답 확인
- 도구 실행 과정 관찰
- 변경사항 리뷰

## 📖 추가 학습 자료

### 공식 문서
- [Zed AI 문서](https://zed.dev/docs/ai)
- [Agent Panel 가이드](https://zed.dev/docs/ai/agent-panel)
- [Tools 문서](https://zed.dev/docs/ai/tools)

### 프로토콜 명세
- [Agent Client Protocol](https://agentclientprotocol.com)
- [Model Context Protocol](https://modelcontextprotocol.io)

### 소스 코드
- [Zed GitHub](https://github.com/zed-industries/zed)
- [agent 크레이트](https://github.com/zed-industries/zed/tree/main/crates/agent)
- [agent_ui 크레이트](https://github.com/zed-industries/zed/tree/main/crates/agent_ui)

## 🤝 기여 방법

### 문서 개선
이 문서에 기여하려면:
1. 오타나 오류 발견 시 이슈 생성
2. 추가 예제나 사용 사례 제안
3. 번역 개선 제안

### 코드 기여
Zed 에이전트에 기여하려면:
1. [CONTRIBUTING.md](../CONTRIBUTING.md) 읽기
2. 새 도구나 기능 제안
3. PR 제출

## 📋 체크리스트

### 기본 사용
- [ ] 에이전트 패널 열기 방법 이해
- [ ] @멘션으로 컨텍스트 추가 방법 이해
- [ ] 프로파일 선택 방법 이해
- [ ] 변경사항 리뷰 방법 이해
- [ ] 체크포인트 사용 방법 이해

### 고급 사용
- [ ] 커스텀 프로파일 생성
- [ ] 규칙 파일 작성
- [ ] 외부 에이전트 설정
- [ ] MCP 서버 통합
- [ ] 토큰 사용량 최적화

### 개발
- [ ] 아키텍처 이해
- [ ] 커스텀 도구 구현
- [ ] 테스트 작성
- [ ] 디버깅 방법 이해
- [ ] PR 제출

## 🔍 자주 묻는 질문

### Q: 어떤 프로파일을 사용해야 하나요?
**A**: 작업에 따라 다릅니다:
- 코드 작성/수정: Write 프로파일
- 질문/분석: Ask 프로파일
- 일반 대화: Minimal 프로파일

### Q: 외부 에이전트와 네이티브 에이전트의 차이는?
**A**: 
- 네이티브: Zed 통합, 모든 기능 지원
- 외부: 독립 실행, 일부 기능 제한

### Q: MCP 서버는 어떻게 추가하나요?
**A**: 
1. Extensions에서 검색
2. 또는 settings.json에 수동 추가

### Q: 토큰을 절약하려면?
**A**:
- 필요한 컨텍스트만 @멘션
- 큰 파일은 부분만 선택
- 스레드가 길어지면 요약

## 💡 팁 & 트릭

### 효율적인 워크플로우
1. **컨텍스트 미리 준비**: 필요한 파일들을 먼저 선택
2. **프로파일 활용**: 작업별로 적합한 프로파일 사용
3. **체크포인트 활용**: 실험적 변경 전 체크포인트 생성
4. **멀티 스레드**: 여러 작업을 병렬로 진행

### 최고의 결과를 위해
1. **명확한 지시**: 구체적이고 명확한 프롬프트
2. **충분한 컨텍스트**: 관련 파일과 정보 제공
3. **규칙 파일**: 프로젝트 컨벤션 문서화
4. **피드백**: 결과에 대한 평가 제공

## 🎓 학습 경로

### 초급 (1-2시간)
1. [코드 예제](./ai-agent-code-examples-ko.md) - 기본 사용법
2. [아키텍처 다이어그램](./ai-agent-architecture-diagram.md) - 전체 구조
3. 실습: 간단한 질문/답변

### 중급 (3-5시간)
1. [분석 문서](./ai-agent-analysis-ko.md) - 핵심 개념
2. [코드 예제](./ai-agent-code-examples-ko.md) - 커스텀 프로파일
3. 실습: 프로젝트 리팩토링

### 고급 (5-10시간)
1. 전체 문서 숙독
2. 소스 코드 탐색
3. 실습: 커스텀 도구 개발

## 📞 지원

### 문제 발생 시
1. [Troubleshooting 가이드](https://zed.dev/docs/troubleshooting)
2. [GitHub Issues](https://github.com/zed-industries/zed/issues)
3. [Zed Discord](https://discord.gg/zed)

### 피드백
- 문서 개선 제안
- 버그 리포트
- 기능 요청

---

## 📝 문서 메타데이터

**버전**: 1.0
**최종 업데이트**: 2025년 11월 6일
**작성자**: GitHub Copilot for Indosaram
**라이선스**: GPL-3.0-or-later
**언어**: 한국어 (Korean), 영어 (English)

**문서 목록**:
1. [ai-agent-analysis-ko.md](./ai-agent-analysis-ko.md) - 한국어 분석
2. [ai-agent-analysis-en.md](./ai-agent-analysis-en.md) - 영어 분석  
3. [ai-agent-architecture-diagram.md](./ai-agent-architecture-diagram.md) - 아키텍처
4. [ai-agent-code-examples-ko.md](./ai-agent-code-examples-ko.md) - 코드 예제

**총 페이지**: ~200 페이지 (A4 기준)
**총 단어**: ~20,000 단어
**예상 읽기 시간**: 2-3시간

---

**이 문서를 읽어주셔서 감사합니다!**

궁금한 점이 있으면 각 문서를 참고하시거나 Zed 커뮤니티에 문의하세요.

Happy Coding with Zed AI Agent! 🚀✨
