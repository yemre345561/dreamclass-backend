# DreamClass backend

Flask backend for DreamClass, a prototype that asks Google Gemini to generate a short, story-based lesson.

You send a theme (called a "universe" in the API), a topic, and a grade or level. The service is meant to return a story, a quiz, and short descriptions of visual elements. Example prompts in `prompts/demo_prompts.py` cover Star Wars physics and environmental science, and Harry Potter math and history.

This repository is the backend only. There is no separate frontend app here, apart from a simple test page.

## Features in the current code

- Lesson generation through Google Gemini. `config.py` names `gemini-1.5-flash` and points `GEMINI_API_URL` at that model. `services/gemini_api.py`, which `main.py` uses, calls the `google-generativeai` client with `gemini-1.5-pro`. `services/lesson_service.py` posts to the `gemini-1.5-flash` URL and also sends `"model": "gemini-pro"` in the JSON body.
- JSON built around `story` and `quiz`. The `main.py` prompt asks for `visualIdea` (a string). `app.py` and `api/generate.py` ask for `visual_elements` (a list). `lesson_service` also tries to store a `url` on each image element by posting the description to another Gemini `generateContent` URL.
- A file-based cache class in `services/cache.py` for universe, topic, and grade. No request path imports it, so repeated calls are not served from that cache.
- Draft image and animation helpers in `services/media_service.py`. They are written to post a text prompt to a Gemini `generateContent` URL and treat the text reply as a media URL. They are not an image or video pipeline. The module references `IMAGEFX_PROMPT_TEMPLATE`, `VEO2_PROMPT_TEMPLATE`, and `IMAGE_API_URL` without importing them. Calling either helper hits `NameError` before any HTTP call. The method catches that and returns `{"success": false, "error": ...}`.
- A small test UI at `/` when you start `main.py` (`templates/test.html`).
- Tests under `tests/`, written with `unittest`. `pytest` can collect them. `tests/test_lesson_service.py` mocks HTTP. `tests/test_ai_integration.py` calls the live Gemini API and is not mocked.

## API

There are two entry points, and their field names are not identical.

`main.py` (includes the test page and CORS for `/api/*`):

```bash
python main.py
# GET  /              test HTML form
# POST /api/generate  JSON: {"universe": "...", "topic": "...", "level": "..."}
```

That handler returns `error`, `message`, and `content`. On success, `content` is the parsed Gemini object (`story`, `quiz`, `visualIdea`). `main.py` listens on port 5000 by default (`PORT`) and on Flask's default host, so the test page is http://127.0.0.1:5000.

`app.py` registers the blueprint in `api/generate.py` and also defines its own `/api/generate` route. Both expect `universe`, `topic`, and `grade` (not `level`). The blueprint is registered first, so Flask matches that rule for `POST /api/generate`. The async view defined later on the app is a second registration of the same path.

```bash
python app.py
# GET  /              plain "DreamClass Backend is running!"
# POST /api/generate  JSON: {"universe": "...", "topic": "...", "grade": "..."}
```

The blueprint calls async `lesson_service.generate_lesson` without `await`, then calls `.get` on the coroutine. It also exposes `POST /api/generate/image` and `POST /api/generate/animation`. The view defined on `app.py` awaits `generate_lesson` and returns `story`, `quiz`, and `visual_elements` at the top level.

For a first local try, `main.py` is the path with the test page. It needs a `GOOGLE_API_KEY` that `services/gemini_api.py` accepts (the client checks that the key is set and starts with `AIza`) because it builds the Gemini client at import time.

## Setup

```bash
git clone https://github.com/yemre345561/dreamclass-backend.git
cd dreamclass-backend
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
# set GOOGLE_API_KEY in .env
python main.py
```

Then open http://127.0.0.1:5000.

`.env` is gitignored. Do not commit a real API key.

The packages this code imports in order to start are Flask, google-generativeai, python-dotenv, flask-cors, and requests. `requirements.txt` also pins libraries the current modules do not import, including python-multipart, python-jose, passlib, bcrypt, aiohttp, and uvloop, plus pytest and the listed formatters and linters.

## Tests

From the repo root, with the virtualenv active:

```bash
pytest tests/test_lesson_service.py
```

`tests/test_ai_integration.py` performs a real Gemini request. A full `pytest` run includes it and needs a working key plus network access. The error-path tests in `tests/test_lesson_service.py` expect `generate_lesson` to return an error object. The service raises instead, so those tests fail as the code stands.

## Layout

```
main.py                 # test UI + /api/generate (universe, topic, level)
app.py                  # alternate app; also registers /api/generate (grade)
config.py               # model name, prompts, host/port
api/generate.py         # blueprint: lesson, image, and animation routes
services/gemini_api.py  # google-generativeai client used by main.py
services/lesson_service.py
services/media_service.py
services/cache.py       # file cache; not imported by the request path
services/error_handler.py
templates/test.html
tests/
data/prompt_template.json
data/error_responses.json
prompts/demo_prompts.py
gemini_test.py          # manual script: list models and call Gemini
test_api.py             # manual script: post an image prompt to Gemini
.env.example
```

## Status

Prototype backend. The two `/api/generate` entry points disagree on field names, model ids, and response shape, and should be unified before anyone else relies on the API. The image and animation helpers should not be treated as media generation.
