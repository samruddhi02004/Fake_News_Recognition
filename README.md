# Fake News Recognition

This repository contains a legacy end-to-end fake news classification project. The codebase combines:

- a Scrapy crawler for collecting raw news pages
- preprocessing utilities for turning scraped HTML into article text
- deep learning training scripts built around FastText embeddings
- a Flask API that serves binary and multi-class predictions
- an Angular 5 frontend for interactive demos

The main application source lives in `FakeNewsRecognition/`. The top-level repository currently wraps that source tree and this README documents how the full project is structured.

## Project status

This is a research-style prototype rather than a polished production system.

- The stack is old: Angular 5, older Keras/TensorFlow APIs, Python 3.5-era conventions, PostgreSQL 9.5 in Docker, and several hard-coded paths.
- The repository does not include the training datasets, populated embeddings database, or model weight files.
- Some scripts assume Linux tooling such as `shuf`, and the API code assumes container-style paths under `/src`.

If you want to run the project without modifying code, Docker is the most practical path.

## What the project does

At a high level, the system follows this pipeline:

1. Crawl configured news domains with Scrapy and store raw HTML in SQLite.
2. Parse the stored HTML with `newspaper3k` to extract cleaned article content and metadata.
3. Preprocess and split the cleaned data into train, validation, and test JSONL files.
4. Train binary and multi-class neural models using 100-dimensional FastText embeddings.
5. Serve predictions through a Flask API and visualize them in the Angular frontend.

## Repository layout

```text
.
|-- README.md
|-- LICENSE
`-- FakeNewsRecognition/
    |-- api/              Flask inference API, uWSGI, nginx, Docker helpers
    |-- front/            Angular 5 demo frontend
    |-- machines/         Model definitions, generators, and training scripts
    |-- misc/             One-off utilities, notebooks, and evaluation scripts
    |-- preprocessing/    HTML-to-article parsing and cleaned dataset creation
    |-- spider/           Scrapy crawler and persistence pipeline
    |-- Dockerfile        Container image for API + nginx
    `-- requirements.txt Python dependencies
```

## Models and outputs

The repository contains code for multiple architectures under `FakeNewsRecognition/machines/`, including:

- BiLSTM binary classifier
- BiLSTM multi-class classifier
- TCN multi-class classifier
- additional experimental CNN/LSTM variants

The serving API uses two models:

- a binary BiLSTM to estimate whether an article is `fake` or `true`
- a multi-class TCN to score these labels:
  `bias`, `clickbait`, `conspiracy`, `fake`, `hate`, `junksci`, `political`, `reliable`, `rumor`, `satire`, `unreliable`

Input text is tokenized and mapped into a `(300, 100)` embedding tensor before inference.

## External dependencies and prerequisites

You will need more than what is stored in Git:

- PostgreSQL with a `word_embeddings` table available to the API
- model weight files for binary and multi-class inference
- training and preprocessing datasets under a local `data/` directory
- outbound access to S3 if you rely on the API's automatic model download
- Node.js/npm if you want to run the Angular frontend

The checked-in `requirements.txt` is only a partial Python dependency list. For the full legacy runtime footprint, use the Docker image or cross-check the packages installed in `FakeNewsRecognition/Dockerfile`.

Important environment variables used by the code:

| Variable | Used by | Purpose |
| --- | --- | --- |
| `FNR_DB_NAME` | `api/database.py` | PostgreSQL database name |
| `FNR_DB_HOST` | `api/database.py` | PostgreSQL host |
| `FNR_DB_USER` | `api/database.py` | PostgreSQL user |
| `FNR_DB_PASSWORD` | `api/database.py` | PostgreSQL password |
| `FNR_PATH_DATA` | `machines/data_generator.py` | Base path for training data and FastText files |
| `PATH_FNR` | `spider/news_spider/spiders/__init__.py` | Project root path used by crawler utilities |

Notes:

- `PATH_FNR` is concatenated with relative strings in the spider code, so give it a trailing slash.
- The API expects model files under `/src/data/` when run unmodified.
- The frontend currently posts to `http://fakenewsrecognition.com/api/v1/predict`, so you must change that URL for local development.

## Running the API with Docker

The Docker image is the least surprising way to run the API because the container matches the code's `/src/...` path assumptions.

From inside `FakeNewsRecognition/`:

```bash
docker build -t fake-news-recognition .
docker run --rm -p 8080:80 \
  -e FNR_DB_NAME=your_db \
  -e FNR_DB_HOST=your_db_host \
  -e FNR_DB_USER=your_db_user \
  -e FNR_DB_PASSWORD=your_db_password \
  fake-news-recognition
```

What happens on startup:

- nginx is started in the container
- uWSGI serves the Flask app from `api/server.py`
- if `/src/data/binary_model.hdf5` or `/src/data/multiclass_model.hdf5` is missing, the API tries to download weights from the S3 bucket `fake-news-recognition`

