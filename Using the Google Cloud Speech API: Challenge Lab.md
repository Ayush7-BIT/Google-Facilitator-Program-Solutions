```text
cat > speech_challenge.sh <<'LABEOF'
#!/bin/bash
# Using the Google Cloud Speech API: Challenge Lab - universal solution

PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
if [ -z "$PROJECT_ID" ] || [ "$PROJECT_ID" = "(unset)" ]; then
  echo "ERROR: no project set. Run: gcloud config set project <Project ID from the lab panel>"
  exit 1
fi
export GOOGLE_CLOUD_PROJECT="$PROJECT_ID"
echo "Project: $PROJECT_ID"

ask() {   # ask "prompt" "default" VARNAME
  local ans
  read -r -p "$1 [$2]: " ans
  printf -v "$3" '%s' "${ans:-$2}"
}

echo
echo "=== Press ENTER to accept each default (the defaults are THIS lab's values) ==="
ask "Task 2 - request file name   " "request.json" REQ_EN
ask "Task 2 - response file name  " "speech_response_en.json" RES_EN
ask "Task 2 - audio file (gs://)  " "gs://spls/arc131/question_en.wav" AUDIO_EN
ask "Task 2 - language code       " "en-US" LANG_EN
ask "Task 3 - request file name   " "request_speech_sp.json" REQ_SP
ask "Task 3 - response file name  " "speech_response_sp.json" RES_SP
ask "Task 3 - audio file (gs://)  " "gs://spls/arc131/multi_es.flac" AUDIO_SP
ask "Task 3 - language code       " "es-ES" LANG_SP
ask "Storage bucket name          " "${PROJECT_ID}-speech" BUCKET_NAME

gcloud services enable speech.googleapis.com apikeys.googleapis.com --quiet 2>/dev/null || true

# ---------- bucket (the lab normally creates it already) ----------
if ! gcloud storage buckets describe "gs://${BUCKET_NAME}" >/dev/null 2>&1; then
  echo "Bucket not found - creating gs://${BUCKET_NAME}"
  gcloud storage buckets create "gs://${BUCKET_NAME}" --location=US --quiet
fi

# ---------- TASK 1: API key ----------
echo
echo "=== TASK 1: API key ==="
read -r -p "Paste an API key you already created, or just press ENTER to let the script create one: " API_KEY
KEY_DISPLAY="speech-challenge-key"
get_key_string() {
  local n
  n=$(gcloud services api-keys list --filter="displayName=$KEY_DISPLAY" --format="value(name)" 2>/dev/null | head -n1)
  if [ -n "$n" ]; then
    gcloud services api-keys get-key-string "$n" --format="value(keyString)" 2>/dev/null
  fi
}
if [ -z "$API_KEY" ]; then
  API_KEY=$(get_key_string)
  if [ -z "$API_KEY" ]; then
    echo "Creating API key (restricted to the Speech-to-Text API)..."
    gcloud services api-keys create --display-name="$KEY_DISPLAY" \
      --api-target=service=speech.googleapis.com --quiet
    for i in $(seq 1 12); do
      API_KEY=$(get_key_string)
      [ -n "$API_KEY" ] && break
      sleep 5
    done
  fi
fi
if [ -z "$API_KEY" ]; then
  echo "FAIL: could not get an API key. Create one in Console > APIs & Services > Credentials, then re-run and paste it."
  exit 1
fi
export API_KEY
echo "API key ready: ${API_KEY:0:8}..."

# ---------- helpers ----------
build_request() {   # file uri lang encoding channels
  local cfg="\"languageCode\": \"$3\""
  [ -n "$4" ] && cfg="\"encoding\": \"$4\", $cfg"
  [ -n "$5" ] && cfg="$cfg, \"audioChannelCount\": $5"
  cat > "$1" <<EOF
{
  "config": {
    $cfg
  },
  "audio": {
    "uri": "$2"
  }
}
EOF
}

transcribe() {      # req res uri lang channels_first(yes/no)
  local req="$1" res="$2" uri="$3" lang="$4" chfirst="$5"
  local ext="${uri##*.}" enc="" idx=0 waits=0 e c
  local variants=()
  ext="${ext,,}"
  case "$ext" in
    wav)  enc="LINEAR16" ;;
    flac) enc="FLAC" ;;
    mp3)  enc="MP3" ;;
    ogg)  enc="OGG_OPUS" ;;
  esac
  if [ "$chfirst" = "yes" ]; then
    variants=("$enc|2" "|2" "$enc|" "|")
  else
    variants=("$enc|" "$enc|2" "|2" "|")
  fi
  while [ $idx -lt ${#variants[@]} ] && [ $waits -lt 20 ]; do
    IFS='|' read -r e c <<< "${variants[$idx]}"
    build_request "$req" "$uri" "$lang" "$e" "$c"
    curl -s -X POST -H "Content-Type: application/json" --data-binary @"$req" \
      "https://speech.googleapis.com/v1/speech:recognize?key=${API_KEY}" > "$res"
    if grep -q '"transcript"' "$res"; then
      echo "   OK - transcript received (encoding='${e}' channels='${c}')"
      return 0
    elif grep -qE 'API_KEY_INVALID|API key not valid|PERMISSION_DENIED|SERVICE_DISABLED|has not been used|API_KEY_SERVICE_BLOCKED|blocked' "$res"; then
      waits=$((waits+1))
      echo "   API key / API not active yet - waiting 15s (try $waits/20)..."
      sleep 15
    else
      echo "   request variant $((idx+1)) rejected or empty - trying the next one"
      idx=$((idx+1))
    fi
  done
  echo "   FAILED. Last response:"
  cat "$res"
  return 1
}

# ---------- TASK 2: English ----------
echo
echo "=== TASK 2: Transcribe speech to English ==="
transcribe "$REQ_EN" "$RES_EN" "$AUDIO_EN" "$LANG_EN" yes
R2=$?
if [ $R2 -eq 0 ]; then
  gcloud storage cp "$REQ_EN" "gs://${BUCKET_NAME}/${REQ_EN}" --quiet
  gcloud storage cp "$RES_EN" "gs://${BUCKET_NAME}/${RES_EN}" --quiet
fi

# ---------- TASK 3: Spanish ----------
echo
echo "=== TASK 3: Transcribe speech to Spanish ==="
transcribe "$REQ_SP" "$RES_SP" "$AUDIO_SP" "$LANG_SP" no
R3=$?
if [ $R3 -eq 0 ]; then
  gcloud storage cp "$REQ_SP" "gs://${BUCKET_NAME}/${REQ_SP}" --quiet
  gcloud storage cp "$RES_SP" "gs://${BUCKET_NAME}/${RES_SP}" --quiet
fi

# ---------- SELF-CHECK ----------
echo
echo "=================== SELF-CHECK ==================="
[ -n "$API_KEY" ] && echo "PASS  API key exists" || echo "FAIL  no API key"
for f in "$REQ_EN" "$RES_EN" "$REQ_SP" "$RES_SP"; do
  if gcloud storage ls "gs://${BUCKET_NAME}/${f}" >/dev/null 2>&1; then
    echo "PASS  $f is in gs://${BUCKET_NAME}/"
  else
    echo "FAIL  $f is NOT in gs://${BUCKET_NAME}/"
  fi
done
for r in "$RES_EN" "$RES_SP"; do
  if grep -q '"transcript"' "$r" 2>/dev/null; then echo "PASS  $r contains a transcript"; else echo "FAIL  $r has no transcript"; fi
done
echo
echo "English transcript:"; grep -m2 '"transcript"' "$RES_EN" 2>/dev/null
echo "Spanish transcript:"; grep -m2 '"transcript"' "$RES_SP" 2>/dev/null
echo
echo "If the lab wants the key in your shell, run this line:"
echo "export API_KEY=\"$API_KEY\""
echo
echo "NOW go to the lab page and click 'Check my progress' for Task 1, Task 2 and Task 3."
LABEOF
```

bash speech_challenge.sh


