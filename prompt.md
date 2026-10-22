# 오디오북 영상 시스템 프롬프트 가이드

Claude Code로 에이전트 + 스킬 기반 오디오북 영상 시스템을 처음부터 만드는 프롬프트 모음

---

## 0단계: Claude Code 스킬/에이전트 구조 이해

2026-10-23 공식 문서 확인: 프로젝트 스킬은 `.claude/skills/<이름>/SKILL.md`에 둔다. 개인 스킬의 경로는 실제 Claude 설정 디렉터리를 확인한다. `allowed-tools`는 선택 항목이며 이 가이드에서는 기본으로 권한을 추가하지 않는다. 스킬 발견을 위해 `settings.json`에 목록을 등록할 필요는 없다.

공식 문서: https://code.claude.com/docs/en/skills

### 스킬 폴더 구조 (필수)
```
.claude/skills/
├── generate-image/
│   └── SKILL.md          ← 반드시 SKILL.md 파일명 사용
├── generate-tts/
│   └── SKILL.md
├── generate-subtitle/
│   └── SKILL.md
└── compose-video/
    └── SKILL.md
```

### SKILL.md 형식 (YAML Frontmatter 필수)
```markdown
---
name: skill-name
description: 스킬 설명. 언제 사용하는지 명시하세요.
---

# Skill Title

## Instructions
사용 방법 설명...

## Usage
코드 예제...
```

### 에이전트/커맨드 구조
```
.claude/
├── agents/
│   └── video-orchestrator.md
├── commands/
│   └── createvideo.md
└── settings.json
```

---

## 실행 범위와 재개 기준

- 현재 요청, 최근 승인 파일·버전, 금지사항, 예산을 먼저 복원한다. 이미 승인한 같은 작업을 매번 다시 묻지 않는다.
- 대본 → 이미지 → 짧은 모션 → 본편의 합의된 관문을 유지한다. 설계 요청만으로 유료 생성·게시까지 실행하지 않는다.
- 사용자가 크레딧 0 또는 특정 서비스 금지를 정했으면 그대로 유지한다. 연결됐다는 이유로 유료 호출을 허가받았다고 간주하지 않는다.
- 기존 스타일을 먼저 유지한다. 아래 연필 스케치·미니멀 예시는 선택 가능한 별도 방향이며 한 작품 안에서 자동 혼용하지 않는다.
- 생성 POST의 시간 초과·서버 오류는 서비스 기록과 작업 ID를 확인한 뒤 처리하며 자동 재전송으로 중복 과금하지 않는다. 이미 생성된 청크·응답을 새 버전 경로에 보존한다.
- 음성 청크 오프셋은 디코딩한 실제 길이와 삽입한 무음 샘플 수로 계산한다. 자막 줄바꿈에서 발화 글자를 버리지 않는다. 배열 길이·유효 시간·겹침을 검사한다.
- 실제 음성을 기준으로 SRT를 검수한다. 파일 생성, 프레임 확인, 정상 속도 재생, 청취를 나눠 기록한다.
- 완성 나레이션 영상은 요청한 MP4·음성·SRT·최종 대본을 확인한다. 썸네일·블로그·PDF는 요청된 경우에만 추가한다.

## 1단계: 아키텍처 설계

```
오디오북 영상 자동 생성 시스템을 만들거야.

파이프라인:
1. 이미지 생성 (Gemini) - 연필 스케치 스타일
2. TTS 생성 (Gemini) - 저음의 진지한 나레이션
3. Whisper 타임스탬프 추출 (large-v3)
4. SRT 자막 생성
5. FFmpeg 영상 합성 (fade-in-out 효과)

이걸 Claude Code의 에이전트와 스킬 구조로 분리해서 설계해줘.

**중요: Claude Code 공식 스킬 구조를 따라야 해:**
- 스킬: .claude/skills/{skill-name}/SKILL.md 형식
- 이 가이드의 YAML frontmatter에는 name, description을 작성한다. allowed-tools는 선택이며 필요 없이 권한을 넓히지 않는다.
- 에이전트: 오케스트레이터 역할

아키텍처 다이어그램과 폴더 구조를 보여줘.
```

---

## 2단계: Python 모듈 생성