Once the container is up, the API is exposed on port `8080` on the host.

## Running the API locally

Local execution is possible, but expect to adjust paths or recreate the original environment.

From inside `FakeNewsRecognition/`:

```bash
pip install -r requirements.txt
python api/server.py
```

Before doing that, you must make sure:

- you have installed the extra runtime packages that are only listed in `Dockerfile`
- the PostgreSQL environment variables are exported
- the `word_embeddings` table is populated
- model files are available where the code expects them, or the process can reach S3
- your environment is compatible with the older Keras/TensorFlow code

For most users, Docker will be easier than trying to run the API unmodified on a modern host.

## API reference

### `GET /api/`

Returns a simple welcome payload.

### `GET /api/v1`

Returns the same welcome payload as `/api/`.

### `POST /api/v1/predict`

Accepts JSON with:

- `url` (required)
- `title` (optional)
- `content` (optional)

If `title` and `content` are omitted, the API downloads the article from the supplied URL with `newspaper3k`.

Example request:

```json
{
  "url": "https://example.com/article",
  "title": "Example headline",
  "content": "Example article body..."
}
```

Example response:

```json
{
  "status": 200,
  "data": {
    "fake": 0.83,
    "prediction": "fake",
    "classes": {
      "bias": 0.02,
      "clickbait": 0.05,
      "conspiracy": 0.04,
      "fake": 0.61,
      "hate": 0.01,
      "junksci": 0.00,
      "political": 0.07,
      "reliable": 0.09,
      "rumor": 0.06,
      "satire": 0.03,
      "unreliable": 0.02
    }
  }
}
```

Additional API behavior:

- CORS is enabled for all origins.
- Binary output is reported both as a `fake` score and a `prediction` string.
- The multi-class output is returned as label-to-score pairs.

## Frontend

The demo frontend lives in `FakeNewsRecognition/front/` and was generated with Angular CLI 1.7.3.

To run it:

```bash
cd front
npm install
npm start
```

Before starting the UI, update the API URL in `front/src/app/app.component.ts`. It is hard-coded to the public deployment:

```ts
this.http.post('http://fakenewsrecognition.com/api/v1/predict', ...)
```

Useful frontend commands:

- `npm start` - development server
- `npm run build` - production build
- `npm test` - unit tests
- `npm run e2e` - end-to-end tests

## Crawling and preprocessing workflow

### 1. Crawl raw pages

The Scrapy project lives in `FakeNewsRecognition/spider/`.

From `FakeNewsRecognition/spider/`:

```bash
scrapy crawl news -a websites_start=0 -a websites_end=50
```

The crawler:

- reads domain lists from spreadsheet files under `data/7_opensources_co/`
- follows links within the configured domains
- stores raw HTML in `news_spider.db`
- deduplicates URLs before insertion

### 2. Parse raw HTML into cleaned articles

From `FakeNewsRecognition/`:

```bash
python preprocessing/__init__.py
```

This stage:

- reads from the raw crawl SQLite database
- parses articles with `newspaper3k`
- deduplicates by URL and content
- writes cleaned article records and metadata into a destination SQLite database

### 3. Prepare training splits

Dataset preparation utilities live in `machines/data_generator.py`.

That module can:

- preprocess text into token sequences
- remove duplicate content
- create binary and multi-class dataset variants
- shuffle and split data into train, validation, and test JSONL files
- load or export FastText embeddings

The current `__main__` entrypoint calls `prepare_all_separate_data()`. Other preparation flows are available as functions in the same file.

### 4. Train models

Training scripts live in `FakeNewsRecognition/machines/`.

Common entrypoints include:

- `python machines/bilstm.py`
- `python machines/bilstm_all.py`
- `python machines/tcn_all.py`

These scripts expect preprocessed JSONL files and FastText embeddings under `FNR_PATH_DATA`.

## Embeddings and helper scripts

The API embeds words by reading vectors from PostgreSQL, not directly from a local FastText file. Relevant helper scripts in `FakeNewsRecognition/misc/` include:

- `2018-02-22 - fasttext_to_sql_embedding.py` - writes embeddings into a local database representation
- `2018-03-11 - fasttext convert.py` - converts FastText vectors into JSONL
- `2018-04-01 - evaluate_model_on_4.py` - ad-hoc evaluation script on an external fake/real dataset

The `misc/notebooks/` directory also contains exploratory notebooks used during experimentation.

## Known limitations

- No end-to-end setup script is included for provisioning data, embeddings, and model artifacts.
- The frontend is wired to a production URL rather than environment-based configuration.
- Several scripts rely on hard-coded dated filenames and local spreadsheet/database layouts.
- Training code assumes older TensorFlow/Keras APIs and may require version pinning or containerization.
- The repository does not include an automated test suite for the Python backend or model training pipeline.

## License

This project is released under the MIT License. See `LICENSE` for details.
