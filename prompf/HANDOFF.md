# Brand Prompt Dashboard — Claude Code Handoff
> Medibuilder DoAi TF · 디자인 모듈 · v0.2

---

## 프로젝트 개요

3개 병원 브랜드(르샤인 · 오블리브 · 온리프)의 AI 이미지 생성 프롬프트를 브랜드 무드보드 DB 기반으로 자동 조합해주는 대시보드.

**핵심 목적**
- 브랜드 정체성을 지키면서 다양한 이미지 프롬프트를 빠르게 생성
- 플래너/디자이너가 직접 쓸 수 있는 self-serve 툴
- 향후 이미지 API(Flux 등) 연결로 원클릭 이미지 생성까지 확장

---

## 현재 완성된 것 (claude.ai에서 작업)

### 1. 브랜드 DB (JSON)
3개 브랜드 각각의 무드보드 데이터 구조화 완료.

```json
{
  "brands": {
    "leshine": {
      "brand_id": "leshine",
      "colors": {
        "primary": "#AE8D4A",
        "sub": "#EDE6D8",
        "accent_dark": "#143428",
        "accent_mid": "#307E61"
      },
      "tone": ["luxury", "elegant", "natural", "refined", "warm"],
      "subject_keywords": ["soft natural light", "golden hour", "minimal jewelry", "fine fabric texture", "editorial beauty"],
      "lighting": { "style": "soft diffused backlit", "color_temp": "5600K warm", "descriptor": "golden hour, warm glowing" },
      "mood_score": { "elegance": 9, "natural": 8, "luxury": 9, "clinical": 2, "fresh": 5 },
      "forbidden": ["harsh flash", "neon colors", "cold blue tone", "oversaturated", "heavy filter"],
      "prompt_base": "warm tone, luxury editorial beauty, korean model, golden light, soft bokeh, high-end fashion magazine style"
    },
    "obliv": {
      "brand_id": "obliv",
      "colors": {
        "primary": "#3A2414",
        "sub": "#E8D9C1",
        "accent_gold": "#7F4800",
        "accent_red": "#9D4724"
      },
      "tone": ["warm", "healing", "wood", "earthy", "stable"],
      "subject_keywords": ["wood grain texture", "warm earth tones", "nature healing", "matte skin glow", "organic material"],
      "lighting": { "style": "warm ambient candlelike", "color_temp": "3200K amber", "descriptor": "dim warm, candle glow, intimate" },
      "mood_score": { "elegance": 7, "natural": 8, "luxury": 7, "clinical": 3, "healing": 9 },
      "forbidden": ["cool gray tone", "bright clinical white", "high contrast harsh", "cold blue filter"],
      "prompt_base": "earth tone, warm amber light, healing atmosphere, korean model, wood texture background, matte skin, intimate cozy mood"
    },
    "onlif": {
      "brand_id": "onlif",
      "display_name": "온리프 성형외과",
      "colors": {
        "primary": "#CCD8E8",
        "accent_green": "#495E43",
        "accent_green_sub": "#607B59",
        "accent_pink": "#FED3D0",
        "accent_pink_sub": "#FFE3E1",
        "bg_offwhite": "#F8F8F8",
        "text_dark": "#383838"
      },
      "tone": ["premium", "trustworthy", "natural", "clean", "sophisticated"],
      "subject_keywords": ["soft blue tone", "natural result", "clean minimal space", "premium clinic atmosphere", "dewy skin", "refined beauty"],
      "lighting": { "style": "clean bright neutral", "color_temp": "5500K neutral daylight", "descriptor": "airy, premium clinic, soft neutral" },
      "mood_score": { "elegance": 8, "natural": 9, "luxury": 7, "clinical": 6, "fresh": 7 },
      "forbidden": ["warm yellow cast", "dark moody filter", "heavy dramatic makeup", "oversaturated color", "harsh medical look"],
      "prompt_base": "soft blue-grey tone, premium plastic surgery clinic, korean model, natural result beauty, clean neutral light, minimal elegant background, trustworthy refined atmosphere",
      "typography": { "primary_font": "Pretendard", "english_font": "Poppins", "point_font": "Sandoll BaikzongyulPen" }
    }
  }
}
```

---

### 2. 브랜드별 허용 풀 + 금지 목록 (정석/변주 모드 기반)

기본값 고정 방식 → **허용 풀 + 금지 목록** 방식으로 설계.
- **정석 모드**: 허용 풀의 첫 번째 값 자동 적용
- **변주 모드**: 허용 풀 안에서 랜덤 조합 → 브랜드 톤 유지하면서 다양한 이미지

