# 📚 API Reference

본 문서는 Anki Flash Card Generator 프로젝트의 **공개(Public) API** 전체를 정리한 문서입니다. 각 모듈-별 주요 클래스·함수의 시그니처, 동작, 파라미터, 반환값, 예제 코드를 포함합니다.

> 모든 예제는 Python 3.10 이상에서 동작하도록 작성되었습니다. 프로젝트 루트를 `PYTHONPATH`에 추가한 뒤 실행하거나, 예제와 동일한 디렉터리에서 `python -m` 형식으로 실행해 주세요.

---

## 목차

1. [Config](#config)
2. [Entity](#entity)
3. [Utils](#utils)
4. [Service](#service)
   1. [LLMService](#llmservice)
   2. [FileReaderService](#filereaderservice)
   3. [FlashcardGeneratorService](#flashcardgeneratorservice)
   4. [ExportService](#exportservice)
5. [Main Application](#main-application)
6. [Command-line 데모 & 테스트 스크립트](#command-line-데모--테스트-스크립트)

---

<a id="config"></a>
## 1. Config

### `LLMConfig`
`src/Config/llm_config.py`

Large Language Model(LLM) 호출에 필요한 모든 환경 변수를 수집·보관합니다.

```python
from src.Config.llm_config import LLMConfig

config = LLMConfig()
print(config.provider)         # 환경 변수 `LLM_PROVIDER` 기본값은 'ollama'
print(config.max_retries)      # 재시도 횟수
```

| 속성 | 타입 | 설명 |
|------|------|------|
| provider | `str` | 사용 LLM 제공자(openai, ollama, openrouter) |
| openai_api_key / openai_model | `str` | OpenAI 사용 시 필요 |
| ollama_base_url / ollama_model | `str` | Ollama 사용 시 필요 |
| openrouter_api_key / openrouter_model | `str` | OpenRouter 사용 시 필요 |
| max_retries / retry_delay | `int` | API 재시도 설정 |
| temperature / max_tokens | `float / int` | 생성 파라미터 |
| cards_per_section | `int` | 섹션당 생성 카드 수 |
| min_card_quality | `float` | LLM 품질 평가 최소 점수(0-1) |

---

<a id="entity"></a>
## 2. Entity

### `Flashcard`
`src/Entity/flashcard.py`

```python
from src.Entity.flashcard import Flashcard

card = Flashcard(
    question="What is Machine Learning?",
    answer="A subset of AI that allows systems to learn patterns from data.",
    tags=["AI", "ML"]
)

print(card.to_anki_format())  # Anki TSV 출력
print(card.is_valid())        # True
```

| 메서드 | 반환 | 설명 |
|---------|-------|-------|
| `to_anki_format()` | `str` | 탭(`\t`) 구분 텍스트(Q, A, Tags) |
| `is_valid()` | `bool` | Q/A 모두 존재 여부 |
| `calculate_quality_score(llm_client=None)` | `float` | LLM으로 카드 품질(0~1) 산출 |

---

<a id="utils"></a>
## 3. Utils

### `TextProcessor`
`src/Utils/text_processor.py`

문자열 토큰 계산, 텍스트 분할, 키워드 추출 등의 헬퍼 메서드를 제공합니다.

```python
from src.Utils.text_processor import TextProcessor

text = "Artificial Intelligence (AI) makes machines intelligent."
print(TextProcessor.estimate_tokens(text))    # 토큰 개수 추정

sections = TextProcessor.smart_divide_text(text, max_tokens=20)
print(sections)

concepts = TextProcessor.extract_key_concepts(text)
print(concepts)  # ['Artificial Intelligence']
```

| 정적 메서드 | 설명 |
|--------------|------|
| `estimate_tokens(text, model='gpt-3.5-turbo')` | 문자열을 토큰화해 개수를 반환 |
| `smart_divide_text(text, max_tokens=1500)` | 최대 토큰 수 단위로 의미 있게 분할한 리스트 반환 |
| `extract_key_concepts(text)` | 대문자 시작 구 표시어를 찾아 상위 10개 반환 |
| `clean_text(text)` | 공백·특수문자 정리 후 반환 |

---

<a id="service"></a>
## 4. Service

서비스 계층은 인터페이스(`src/IService/*`)를 구현하며, 다른 계층에 의존성을 주입받아 동작합니다.

> 내부(private) 메서드(`_로 시작`)는 생략했습니다.

<a id="llmservice"></a>
### 4.1 `LLMService`
`src/Service/llm_service.py`

통합 LLM 호출 서비스. 재시도 로직과 제공자별 API 래퍼를 포함합니다.

```python
from src.Config.llm_config import LLMConfig
from src.Service.llm_service import LLMService

config = LLMConfig()
llm = LLMService(config)

messages = [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "Say hello"}
]
print(llm.call_api_with_retry(messages))
```

| 메서드 | 설명 |
|---------|------|
| `call_api_with_retry(messages: List[Dict]) -> str` | 실패 시 `max_retries`까지 재시도해 응답 반환 |
| `generate_prompt(system_prompt, user_prompt) -> List[Dict]` | 두 문자열을 Chat API 포맷으로 변환 |

<a id="filereaderservice"></a>
### 4.2 `FileReaderService`
`src/Service/pdf_reader_service.py`

PDF · Markdown · 일반 텍스트 파일에서 본문과 메타데이터를 추출합니다.

```python
from src.Service.pdf_reader_service import FileReaderService

reader = FileReaderService()
text, meta = reader.read_file("docs/sample.pdf")
print(meta)
```

| 메서드 | 설명 |
|---------|------|
| `read_file(file_path: str) -> Tuple[str, Dict]` | 파일 형식에 따라 `_read_pdf/_read_markdown/_read_text` 분기 후 결과 반환 |

<a id="flashcardgeneratorservice"></a>
### 4.3 `FlashcardGeneratorService`
`src/Service/flashcard_generator_service.py`

LLM을 사용해 텍스트(또는 PDF 전체)를 Anki 플래시카드로 변환하는 핵심 로직을 담고 있습니다.

```python
from src.Config.llm_config import LLMConfig
from src.Service.llm_service import LLMService
from src.Service.pdf_reader_service import FileReaderService
from src.Service.flashcard_generator_service import FlashcardGeneratorService

config = LLMConfig()
llm = LLMService(config)
reader = FileReaderService()
generator = FlashcardGeneratorService(llm, reader, config)

cards = generator.generate_cards_from_pdf("docs/your_doc.pdf", process_all=False)
print(len(cards))
```

| 메서드 | 설명 |
|---------|------|
| `generate_cards_from_section(text, context) -> List[Flashcard]` | 텍스트 섹션 1개에서 카드 n개 생성·유효성/품질 체크 |
| `generate_cards_from_pdf(file_path, process_all=False) -> List[Flashcard]` | 파일에서 섹션 분할 후 병렬로 카드 생성 |

<a id="exportservice"></a>
### 4.4 `ExportService`
`src/Service/export_service.py`

생성된 `Flashcard` 리스트를 원하는 형식으로 저장합니다.

```python
from src.Service.export_service import ExportService
from src.Entity.flashcard import Flashcard

cards = [Flashcard("Q?", "A.")]
exporter = ExportService()
exporter.export_to_csv(cards, "output/cards.csv")
```

| 메서드 | 설명 |
|---------|------|
| `export_to_anki_txt(cards, output_path)` | 탭 구분(.txt) 형식(Anki용) 저장 |
| `export_to_csv(cards, output_path)` | CSV 저장 |
| `export_to_json(cards, output_path)` | JSON 저장 |

---

<a id="main-application"></a>
## 5. Main Application

### `AnkiFlashcardMaker`
`src/main.py`

```python
from src.main import AnkiFlashcardMaker

maker = AnkiFlashcardMaker()
# PDF 1개 처리
cards = maker.process_file("SOURCE_DOCUMENTS/intro.pdf", process_all=False)
maker.save_flashcards(cards, "intro")
```

| 메서드 | 설명 |
|---------|------|
| `process_file(file_path, process_all=False)` | 파일 1개를 읽어 카드 리스트 반환 |
| `save_flashcards(cards, base_name)` | `output/` 디렉터리에 Anki, CSV, JSON 동시 저장 |
| `get_supported_files(source_dir)` | 지정 폴더에서 지원 확장자 파일 찾기 |
| `generate_statistics(cards)` | 카드 개수, 태그 분포 등 통계 dict 반환 |

> 콘솔 실행: `python -m src.main` 혹은 `python src/main.py`

---

<a id="command-line-데모--테스트-스크립트"></a>
## 6. Command-line 데모 & 테스트 스크립트

| 스크립트 | 설명 |
|-----------|------|
| `demo_multi_provider.py` | 동일 입력 텍스트를 세 가지 LLM(OpenAI, Ollama, OpenRouter)로 테스트 |
| `test_llm_providers.py` | 현재 또는 모든 provider 연결 테스트 |
| `run.sh` / `run.bat` | *빠른 시작* 실행 스크립트 – `.env` 설정 후 더블클릭 or 터미널 실행 |

예)

```bash
# 모든 provider 연결 테스트 (비인터랙티브 환경)
python test_llm_providers.py --all

# Ollama 모델로 데모 실행
export LLM_PROVIDER=ollama
python demo_multi_provider.py
```

---

## 부록: 인터페이스(Interface) 정의

각 서비스는 `src/IService/*` 의 추상 클래스(ABC)를 구현합니다. DI(의존성 주입)를 통해 테스트 가능성을 높이고, 새 구현체 추가를 간소화합니다.

```
ILLMService                -> LLMService
IFileReaderService         -> FileReaderService
IFlashcardGeneratorService -> FlashcardGeneratorService
IExportService             -> ExportService
```

---

### 문서 최종 갱신: <!--DATE_PLACEHOLDER-->