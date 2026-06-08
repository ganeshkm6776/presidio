# Building Custom Docker Images for Presidio

This guide explains how to build custom Presidio Docker images — for example, to add
multi-language NLP support, use alternative models, or include custom recognizers.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) installed and running
- [Git](https://git-scm.com/) to clone the repository
- At least **4 GB of free disk space** (NLP models are large; 10+ languages may require 8–16 GB)

---

## Overview: What Controls a Custom Image

Three YAML files in `presidio-analyzer/presidio_analyzer/conf/` drive the build:

| File | Purpose |
|------|---------|
| `default.yaml` | NLP engine + language models to download |
| `default_analyzer.yaml` | Analyzer service settings |
| `default_recognizers.yaml` | Which PII recognizers are enabled |

The `Dockerfile` accepts these as **build arguments**, so you can swap in your own
config files without modifying the Dockerfile itself.

---

## Step 1 — Clone the Repository

```bash
git clone https://github.com/microsoft/presidio.git
cd presidio
```

---

## Step 2 — Choose or Create an NLP Config

The `conf/` directory ships with several ready-made configs:

| File | Engine | Languages |
|------|--------|-----------|
| `default.yaml` | spaCy | English only (`en_core_web_lg`) |
| `spacy_multilingual.yaml` | spaCy | English, German, Spanish |
| `slim_nlp.yaml` | slim (lightweight) | English only (`en_core_web_sm`) |
| `stanza_multilingual.yaml` | Stanza | Multiple |
| `transformers.yaml` | HuggingFace Transformers | Multiple |

### Example: Add French and Italian to spaCy

Create `presidio-analyzer/presidio_analyzer/conf/my_nlp_config.yaml`:

```yaml
nlp_engine_name: spacy
models:
  - lang_code: en
    model_name: en_core_web_lg
  - lang_code: fr
    model_name: fr_core_news_md
  - lang_code: it
    model_name: it_core_news_md
  - lang_code: de
    model_name: de_core_news_md

ner_model_configuration:
  model_to_presidio_entity_mapping:
    PER: PERSON
    PERSON: PERSON
    NORP: NRP
    FAC: LOCATION
    LOC: LOCATION
    GPE: LOCATION
    LOCATION: LOCATION
    ORG: ORGANIZATION
    ORGANIZATION: ORGANIZATION
    DATE: DATE_TIME
    TIME: DATE_TIME
  low_confidence_score_multiplier: 0.4
  low_score_entity_names: []
  labels_to_ignore:
    - CARDINAL
    - EVENT
    - LANGUAGE
    - LAW
    - MONEY
    - ORDINAL
    - PERCENT
    - PRODUCT
    - QUANTITY
    - WORK_OF_ART
```

> **Available spaCy model names:** Find them at [spacy.io/models](https://spacy.io/models).
> Common ones: `fr_core_news_md`, `it_core_news_md`, `de_core_news_md`,
> `pt_core_news_md`, `zh_core_web_md`, `ja_core_news_md`.

---

## Step 3 — Build the Custom Image

Pass your custom config file as a build argument using `--build-arg`:

```bash
docker build \
  --build-arg NLP_CONF_FILE=presidio_analyzer/conf/my_nlp_config.yaml \
  -t presidio-analyzer-multilingual:latest \
  ./presidio-analyzer
```

### Build Arguments Reference

| Argument | Default | Description |
|----------|---------|-------------|
| `NLP_CONF_FILE` | `presidio_analyzer/conf/default.yaml` | NLP engine + models |
| `ANALYZER_CONF_FILE` | `presidio_analyzer/conf/default_analyzer.yaml` | Analyzer settings |
| `RECOGNIZER_REGISTRY_CONF_FILE` | `presidio_analyzer/conf/default_recognizers.yaml` | Enabled recognizers |

---

## Step 4 — Run the Custom Image

```bash
docker run -d \
  -p 5002:3000 \
  --name presidio-analyzer-custom \
  presidio-analyzer-multilingual:latest
```

Test it with a French text sample:

```bash
curl -X POST http://localhost:5002/analyze \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Mon numéro de sécurité sociale est 2 85 06 75 104 182",
    "language": "fr"
  }'
```

---

## Step 5 — Using Docker Compose (Recommended)

To integrate your custom image into the full Presidio stack, override the build
args in `docker-compose.yml`:

```yaml
services:
  presidio-analyzer:
    build:
      context: ./presidio-analyzer
      args:
        NLP_CONF_FILE: presidio_analyzer/conf/my_nlp_config.yaml
    environment:
      - PORT=5001
    ports:
      - "5002:5001"
```

Then build and start:

```bash
docker-compose up --build -d presidio-analyzer
```

---

## Troubleshooting

### ⚠ Warning: NLP recognizer is not in the list of recognizers for language `en`

This warning appears when you add language models but the recognizer registry
only has recognizers configured for `en`. It is **non-fatal** — Presidio will
still work for the new languages using NER-based detection.

To add pattern-based recognizers for a new language, edit
`default_recognizers.yaml` and add a `supported_languages` entry:

```yaml
- name: CreditCardRecognizer
  supported_languages:
    - language: en
      context: [credit, card, visa]
    - language: fr
      context: [carte, crédit, visa]
```

---

### ⚠ Docker image runs out of memory with 10+ languages

Each spaCy model is 50–500 MB. Loading 10+ models simultaneously can exhaust
memory during the build or at runtime.

**Recommendations:**

1. **Use smaller models** — swap `lg` → `md` or `sm`:
   ```yaml
   # Instead of en_core_web_lg (742 MB), use:
   model_name: en_core_web_sm   # 12 MB
   ```

2. **Increase Docker memory limit** — in Docker Desktop:
   *Settings → Resources → Memory* → set to at least `4 GB` for 5+ languages,
   `8 GB` for 10+ languages.

3. **Use a multilingual model** — one model covers many languages:
   ```yaml
   nlp_engine_name: spacy
   models:
     - lang_code: xx   # 'xx' = multilingual
       model_name: xx_ent_wiki_sm
   ```
   Install: `python -m spacy download xx_ent_wiki_sm`

4. **Use Stanza** — more memory-efficient for large language sets:
   ```yaml
   nlp_engine_name: stanza
   models:
     - lang_code: en
       model_name: en
     - lang_code: fr
       model_name: fr
     - lang_code: de
       model_name: de
   ```

---

### Build takes a very long time

The first build downloads all NLP models — this is expected.
Use Docker's build cache to speed up subsequent builds:

```bash
docker build \
  --build-arg NLP_CONF_FILE=presidio_analyzer/conf/my_nlp_config.yaml \
  --cache-from presidio-analyzer-multilingual:latest \
  -t presidio-analyzer-multilingual:latest \
  ./presidio-analyzer
```

---

## Full Example: French + German Custom Image

```bash
# 1. Clone
git clone https://github.com/microsoft/presidio.git && cd presidio

# 2. Create config
cat > presidio-analyzer/presidio_analyzer/conf/fr_de_nlp.yaml << 'EOF'
nlp_engine_name: spacy
models:
  - lang_code: en
    model_name: en_core_web_lg
  - lang_code: fr
    model_name: fr_core_news_md
  - lang_code: de
    model_name: de_core_news_md

ner_model_configuration:
  model_to_presidio_entity_mapping:
    PER: PERSON
    PERSON: PERSON
    LOC: LOCATION
    GPE: LOCATION
    ORG: ORGANIZATION
    DATE: DATE_TIME
  low_confidence_score_multiplier: 0.4
  low_score_entity_names: []
  labels_to_ignore:
    - CARDINAL
    - MONEY
    - PERCENT
EOF

# 3. Build
docker build \
  --build-arg NLP_CONF_FILE=presidio_analyzer/conf/fr_de_nlp.yaml \
  -t presidio-analyzer-fr-de:latest \
  ./presidio-analyzer

# 4. Run
docker run -d -p 5002:3000 presidio-analyzer-fr-de:latest

# 5. Test
curl -X POST http://localhost:5002/analyze \
  -H "Content-Type: application/json" \
  -d '{"text": "Mein Name ist Hans Müller", "language": "de"}'
```

---

## See Also

- [Installation guide](installation.md)
- [Supported entities](supported_entities.md)
- [Adding a custom recognizer](analyzer/adding_recognizers.md)
- [spaCy model list](https://spacy.io/models)
- [Stanza model list](https://stanfordnlp.github.io/stanza/available_models.html)
