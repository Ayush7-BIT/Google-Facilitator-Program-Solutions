Check that the prompt shows the project in brackets, like (qwiklabs-gcp-...). If it doesn’t, run gcloud config set project <Project ID from the lab panel>


```text
cat > speech_lab.sh <<'LABEOF'
#!/bin/bash
# Universal solver (Cloud Shell version): Implement Speech and Language Solutions with Pre-trained APIs
# Run in Cloud Shell with:  bash speech_lab.sh

ask() { local r; read -e -r -p "$2 [$3]: " r; printf -v "$1" '%s' "${r:-$3}"; }
die() { echo; echo "STOP: $1"; exit 1; }

# Fixed by the lab text (not yellow)
STT_LANG="fr-FR"
TARGET_LANG="en"
TRANSLATE_REQ="translation_request.json"
DETECT_REQ="detect_request.json"
TTS_AUDIO="synthesize-text-audio.mp3"

retry() {            # retry <output-file> <curl command...>
  local out="$1"; shift
  for i in 1 2 3 4 5 6; do
    "$@" > "$out"
    grep -q '"error": *{' "$out" || return 0
    echo "   attempt $i failed, waiting 10s (a new key can take a minute to activate)..."
    sleep 10
  done
  return 1
}

PROJECT=$(gcloud config get-value project 2>/dev/null)
[ -z "$PROJECT" ] && die "no project set. Run: gcloud config set project <Project ID from the lab panel>"
echo "Project: $PROJECT"

echo
echo "=== Task 1: API key ==="
gcloud services enable apikeys.googleapis.com texttospeech.googleapis.com speech.googleapis.com translate.googleapis.com
get_key() {
  local n
  n=$(gcloud services api-keys list --filter="displayName=lab-key" --format="value(name)" 2>/dev/null | head -1)
  [ -n "$n" ] && gcloud services api-keys get-key-string "$n" --format="value(keyString)" 2>/dev/null
}
API_KEY=$(get_key)
if [ ${#API_KEY} -ne 39 ]; then
  gcloud services api-keys create --display-name="lab-key" \
    --api-target=service=texttospeech.googleapis.com \
    --api-target=service=speech.googleapis.com \
    --api-target=service=translate.googleapis.com
  sleep 10
  API_KEY=$(get_key)
fi
if [ ${#API_KEY} -ne 39 ]; then   # last resort: any key already in the project
  n=$(gcloud services api-keys list --format="value(name)" 2>/dev/null | head -1)
  [ -n "$n" ] && API_KEY=$(gcloud services api-keys get-key-string "$n" --format="value(keyString)")
fi
[ ${#API_KEY} -ne 39 ] && die "no API key. Create one in Console > APIs & Services > Credentials > Create credentials > API key (restrict it to Text-to-Speech, Speech-to-Text and Translation), then run: bash speech_lab.sh"
export API_KEY
echo "API key ready (length ${#API_KEY}). Click 'Check my progress' for Task 1 on the lab page."

echo
echo "=== Enter the YELLOW values from the lab (press Enter to keep the default) ==="
DEFAULT_BUCKET=$(gcloud storage buckets list --format="value(name)" 2>/dev/null | grep -m1 -- '-speech$')
ask BUCKET         "Bucket name (named in the lab intro)"          "${DEFAULT_BUCKET:-${PROJECT}-speech}"
ask TTS_RESPONSE   "Task 2 - file for the Text-to-Speech result"   "synthesize-text.txt"
ask STT_REQUEST    "Task 3 - request file name (.json)"            "speech_request_fr.json"
ask STT_RESPONSE   "Task 3 - file for the result"                  "speech_response.json"
ask TRANSLATE_TEXT "Task 4 - sentence to translate"                "これは日本語です。"
ask TRANSLATE_OUT  "Task 4 - file for the result"                  "translation_response.txt"
ask DETECT_TEXT    "Task 5 - sentence to detect"                   "Este%é%japonês."
ask DETECT_OUT     "Task 5 - file for the result"                  "detected_response.txt"

gcloud storage ls "gs://$BUCKET" >/dev/null 2>&1 || die "bucket gs://$BUCKET not found. Copy the exact bucket name from the lab intro and run again."
cd ~

echo
echo "=== Task 2: Text-to-Speech ==="
cat > synthesize-text.json <<'JSON1'
{
  "input": {
    "text": "Cloud Text-to-Speech API allows developers to include natural-sounding, synthetic human speech as playable audio in their applications. The Text-to-Speech API converts text or Speech Synthesis Markup Language (SSML) input into audio data like MP3 or LINEAR16 (the encoding used in WAV files)."
  },
  "voice": {
    "languageCode": "en-gb",
    "name": "en-GB-Standard-A",
    "ssmlGender": "FEMALE"
  },
  "audioConfig": {
    "audioEncoding": "MP3"
  }
}
JSON1
retry "$TTS_RESPONSE" curl -s -X POST -H "X-Goog-Api-Key: ${API_KEY}" \
  -H "Content-Type: application/json; charset=utf-8" --data @synthesize-text.json \
  "https://texttospeech.googleapis.com/v1/text:synthesize" || die "Text-to-Speech call failed: $(head -c 400 "$TTS_RESPONSE")"
cat > tts_decode.py <<'PY1'
import argparse
from base64 import decodebytes
import json

def decode_tts_output(input_file, output_file):
    with open(input_file) as input:
        response = json.load(input)
        audio_data = response['audioContent']
        with open(output_file, "wb") as new_file:
            new_file.write(decodebytes(audio_data.encode('utf-8')))

if __name__ == '__main__':
    parser = argparse.ArgumentParser(description="Decode output from Cloud Text-to-Speech")
    parser.add_argument('--input', help='The response from the Text-to-Speech API.', required=True)
    parser.add_argument('--output', help='The name of the audio file to create', required=True)
    args = parser.parse_args()
    decode_tts_output(args.input, args.output)
PY1
python3 tts_decode.py --input "$TTS_RESPONSE" --output "$TTS_AUDIO"
ls -l "$TTS_AUDIO"
gcloud storage cp "$TTS_RESPONSE" "gs://$BUCKET/"
gcloud storage cp "$TTS_AUDIO" "gs://$BUCKET/"

echo
echo "=== Task 3: Speech-to-Text ==="
cat > "$STT_REQUEST" <<JSON2
{
  "config": {
    "encoding": "FLAC",
    "languageCode": "${STT_LANG}"
  },
  "audio": {
    "uri": "gs://cloud-samples-data/speech/corbeau_renard.flac"
  }
}
JSON2
retry "$STT_RESPONSE" curl -s -X POST -H "Content-Type: application/json" --data-binary @"$STT_REQUEST" \
  "https://speech.googleapis.com/v1/speech:recognize?key=${API_KEY}" || die "Speech-to-Text call failed: $(head -c 400 "$STT_RESPONSE")"
cat "$STT_RESPONSE"
gcloud storage cp "$STT_REQUEST" "gs://$BUCKET/"
gcloud storage cp "$STT_RESPONSE" "gs://$BUCKET/"

echo
echo "=== Task 4: Translate ==="
jq -n --arg q "$TRANSLATE_TEXT" --arg t "$TARGET_LANG" '{q: $q, target: $t}' > "$TRANSLATE_REQ"
cat "$TRANSLATE_REQ"
retry "$TRANSLATE_OUT" curl -s -X POST -H "Content-Type: application/json" --data-binary @"$TRANSLATE_REQ" \
  "https://translation.googleapis.com/language/translate/v2?key=${API_KEY}" || die "Translation call failed: $(head -c 400 "$TRANSLATE_OUT")"
cat "$TRANSLATE_OUT"
gcloud storage cp "$TRANSLATE_OUT" "gs://$BUCKET/"

echo
echo "=== Task 5: Detect language ==="
jq -n --arg q "$DETECT_TEXT" '{q: $q}' > "$DETECT_REQ"
cat "$DETECT_REQ"
retry "$DETECT_OUT" curl -s -X POST -H "Content-Type: application/json" --data-binary @"$DETECT_REQ" \
  "https://translation.googleapis.com/language/translate/v2/detect?key=${API_KEY}" || die "Detection call failed: $(head -c 400 "$DETECT_OUT")"
cat "$DETECT_OUT"
gcloud storage cp "$DETECT_OUT" "gs://$BUCKET/"

echo
echo "=== Self-check: files in gs://$BUCKET ==="
LISTING=$(gcloud storage ls "gs://$BUCKET/" 2>/dev/null)
for f in "$TTS_RESPONSE" "$TTS_AUDIO" "$STT_REQUEST" "$STT_RESPONSE" "$TRANSLATE_OUT" "$DETECT_OUT"; do
  if echo "$LISTING" | grep -q "/$f$"; then echo "  PASS  $f is in the bucket"; else echo "  FAIL  $f is missing from the bucket"; fi
done
[ "$(stat -c%s "$TTS_AUDIO" 2>/dev/null || echo 0)" -gt 10000 ] && echo "  PASS  MP3 looks valid" || echo "  FAIL  MP3 is too small"
grep -q transcript "$STT_RESPONSE" && echo "  PASS  French transcript received" || echo "  FAIL  no transcript in $STT_RESPONSE"
grep -q translatedText "$TRANSLATE_OUT" && echo "  PASS  translation received" || echo "  FAIL  no translation in $TRANSLATE_OUT"
grep -q '"language"' "$DETECT_OUT" && echo "  PASS  language detected" || echo "  FAIL  no detection in $DETECT_OUT"
echo
echo ">>> Done. On the lab page, click 'Check my progress' for Tasks 2, 3, 4 and 5."
LABEOF
```

bash speech_lab.sh