```
이제 Python 모듈을 만들어줘.

src/quote_video/ 폴더 아래에:
1. config.py - 공통 설정 (환경변수로 읽는 API 키, 모델명, 경로, 스타일 프롬프트). 비밀값은 소스·로그에 저장하지 않는다
2. image_generator.py - 노션 스타일 이미지 생성
3. tts_generator.py - TTS 나레이션 생성
4. subtitle_sync.py - Whisper 타임스탬프 + SRT 생성
5. video_composer.py - FFmpeg 영상 합성
6. pipeline.py - 전체 파이프라인 조율

각 모듈은 독립적으로 테스트 가능하게 만들어줘.
```

### 세부 요구사항 (필요시 추가)

```
이미지 생성 요구사항:
- 모델: 현재 계정에서 지원 여부·요금·입출력을 확인한 모델을 설정으로 지정한다. 과거 preview 모델명을 강제하지 않는다
- 스타일: 노션 스타일 미니멀 일러스트
- 배경: 크림색/베이지
- 선: 굵은 검정 라인
```

```
TTS 요구사항:
- 모델: 현재 연결에서 지원·비용을 확인한 TTS 모델
- 보이스: 승인된 한 목소리를 설정으로 유지한다. Charon·Enceladus는 비교 후보이며 자동 확정하지 않는다
- 용도: 수면/명상 콘텐츠
- 출력: WAV (24000Hz, 16bit, mono)
```

```
영상 합성 요구사항:
- 해상도: 1920x1080, 30fps
- 자막: 손글씨체, 하단 배치
- BGM: 사용 권한이 확인된 녹음만 사용한다. 나레이션 명료도·피크·덕킹을 실제 청취해 조절하고 고정 백분율을 품질 기준으로 삼지 않는다
```

---

## 3단계: 스킬 정의 (Claude Code 공식 구조)

````text
이제 .claude/skills/ 폴더에 스킬을 정의해줘.

**중요: Claude Code 공식 스킬 구조를 따라야 해!**

폴더 구조:
.claude/skills/
├── generate-image/
│   └── SKILL.md
├── generate-tts/
│   └── SKILL.md
├── generate-subtitle/
│   └── SKILL.md
└── compose-video/
    └── SKILL.md

각 SKILL.md는 반드시 YAML frontmatter로 시작해야 해:

---
name: generate-image
description: 승인된 모델·스타일로 이미지를 생성합니다. 명언 영상의 배경 이미지가 필요할 때, 철학적/예술적 스케치 이미지를 만들 때 사용하세요.
---

# Generate Image

## Instructions
...

## Usage
```python
from src.quote_video.image_generator import ImageGenerator
generator = ImageGenerator()
generator.generate(prompt="...", output_path=...)
```

각 스킬에 포함할 내용:
- Instructions: 사용 시점과 방법
- Usage: 코드 예제
- Config: 모델명, 설정값 표
- Features: 제공 기능 목록
````

---

## 4단계: 에이전트 정의

```
이제 .claude/agents/ 폴더에 에이전트를 정의해줘.

만들 에이전트:
1. quote-video-agent - 영상 제작 오케스트레이터
   - 스킬들을 순차 호출
   - 에러 핸들링
   - 결과 집계

2. quote-writer-agent - 콘텐츠 기획
   - 명언 선별
   - 이미지 프롬프트 작성
   - scenes 데이터 구성

각 에이전트의 역할, 사용 스킬, 입출력 형식을 명시해줘.
```

---

## 5단계: 커맨드 정의

```
/createvideo 슬래시 커맨드를 만들어줘.

.claude/commands/createvideo.md에:
- 사용법
- 파이프라인 흐름
- 실행 절차
- 옵션 (BGM 등)
을 정의해줘.
```

---

## 6단계: 테스트 실행

```
출처·문구·번역과 요청 범위를 확인한 명언으로 테스트 영상을 만들어줘.
처음에는 승인된 한 장면으로 전체 흐름을 검증하고, 배치 제작이 요청됐을 때만 장면 수를 늘려줘.

scenes 데이터:
- narration: 한국어 명언
- image_prompt: 영어 씬 설명

배경음악은 곡명만 보고 사용하지 말고 해당 녹음의 사용 조건을 확인해줘. 권한이 확인된 파일이 없으면 음성만으로 검증하고 누락을 알려줘.
```

