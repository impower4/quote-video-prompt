# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 저장소 개요

이 저장소는 **오디오북/명언 영상 자동 생성 시스템을 Claude Code로 구축하기 위한 프롬프트 가이드**입니다. 현재는 문서만 존재합니다:

- `prompt.md` — 단계별(0~6단계) 프롬프트 모음 + 전체 원샷 프롬프트 + 팁. 이 저장소의 핵심 산출물.
- `README.md` — 가이드 소개, 파이프라인 다이어그램, 고정 설정값 표.

빌드/테스트/린트 명령은 없습니다. Python 코드도 아직 없습니다. 이 저장소에서의 작업은 (a) 프롬프트 문서 자체를 개선하거나, (b) `prompt.md`의 프롬프트를 실행해 시스템을 실제로 구축하는 것 중 하나입니다.

## prompt.md가 정의하는 시스템 (구축 시 반드시 준수)

`prompt.md`의 프롬프트로 시스템을 구축할 때는 아래 규칙을 따라야 합니다. `prompt.md`와 `README.md`의 설정값이 곧 스펙입니다.

### 파이프라인

Scenes 입력 → 이미지 생성(Gemini) → TTS(Gemini) → Whisper 타임스탬프 → SRT 자막 → FFmpeg 합성(fade-in-out, BGM 믹싱) → 1920x1080 MP4

### 고정 모델 (다른 모델로 대체 금지)

| 컴포넌트 | 모델 |
|----------|------|
| 이미지 | `gemini-3-pro-image-preview` |
| TTS | `gemini-2.5-pro-preview-tts` (Voice: Enceladus, 깊은 저음) |
| 타임스탬프 | Whisper `large-v3` |

Gemini API Key(Google AI Studio 발급)가 필수입니다.

### 고정 영상 스펙

1920x1080 / 30fps / 페이드 0.5초 / BGM 볼륨 15% / 자막 폰트 `assets/font/KOTRA_SONGEULSSI.ttf`(손글씨체, 하단 배치)

### 목표 폴더 구조

```
.claude/
├── skills/{generate-image,generate-tts,generate-subtitle,compose-video}/SKILL.md
├── agents/          # quote-video-agent(오케스트레이터), quote-writer-agent(콘텐츠 기획)
├── commands/        # create-quote-video.md (/createvideo)
└── settings.json
src/quote_video/     # config.py, image_generator.py, tts_generator.py,
                     # subtitle_sync.py, video_composer.py, pipeline.py
assets/font/, assets/bgm/
```

### 스킬 작성 규칙

- 스킬은 반드시 `.claude/skills/{skill-name}/SKILL.md` 경로 (파일명 정확히 `SKILL.md`)
- YAML frontmatter에 `name`, `description`, `allowed-tools` 필수 — `description`에는 "언제 사용하는지" 명시
- 본문에 Instructions / Usage / Config / Features 섹션 포함
- 각 Python 모듈은 독립적으로 테스트 가능하게 작성

### 작업 방식

복잡한 시스템은 한 번에 만들지 말고 점진적으로: Python 모듈 → 스킬 → 에이전트 → 커맨드 → settings.json 순으로 빌드하고 각 단계에서 테스트. 디버깅 시 단계별 출력을 보여주고 중간 파일은 삭제하지 않습니다.

## 문서 수정 시 주의

`prompt.md`와 `README.md`는 설정값(모델명, 영상 스펙, 폴더 구조)을 중복해서 담고 있습니다. 한쪽을 수정하면 다른 쪽도 일치하도록 갱신하세요.