```json
{
  "leshine": {
    "locked": {
      "forbidden_lighting": ["neutral_clinical", "cold_blue", "harsh_flash"],
      "forbidden_grade": ["cool_premium", "earth_matte"]
    },
    "allowed_pool": {
      "lighting": ["golden_editorial", "warm_backlit", "soft_diffused_warm"],
      "color_grade": ["warm_gold", "bright_airy_warm", "soft_peach"]
    },
    "free_variables": ["composition", "pose", "expression", "hair", "outfit", "background"]
  },
  "obliv": {
    "locked": {
      "forbidden_lighting": ["cool_blue", "bright_clinical", "harsh_flash"],
      "forbidden_grade": ["cool_premium", "bright_airy"]
    },
    "allowed_pool": {
      "lighting": ["warm_amber", "soft_diffused_warm", "intimate_dim"],
      "color_grade": ["earth_matte", "warm_gold", "amber_soft"]
    },
    "free_variables": ["composition", "pose", "expression", "hair", "outfit", "background"]
  },
  "onlif": {
    "locked": {
      "forbidden_lighting": ["warm_amber", "golden_editorial", "harsh_flash"],
      "forbidden_grade": ["earth_matte", "warm_gold"]
    },
    "allowed_pool": {
      "lighting": ["neutral_clinical", "bright_natural", "soft_diffused_cool"],
      "color_grade": ["cool_premium", "bright_airy", "soft_blue_grey"]
    },
    "free_variables": ["composition", "pose", "expression", "hair", "outfit", "background"]
  }
}
```

---

### 3. 카테고리 × 파라미터 구조

```
카테고리 6개
├── beauty_face       뷰티모델 얼굴이펙트
├── portrait_daily    인물 일상
├── body_model        바디모델
├── procedure_before  시술 전
├── procedure_after   시술 후
└── procedure_compare 전후 비교

파라미터 8개 (모든 카테고리 공통)
├── composition   구도 (bust_up / face_close / half_body)
├── ratio         종횡비 (9:16 / 1:1 / 4:5)
├── lighting      조명 → 브랜드 허용 풀에서만 선택
├── color_grade   색감 → 브랜드 허용 풀에서만 선택
├── pose          포즈 (hand_to_face / straight_gaze / neck_elongated)
├── expression    표정 (warm_calm / confident / joyful_soft)
├── hair          헤어 (long_straight_black / long_wavy_brown / updo_clean / short_bob)
├── outfit        의상 (slip_white / nude_shoulder / clinic_robe / casual_minimal)
└── background    배경 (pastel_gradient / pure_white / cream_warm / dark_elegant)

파라미터 1개 (beauty_face 전용)
└── effect        스킨이펙트 (dewy_glass / shimmer_highlight / light_refraction / luminous_moisture / prism_scatter)
```

---

### 4. 프롬프트 조합 공식

```
최종 프롬프트 = 
  [identity_lock]           ← 얼굴 고정 (참조 이미지 있을 때)
  [composition + ratio]
  [fixed_style]             ← "soft feminine clean studio beauty portrait..."
  [outfit]
  [pose]
  [expression]
  [hair]
  [lighting]                ← 브랜드 허용 풀
  [color_grade]             ← 브랜드 허용 풀
  [background]
  [effect]                  ← beauty_face만
  [brand.prompt_base]       ← 브랜드 고정 베이스
  [camera_spec]             ← "full-frame, 85mm, shallow DoF..."
  [quality_suffix]          ← "8K, RAW photo, hyperrealistic..."

Negative prompt =
  [brand.forbidden 반영]
  + "cartoon, CGI, anime, deformed face, watermark..."
```

---

### 5. 현재 UI (IBM Carbon 스타일 HTML)

- 첨부 파일: `prompt-dashboard.html`
- 3단 레이아웃: 브랜드/카테고리 선택(좌) · 옵션(중) · 프롬프트 출력(우)
- 정석/변주 모드 토글
- 브랜드 전환 시 허용 풀 칩 자동 업데이트
- Copy Prompt / Copy Negative 버튼

---

## Claude Code에서 할 것 (우선순위 순)

### Phase 1 — 로컬 완성 (즉시 시작 가능)

| # | 작업 | 설명 |
|---|------|------|
| 1 | **파일 구조 분리** | `brandDB.json` / `promptTemplates.json` / `index.html` / `main.js` 분리 |
| 2 | **참조 이미지 업로드** | `<input type="file">` → base64 → 미리보기 표시, identity_lock 자동 on/off |
| 3 | **프롬프트 히스토리** | localStorage에 최근 20개 저장, 재불러오기 |
| 4 | **카테고리별 파라미터 분기** | 시술 전/후/비교는 별도 파라미터 세트 필요 (clinical 옵션 강화) |
| 5 | **변주 모드 고도화** | free_variables도 랜덤 조합 옵션 추가 |

### Phase 2 — 이미지 API 연결

| # | 작업 | 설명 |
|---|------|------|
| 6 | **Flux API 연결** | `fal.ai` 또는 `replicate.com` Flux 엔드포인트, 프롬프트 → 이미지 생성 |
| 7 | **img2img / 얼굴 고정** | 참조 이미지 + 프롬프트 → IP-Adapter 또는 Flux Redux 방식 |
| 8 | **생성 이미지 갤러리** | 생성 결과 그리드 뷰, 브랜드/카테고리 필터 |
| 9 | **전후 비교 생성** | 동일 인물 before/after 일관성 유지 로직 |