---

## 전체 원샷 프롬프트 (한 번에 다 만들기)

```
명언 영상 자동 생성 시스템을 만들어줘.

## 요구사항
1. 현재 지원·비용을 확인한 모델로 승인된 연필 스케치 스타일 이미지 생성
2. 승인된 TTS 모델·보이스로 나레이션. 무료 방식과 기존 음성을 먼저 검토
3. 환경에 맞는 전사·정렬 도구로 타임스탬프를 추출하고 실제 한국어 발화와 대조
4. FFmpeg로 영상 합성 (fade-in-out, 자막)
5. 사용 권한이 확인된 BGM을 실제 청취하며 믹싱

## Claude Code 스킬 구조 (공식 가이드라인)
반드시 아래 구조를 따라야 해:

.claude/skills/
├── generate-image/
│   └── SKILL.md     ← YAML frontmatter 필수
├── generate-tts/
│   └── SKILL.md
├── generate-subtitle/
│   └── SKILL.md
└── compose-video/
    └── SKILL.md

각 SKILL.md는 이 형식을 따라:
---
name: skill-name
description: 스킬 설명 + 사용 시점
---

## 전체 구조
- Python 모듈: src/quote_video/
- 스킬: .claude/skills/{skill-name}/SKILL.md
- 에이전트: .claude/agents/video-orchestrator.md
- 커맨드: .claude/commands/createvideo.md
- 설정: .claude/settings.json은 필요한 실제 설정이 있을 때만 사용. 스킬 목록 등록용으로 만들지 않음

## 스타일
- 이미지: 연필 스케치, 빈티지 종이 배경
- 영상: 1920x1080, 30fps, fade-in-out 0.5초
- 자막: 손글씨체 (KOTRA_SONGEULSSI), 하단 배치
- 폰트: assets/font/KOTRA_SONGEULSSI.ttf
- BGM: assets/bgm/ 폴더의 음악

각 모듈은 독립 테스트 가능하게,
스킬/에이전트는 SKILL.md에 사용법 문서화해줘.
```

---

## 팁

### 프로젝트별 모델·보이스 유지
```
현재 지원되는 모델과 비용을 먼저 확인해.
프로젝트에서 승인한 모델·보이스·스타일·해상도는 설정 한 곳에 기록하고 유지해.
지원 중단·오류가 발생하면 원인을 확인하고, 비용이나 결과가 달라지는 교체는 현재 승인 범위에 맞춰 결정해.
전사 도구는 환경에 맞게 선택하고 실제 발화와 싱크를 검수해.
```

### Claude Code 스킬 구조 체크리스트
```
스킬 만들 때 이 체크리스트 확인해:
☐ .claude/skills/{skill-name}/ 폴더 생성
☐ 폴더 안에 SKILL.md 파일 생성 (파일명 정확히)
☐ YAML frontmatter에 name, description 포함. allowed-tools는 필요한 경우에만 검토
☐ description에 "언제 사용하는지" 명시
☐ Instructions, Usage, Config 섹션 포함
☐ 새 세션에서 스킬 목록·명시적 호출로 발견 여부 확인
```

### 점진적 빌드
복잡한 시스템은 한 번에 만들지 말고:
1. 먼저 Python 모듈 → 테스트
2. 스킬 폴더 구조 생성 → SKILL.md 작성
3. 에이전트 정의 → 스킬 연결
4. 커맨드 → 통합 테스트
5. 실제 호출과 필요한 설정 확인. 스킬 등록을 이유로 권한 설정을 변경하지 않기

### 디버깅
```
에러 나면 각 단계별로 출력 보여줘.
중간 파일들 삭제하지 마.
```

### SKILL.md 예제 템플릿
```markdown
---
name: my-skill
description: 이 스킬의 설명. OO할 때, XX가 필요할 때 사용하세요.
---

# My Skill

설명...

## Instructions

1. 단계 1
2. 단계 2
3. 단계 3

## Usage

\`\`\`python
from src.module import MyClass
instance = MyClass()
instance.do_something()
\`\`\`

## Config

| 항목 | 값 |
|------|-----|
| 모델 | model-name |
| 옵션 | value |

## Features

1. **feature_1**: 기능 설명
2. **feature_2**: 기능 설명
```
