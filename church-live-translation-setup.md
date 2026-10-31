# Church Live Translation --- Starter Setup

## Goal

Build the project in layers, starting with **text input** so we can
validate language detection and translation before connecting a
microphone.

Initial languages:

-   English (`en`)
-   Spanish (`es`)
-   Hebrew (`he`)

For now, each line in `samples.txt` represents text that a future
speech-to-text component (such as Whisper) would produce. We are **not**
installing speech recognition or audio capture yet.

## Architecture

``` text
samples.txt
    |
    v
Python processing script
    |
    +--> Detect source language
    |
    +--> Translate into the other two languages
    |
    v
Terminal output (first milestone)
    |
    v
Web caption screen (later milestone)

Future:
Microphone --> Whisper --> same text-processing pipeline
```

## Why this first stack

-   **Docker:** runs the translation service independently of Ubuntu's
    desktop environment.
-   **LibreTranslate:** self-hosted translation API, powered by Argos
    Translate.
-   **Python:** reads sample text, detects its language, and requests
    translations.
-   **No LLM container yet:** a general-purpose chat LLM is not
    necessary for this first milestone. We should first measure a
    dedicated translation engine on the MacBook Air 2015.

> Important: support for a language in the API does not guarantee that
> every translation direction is installed. We will inspect the
> service's `/languages` endpoint and test all six directions
> (English↔Spanish, English↔Hebrew, Spanish↔Hebrew) before choosing the
> final translation engine.

------------------------------------------------------------------------

# 1. Prerequisites

Run these commands on the Ubuntu Server over SSH.

``` bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2 python3 python3-venv python3-pip curl
sudo systemctl enable --now docker
```

Check Docker:

``` bash
docker --version
docker compose version
```

If `docker compose version` is unavailable, stop here and install the
Compose plugin using the current instructions for your Ubuntu release.
Do not substitute the old `docker-compose` command without checking
compatibility.

## Optional: run Docker without sudo

``` bash
sudo usermod -aG docker "$USER"
```

Log out of SSH and log back in for the group change to take effect. Then
verify:

``` bash
docker ps
```

If this does not work, use `sudo docker ...` for now.

------------------------------------------------------------------------

# 2. Create the project directory

``` bash
mkdir -p ~/church-translation
cd ~/church-translation
```

All project files in this guide will live in this directory.

------------------------------------------------------------------------

# 3. Pull and start the translation service

Pull the official LibreTranslate image:

``` bash
docker pull libretranslate/libretranslate:latest
```

Create a persistent Docker volume so downloaded translation models
survive container replacement:

``` bash
docker volume create libretranslate_models
```

Start the service:

``` bash
docker run -d \
  --name libretranslate \
  --restart unless-stopped \
  -p 127.0.0.1:5000:5000 \
  -v libretranslate_models:/home/libretranslate/.local \
  libretranslate/libretranslate:latest \
  --load-only en,es,he
```

The port is bound to `127.0.0.1`, so it is not exposed directly to the
local network. The Python script running on this same Ubuntu server can
still reach it.

Check container status and logs:

``` bash
docker ps
docker logs -f libretranslate
```

The first startup may take time because language models may need to be
downloaded. Keep the terminal open while checking the logs; press
`Ctrl+C` to stop following logs (the container keeps running).

Check the health endpoint:

``` bash
curl http://127.0.0.1:5000/health
```

Inspect the languages and translation pairs actually available:

``` bash
curl -s http://127.0.0.1:5000/languages
```

Do not assume all three languages or every pair are ready until this
endpoint and test translations confirm it.

## Useful Docker commands

``` bash
# Show running containers
docker ps

# Follow translation-service logs
docker logs -f libretranslate

# Stop the service
docker stop libretranslate

# Start it again
docker start libretranslate

# Restart it
docker restart libretranslate

# Remove the container (the named model volume remains)
docker rm -f libretranslate
```

To remove downloaded models too (destructive; normally do not run this):

``` bash
docker volume rm libretranslate_models
```

------------------------------------------------------------------------

# 4. Create sample input

Create `samples.txt`:

``` bash
cat > samples.txt <<'EOF'
Welcome everyone. We are grateful to gather together today.
Bienvenidos a todos. Estamos agradecidos de reunirnos hoy.
שלום לכולם. אנחנו שמחים להתכנס כאן היום.
EOF
```

Each non-empty line is one independent sample. Use short, complete
sentences for the first tests.

------------------------------------------------------------------------

# 5. Create a Python virtual environment

``` bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install requests
```

Whenever you reconnect to SSH and want to run the script, activate the
environment again:

``` bash
cd ~/church-translation
source .venv/bin/activate
```

------------------------------------------------------------------------

# 6. Create the processing script

Create `translate_samples.py`:

``` bash
cat > translate_samples.py <<'PY'
from pathlib import Path
import requests

API = "http://127.0.0.1:5000"
TARGET_LANGUAGES = ("en", "es", "he")
SAMPLES_FILE = Path("samples.txt")
TIMEOUT_SECONDS = 120


def detect_language(text: str) -> str:
    response = requests.post(
        f"{API}/detect",
        data={"q": text},
        timeout=TIMEOUT_SECONDS,
    )
    response.raise_for_status()
    detections = response.json()

    if not detections:
        raise RuntimeError("The language detector returned no result.")

    # LibreTranslate returns detections ordered by confidence.
    best = max(detections, key=lambda item: item.get("confidence", 0))
    language = best.get("language")

    if not language:
        raise RuntimeError(f"Unexpected detection response: {detections}")

    return language


def translate(text: str, source: str, target: str) -> str:
    response = requests.post(
        f"{API}/translate",
        data={
            "q": text,
            "source": source,
            "target": target,
            "format": "text",
        },
        timeout=TIMEOUT_SECONDS,
    )
    response.raise_for_status()
    result = response.json()
    return result["translatedText"]


def main() -> None:
    if not SAMPLES_FILE.exists():
        raise FileNotFoundError(
            f"{SAMPLES_FILE} not found. Create it with one sample per line."
        )

    samples = [
        line.strip()
        for line in SAMPLES_FILE.read_text(encoding="utf-8").splitlines()
        if line.strip()
    ]

    if not samples:
        print("No sample text found in samples.txt")
        return

    for number, text in enumerate(samples, start=1):
        print(f"\n{'=' * 65}")
        print(f"Sample {number}")
        print(f"Original: {text}")

        try:
            source = detect_language(text)
            print(f"Detected language: {source}")

            if source not in TARGET_LANGUAGES:
                print(
                    "Skipped: detected language is outside the configured "
                    f"set {TARGET_LANGUAGES}."
                )
                continue

            for target in TARGET_LANGUAGES:
                if target == source:
                    continue

                try:
                    translated = translate(text, source, target)
                    print(f"\n{target}: {translated}")
                except requests.RequestException as error:
                    print(
                        f"\nTranslation to {target} failed. "
                        f"Check whether the {source}->{target} model pair "
                        f"is installed. Details: {error}"
                    )

        except (requests.RequestException, RuntimeError, KeyError) as error:
            print(f"Processing failed: {error}")


if __name__ == "__main__":
    main()
PY
```

------------------------------------------------------------------------

# 7. Run the first end-to-end text test

Make sure the LibreTranslate container is running:

``` bash
docker ps
```

Then run:

``` bash
python translate_samples.py
```

For each sample, the script should:

1.  Read the sentence.
2.  Detect whether it is English, Spanish, or Hebrew.
3.  Translate it into the other two languages.
4.  Print the results in the terminal.

The first run can be slower while translation models initialize.

------------------------------------------------------------------------

# 8. Test the API independently

Before debugging Python, you can test language detection directly:

``` bash
curl -s -X POST http://127.0.0.1:5000/detect \
  -d "q=Welcome everyone to our church"
```

Test one translation direction:

``` bash
curl -s -X POST http://127.0.0.1:5000/translate \
  -d "q=Welcome everyone to our church" \
  -d "source=en" \
  -d "target=es"
```

Try Hebrew as a target:

``` bash
curl -s -X POST http://127.0.0.1:5000/translate \
  -d "q=Welcome everyone to our church" \
  -d "source=en" \
  -d "target=he"
```

If an API call reports that a language pair is unavailable, inspect
`/languages` and the container logs. Do not treat that as a Python bug.

------------------------------------------------------------------------

# 9. What we are deliberately postponing

Do not install these components yet:

-   Whisper / faster-whisper
-   Microphone capture or ALSA configuration
-   Real-time audio streaming
-   React or another frontend
-   WebSockets
-   Text-to-speech
-   A general-purpose LLM

The goal is to validate the language-processing pipeline independently
of audio hardware.

# 10. Planned milestones

  Milestone   Deliverable                                         Status
  ----------- --------------------------------------------------- --------
  1           Ubuntu + Docker prerequisites                       To do
  2           LibreTranslate container starts                     To do
  3           English, Spanish, Hebrew detection                  To do
  4           All required translation directions verified        To do
  5           Python reads text samples and prints translations   To do
  6           Browser-based live caption screen                   Later
  7           Replace text input with Whisper output              Later
  8           Connect microphone and test in the church           Later

## Performance and quality notes

The MacBook Air 2015 is an older, CPU-limited machine. Start with this
small service and measure its memory use, CPU use, startup time, and
translation latency before adding more models.

For a church service, translation quality matters: test real sermon
vocabulary, names, scripture references, and longer sentences with
fluent English, Spanish, and Hebrew speakers. Do not rely on unreviewed
machine translation for critical religious or pastoral wording.

## Official references

-   LibreTranslate installation and Docker guidance:
    https://docs.libretranslate.com/guides/installation/
-   LibreTranslate API documentation: https://docs.libretranslate.com/
-   Official Docker image:
    https://hub.docker.com/r/libretranslate/libretranslate

------------------------------------------------------------------------

## Next step

Run sections 1--3 first. Once the container is healthy, inspect
`/languages`. We can then adjust the translation engine if any of the
six language directions are missing, before writing more application
code.
