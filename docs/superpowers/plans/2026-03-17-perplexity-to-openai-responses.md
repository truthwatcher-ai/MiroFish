# Replace Perplexity with OpenAI Responses API

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace Perplexity Sonar API calls in the seed generator with OpenAI Responses API + `web_search` tool, using gpt-5-mini (quick) and gpt-5 (thorough).

**Architecture:** Only `seed_generator.py` and `config.py` change. The `generate_file()` method swaps `self.perplexity.chat.completions.create()` for `self.llm.client.responses.create()` with the `web_search` tool. The Perplexity client and config are removed. Frontend is untouched.

**Tech Stack:** OpenAI Python SDK (already installed), Responses API with `web_search` tool.

---

## Chunk 1: Replace Perplexity with OpenAI Responses API

### Task 1: Update config.py

**Files:**
- Modify: `backend/app/config.py`

- [ ] **Step 1: Replace Perplexity config with seed model config**

Replace lines 42-45:
```python
# Perplexity API (for seed data generation with web search)
PERPLEXITY_API_KEY = os.environ.get('PERPLEXITY_API_KEY')
PERPLEXITY_MODEL = os.environ.get('PERPLEXITY_MODEL', 'sonar-pro')
PERPLEXITY_MODEL_DEEP = os.environ.get('PERPLEXITY_MODEL_DEEP', 'sonar-deep-research')
```

With:
```python
# Seed data generation (uses OpenAI Responses API with web_search tool)
SEED_MODEL_QUICK = os.environ.get('SEED_MODEL_QUICK', 'gpt-5-mini')
SEED_MODEL_THOROUGH = os.environ.get('SEED_MODEL_THOROUGH', 'gpt-5')
```

- [ ] **Step 2: Commit**

```bash
git add backend/app/config.py
git commit -m "refactor(config): replace Perplexity config with OpenAI seed model config"
```

---

### Task 2: Rewrite seed_generator.py

**Files:**
- Modify: `backend/app/services/seed_generator.py`

- [ ] **Step 1: Update module docstring and imports**

Replace the docstring and remove the Perplexity-specific import:
```python
"""
Seed data generator service.

Uses gpt-4.1-mini to analyze topics into research categories,
then OpenAI Responses API with web_search to generate research files.
"""

import os
import json
import uuid
import threading
from typing import Optional

from ..config import Config
from ..utils.llm_client import LLMClient
from ..utils.logger import get_logger
from ..i18n import get_prompt

logger = get_logger('arus.seed')
```

Note: Remove `from openai import OpenAI` — no longer needed since we use `self.llm.client` which is already an OpenAI instance.

- [ ] **Step 2: Rewrite `__init__` method**

Replace the existing `__init__` that creates a Perplexity client:

```python
def __init__(self):
    self.llm = LLMClient()
    self.output_dir = os.path.join(Config.UPLOAD_FOLDER, 'seed')
    os.makedirs(self.output_dir, exist_ok=True)
```

This removes:
- `Config.PERPLEXITY_API_KEY` check
- `self.perplexity = OpenAI(...)` client creation

- [ ] **Step 3: Rewrite `generate_file` method**

Replace the Perplexity chat.completions call with OpenAI Responses API:

```python
def generate_file(self, prompt: str, category: dict, depth: str) -> str:
    """Use OpenAI Responses API with web_search to generate one seed file."""
    model = Config.SEED_MODEL_THOROUGH if depth == 'thorough' else Config.SEED_MODEL_QUICK
    system_prompt = get_prompt('seed_generate_prompt')

    user_message = (
        f"Topic: {prompt}\n\n"
        f"Category: {category['name']}\n"
        f"Description: {category.get('description', '')}\n\n"
        f"Write a comprehensive research document about this category "
        f"as it relates to the topic above."
    )

    logger.info(f'Generating seed file: {category["name"]} (model={model})')

    response = self.llm.client.responses.create(
        model=model,
        tools=[{'type': 'web_search'}],
        input=[
            {'role': 'system', 'content': system_prompt},
            {'role': 'user', 'content': user_message},
        ],
    )

    content = response.output_text
    logger.info(f'Generated {len(content)} chars for {category["name"]}')
    return content
```

Key differences from old code:
- `self.llm.client.responses.create()` instead of `self.perplexity.chat.completions.create()`
- `tools=[{'type': 'web_search'}]` enables web search
- `input=[...]` instead of `messages=[...]` (Responses API uses `input`)
- `response.output_text` instead of `response.choices[0].message.content`

- [ ] **Step 4: Verify no other references to Perplexity remain**

Search for any remaining Perplexity references in the file. The `analyze_topic()`, `start_generation()`, `_generate_all()`, `get_task()`, `get_file_content()`, and `get_file_path()` methods should be unchanged.

- [ ] **Step 5: Commit**

```bash
git add backend/app/services/seed_generator.py
git commit -m "feat(seed): replace Perplexity with OpenAI Responses API web_search"
```

---

### Task 3: Clean up .env

**Files:**
- Modify: `.env` (project root)

- [ ] **Step 1: Remove or comment out Perplexity key**

Remove or comment out `PERPLEXITY_API_KEY` line. Optionally add seed model overrides (not required since defaults are set in config):

```env
# Seed generation models (optional, defaults: gpt-5-mini / gpt-5)
# SEED_MODEL_QUICK=gpt-5-mini
# SEED_MODEL_THOROUGH=gpt-5
```

- [ ] **Step 2: Commit**

```bash
git add .env
git commit -m "chore: remove Perplexity API key from env"
```

---

### Task 4: Test

- [ ] **Step 1: Restart backend**

```bash
# Kill existing backend and restart
lsof -ti:5001 | xargs kill -9
cd /Users/truthwatcher/Documents/MiroFish && npm run backend
```

- [ ] **Step 2: Test seed generation flow**

1. Open http://localhost:3000
2. Enter a prompt, click Launch Engine (no files)
3. Modal appears with categories
4. Select categories, choose Quick depth
5. Click Start Research
6. Verify progress updates in modal
7. Verify files are generated and can be previewed
8. Test with Thorough depth

- [ ] **Step 3: Test FAB progress ring**

1. Start generation, close modal
2. Verify FAB shows progress ring
3. Click FAB → Research segment → modal reopens with progress

- [ ] **Step 4: Push**

```bash
git push origin feat-001-circular-fab-menu
```