### Phase 3 — 팀 배포

| # | 작업 | 설명 |
|---|------|------|
| 10 | **Supabase or Firebase** | 프롬프트/이미지 히스토리 팀 공유 DB |
| 11 | **브랜드별 권한** | 르샤인 담당자는 르샤인만 접근 등 |
| 12 | **Slack 연동** | 생성된 이미지 자동 공유 |

---

## Gamma 스타일 프롬프트 (확정)

Gamma AI 이미지 스타일 필드에 바로 복붙하는 브랜드별 확정 프롬프트.

### 테스트 과정에서 확인된 규칙

| 규칙 | 이유 |
|------|------|
| 브랜드명 절대 쓰지 말 것 | "Le Shine" 입력 시 로고가 이미지에 직접 렌더링됨 |
| 레퍼런스 브랜드 쓰지 말 것 | Aesop 입력 시 배경 컬러가 해당 브랜드로 오염됨 |
| Negative 문장 쓰지 말 것 | Gamma 스타일 필드는 positive style guide로만 작동 |
| 피사체를 명시적으로 지정 | 미지정 시 골드 오브젝트·3D 렌더로 이탈 |
| 3D 제거는 예외적으로 직접 명시 | "No 3D renders, no objects" — 강하게 이탈할 때만 |

### 르샤인 Le Shine — 확정
```
Photorealistic beauty photography only.
Subject: young korean woman, clean natural makeup, soft elegant features, glass skin, calm expression.
Single subject, minimal composition, no props or floating elements.
Warm golden tones, soft diffused backlight, luminous skin glow.
Clean cream or soft beige seamless background, generous negative space.
Premium skincare ad feel — quiet, refined, effortless.
```

### 오블리브 Obliv — 확정
```
Photorealistic beauty photography only.
Subject: young korean woman, soft natural makeup, warm matte skin, calm healing expression.
Single subject, minimal composition, no props or floating elements.
Warm amber tones, intimate dim candlelike light, soft shadows on face.
Clean warm beige or dark wood-tone seamless background, generous negative space.
Premium wellness clinic feel — grounded, warm, restorative.
```

### 온리프 Onlif — 확정
```
Photorealistic beauty photography only.
Subject: young korean woman, clean minimal makeup, natural result skin, fresh calm expression.
Single subject, minimal composition, no props or floating elements.
Soft blue-grey tones, clean neutral studio light, airy bright atmosphere.
Clean off-white or soft pale blue seamless background, generous negative space.
Premium plastic surgery clinic feel — trustworthy, refined, quietly confident.
```

### 3개 브랜드 비교

| | 르샤인 | 오블리브 | 온리프 |
|--|-------|---------|-------|
| 인물 톤 | glass skin, elegant | warm matte skin | natural result skin |
| 조명 | warm golden backlight | amber candlelike | clean neutral studio |
| 배경 | cream / soft beige | warm beige / dark wood | off-white / pale blue |
| 무드 | radiant, effortless | grounded, restorative | trustworthy, confident |

### 제형 텍스처 전용 (브랜드 공통)
인물과 제형은 슬라이드를 분리해서 쓸 것. 같은 슬라이드에 넣으면 splash 합성 이미지로 이탈함.
```
Photorealistic skincare product texture photography only.
Subject: single drop or pool of golden serum on clean surface. No bottles, no props.
Warm golden tones, soft studio light, macro close-up.
Clean minimal background, generous negative space.
Premium beauty editorial texture — luminous, refined.
```

---

## 파일 목록

```
prompt-dashboard.html   ← 현재 완성된 UI 프로토타입 (이걸 기반으로 시작)
HANDOFF.md              ← 이 문서 (v0.2)
```

---

## Claude Code 시작 프롬프트 (복붙용)

```
아래 handoff 문서와 HTML 파일을 기반으로 브랜드 프롬프트 대시보드를 개발해줘.

현재 상태:
- IBM Carbon 디자인 스타일 HTML 프로토타입 완성
- 브랜드 DB (leshine/obliv/onlif), 허용 풀, 프롬프트 조합 로직 구현됨
- 정석/변주 모드, 카테고리 6개, 파라미터 8+1개

Phase 1 작업:
1. 파일 구조 분리: brandDB.json / promptTemplates.json / index.html / main.js
2. 참조 이미지 업로드 기능 추가 (base64 미리보기 + identity_lock 토글)
3. 프롬프트 히스토리 localStorage 저장 (최근 20개)
4. 카테고리 시술전/후/비교 파라미터 세트 분리

스택: Vanilla JS 유지 or Vite + Vanilla 으로 가도 됨. React 불필요.
디자인: IBM Carbon 스타일 그대로 유지.
```

---

*Medibuilder DoAi TF — Design Module*  
*작성: claude.ai 대화 세션 → Claude Code 이관*
