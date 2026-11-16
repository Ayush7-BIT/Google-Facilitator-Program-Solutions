TASK 1.4 ----------------------------------------------------------------

```/text
%%bash

# Task 1.4
# Use cURL to test a prompt with the API, by modifying prompt with the prompt from the lab guide.

MODEL_ID="gemini-2.5-flash"

curl -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  https://${API_ENDPOINT}/v1/projects/${PROJECT_ID}/locations/${LOCATION}/publishers/google/models/${MODEL_ID}:streamGenerateContent \
  -d '{
    "contents": {
      "role": "USER",
      "parts": { "text": "I am a Chef. I need to create Japanese recipes for customers who want low sodium meals. However, I do not want to recommend dishes that contain peanuts as some customers have peanut allergies. Please recommend a recipe for dishes that contain ahi tuna, fresh ginger, and edamame. I would also like a wine pairing for each recommendation, I prefer red wine. Include calories and nutritional facts for each recipe." }
    }
  }'
```


```/text
set +H
cat > chef_lab.sh <<'LABEOF'
#!/bin/bash
# Develop GenAI Apps with Gemini and Streamlit: Challenge Lab (GSP517)
# Tasks 2-5 in Cloud Shell. Task 1 is in Workbench JupyterLab (by hand).
# Run: bash chef_lab.sh   (3 questions, then runs by itself ~12 min)
# Runs in background: if Cloud Shell disconnects, run it again - it
# skips finished tasks.  Watch: tail -f ~/chef_lab.log
LOG=~/chef_lab.log; ENVF=~/.chef_lab.env
ask() { local r; read -e -r -p "$2 [$3]: " r; printf -v "$1" '%s' "${r:-$3}"; }

if [ "$1" != "--bg" ]; then
  P=$(gcloud config get-value project 2>/dev/null)
  [ -z "$P" ] && { echo "run: gcloud config set project <Project ID>"; exit 1; }
  if [ "$(grep -s '^P=' $ENVF | cut -d= -f2)" = "$P" ]; then
    source $ENVF; echo "=== Resuming (REGION=$REGION MODEL=$MODEL) ==="
  else
    DR=$(gcloud workbench instances list --format="value(location)" \
      2>/dev/null | head -1 | sed 's/-[a-z]$//')
    echo "=== YELLOW values from the lab page ==="
    ask REGION "REGION" "${DR:-us-central1}"
    ask MODEL "Gemini Flash model ID (GEMINI_FLASH_MODEL_ID)" "gemini-2.5-flash"
    ask CHEF "Chef Python file path (gs://...chef.py)" "gs://spls/gsp517/chef.py"
    printf 'P=%s\nREGION=%s\nMODEL=%s\nCHEF=%s\n' "$P" "$REGION" "$MODEL" \
      "$CHEF" > $ENVF
  fi
  if [ -f ~/.chef_lab.pid ] && kill -0 "$(cat ~/.chef_lab.pid)" 2>/dev/null
  then echo "Already running - showing the log"
  else nohup bash "$0" --bg > $LOG 2>&1 & echo $! > ~/.chef_lab.pid
       echo "Started in background. Ctrl+C only stops watching."; fi
  sleep 1; tail -n +1 -f $LOG --pid="$(cat ~/.chef_lab.pid)"; exit 0
fi

# ===================== background part =====================
source $ENVF; export PROJECT=$P REGION
gcloud config set project "$P" -q 2>/dev/null
D=~/.chef_done/$P; mkdir -p $D
step() { if [ -f $D/$1 ]; then echo; echo "=== $2: already done ==="; return 1
  fi; echo; echo "=== $2 ==="; }
done_() { touch $D/$1; }
die() { echo; echo "STOP: $1 - send Claude a screenshot"; exit 1; }
AR_REPO=chef-repo; SERVICE_NAME=chef-streamlit-app
IMG=$REGION-docker.pkg.dev/$P/$AR_REPO/$SERVICE_NAME
gcloud services enable aiplatform.googleapis.com run.googleapis.com \
  cloudbuild.googleapis.com artifactregistry.googleapis.com -q

cd ~
[ -d generative-ai ] || git clone -q \
  https://github.com/GoogleCloudPlatform/generative-ai.git \
  || die "git clone failed"
APP=~/generative-ai/gemini/sample-apps/gemini-streamlit-cloudrun
cd $APP || die "app folder missing"

if step t2 "Task 2: chef.py (wine radio + prompt) and upload"; then
grep -q '^google-cloud-logging' requirements.txt || \
  echo "google-cloud-logging" >> requirements.txt
[ -f chef.py.orig ] || { gcloud storage cp "$CHEF" chef.py.orig \
  || die "cannot download $CHEF - check the Chef Python file path"; }
cp chef.py.orig chef.py
python3 - "$P" "$MODEL" <<'PY' || exit 1
import re, sys
P, M = sys.argv[1], sys.argv[2]
t = open("chef.py").read()
for ph, v in (("GCP_PROJECT_ID", P), ("GEMINI_FLASH_MODEL_ID", M)):
    t = t.replace(f'"{ph}"', f'"{v}"').replace(f"'{ph}'", f'"{v}"')
# 1) wine radio button, right after the ingredient_3 widget
if not re.search(r"^\s*wine\s*=\s*st\.radio", t, re.M):
    m = re.search(r"^ingredient_3\s*=.*?\(", t, re.M)
    if not m:
        sys.exit("STOP: ingredient_3 widget not found - send Claude: cat -n chef.py")
    i, depth = m.end() - 1, 0
    while True:
        c = t[i]; depth += (c == "(") - (c == ")"); i += 1
        if depth == 0: break
    i = t.index("\n", i) + 1
    radio = ('\nwine = st.radio(\n    "What wine do you prefer?\\n\\n",\n'
             '    ["Red", "White", "None"],\n    key="wine",\n'
             '    horizontal=True,\n)\n')
    t = t[:i] + radio + t[i:]
# 2) the chef prompt
prompt = '''prompt = f"""I am a Chef.  I need to create {cuisine} \\n
recipes for customers who want {dietary_preference} meals. \\n
However, don't include recipes that use ingredients with the {allergy} allergy. \\n
I have {ingredient_1}, \\n
{ingredient_2}, \\n
and {ingredient_3} \\n
in my kitchen and other ingredients. \\n
The customer's wine preference is {wine} \\n
Please provide some for meal recommendations.
For each recommendation include preparation instructions,
time to prepare
and the recipe title at the beginning of the response.
Then include the wine paring for each recommendation.
At the end of the recommendation provide the calories associated with the meal
and the nutritional facts.
"""'''
pat = re.compile(r'^prompt\s*=\s*f?(""".*?"""|\'\'\'.*?\'\'\'|".*?"|\'.*?\')',
                 re.M | re.S)
if pat.search(t):
    t = pat.sub(lambda _: prompt, t, count=1)
else:
    m = re.search(r"^(config|generate_t2t|max_output_tokens)\b", t, re.M)
    if not m:
        sys.exit("STOP: no place for the prompt - send Claude: cat -n chef.py")
    t = t[:m.start()] + prompt + "\n\n" + t[m.start():]
open("chef.py", "w").write(t)
print("   chef.py edited")
PY
python3 -m py_compile chef.py || die "chef.py has a syntax error (cat -n chef.py)"
echo "   --- changes made to chef.py ---"
diff -u chef.py.orig chef.py | sed 's/^/   /' | head -80
gcloud storage buckets describe gs://$P-generative-ai >/dev/null 2>&1 || \
  gcloud storage buckets create gs://$P-generative-ai --location=$REGION -q
gcloud storage cp chef.py gs://$P-generative-ai/ || die "upload failed"
echo ">>> DONE - click 'Check my progress' (Write Streamlit and prompt code)"
done_ t2; fi

if step t3 "Task 3: run the app in Cloud Shell"; then
[ -d gemini-streamlit ] || python3 -m venv gemini-streamlit
source gemini-streamlit/bin/activate
pip install -q -r requirements.txt || die "pip install failed"
pkill -f "streamlit run" 2>/dev/null; sleep 1
nohup streamlit run chef.py --browser.serverAddress=localhost \
  --server.enableCORS=false --server.enableXsrfProtection=false \
  --server.port 8080 > ~/streamlit.log 2>&1 &
for i in $(seq 1 30); do
  curl -s -o /dev/null http://localhost:8080 && break; sleep 2; done
curl -s -o /dev/null -w "   app on port 8080: HTTP %{http_code}\n" \
  http://localhost:8080
echo "   generating one recipe through the app (about 1 minute)..."
python3 - <<'PY'
from streamlit.testing.v1 import AppTest
at = AppTest.from_file("chef.py", default_timeout=240)
at.run()
try:
    for sb in at.selectbox:
        if sb.value is None and sb.options: sb.select(sb.options[0])
    at.run()
    btn = [b for b in at.button if "generate" in (b.key or "").lower()
           or "generate" in str(b.label).lower()] or list(at.button)
    btn[0].click().run()
    txt = " ".join(str(m.value) for m in at.markdown)
    print("   recipe text received:", len(txt), "chars")
    print("  ", txt[:300].replace("\n", " "))
    if at.exception: print("   app exception:", at.exception[0].message[:300])
except Exception as e:
    print("   (test click failed:", str(e)[:200], ")")
PY
deactivate
echo ">>> DONE - click 'Check my progress' (Test the application in Cloud Shell)"
echo "    Optional: Web Preview > port 8080 shows the app; click Generate."
done_ t3; fi

if step t4 "Task 4: Dockerfile + Artifact Registry image (~8 min)"; then
sed -i 's/app\.py/chef.py/g' Dockerfile
grep -q "chef.py" Dockerfile || die "Dockerfile has no app.py to replace (cat Dockerfile)"
grep -n "chef.py" Dockerfile | sed 's/^/   Dockerfile: /'
gcloud artifacts repositories describe $AR_REPO --location=$REGION \
  >/dev/null 2>&1 || gcloud artifacts repositories create $AR_REPO \
  --location=$REGION --repository-format=Docker -q || die "repo create failed"
for i in 1 2 3; do
  gcloud builds submit --tag $IMG --region=$REGION . && break
  [ $i = 3 ] && die "Cloud Build failed 3 times"
  echo "   build failed, retry $i in 30s"; sleep 30
done
echo ">>> DONE - click 'Check my progress' (Modify Dockerfile and push image)"
done_ t4; fi

if step t5 "Task 5: deploy to Cloud Run"; then
for i in 1 2 3; do
  gcloud run deploy $SERVICE_NAME --port=8080 --image=$IMG \
    --allow-unauthenticated --region=$REGION --platform=managed \
    --project=$P --set-env-vars=PROJECT=$P,REGION=$REGION -q && break
  [ $i = 3 ] && die "Cloud Run deploy failed 3 times"
  echo "   deploy failed, retry $i in 30s"; sleep 30
done
URL=$(gcloud run services describe $SERVICE_NAME --region=$REGION \
  --format="value(status.url)")
curl -s -o /dev/null -w "   service answers: HTTP %{http_code}\n" "$URL"
echo "   App URL: $URL"
echo ">>> DONE - open the URL, click 'Generate my recipes', then"
echo "    click 'Check my progress' (Deploy to Cloud Run and test)"
done_ t5; fi
echo ">>> All Cloud Shell tasks done."
LABEOF
echo "SCRIPT SAVED OK - now run: bash chef_lab.sh"
```

bash chef_lab.sh
