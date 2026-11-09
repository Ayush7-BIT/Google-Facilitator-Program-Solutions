```/text
cat > promptdesign_tasks12.sh <<'LABEOF'
#!/bin/bash
# Prompt Design in Agent Platform: Challenge Lab (GSP519) - universal solution
#
# HOW TO USE (everything runs from Cloud Shell, about 10-15 minutes):
#   1. Start the lab, open Cloud Shell, run:  gcloud config set project <Project ID from the lab panel>
#   2. Run:  bash promptdesign_lab.sh
#   3. Answer the 4 questions (each shows where the yellow value is in the lab text; ENTER keeps the default)
#   4. When it pauses: click 'Check my progress' for Task 1 and Task 2, then press ENTER
#   5. It then prints the ready-to-paste notebook code for Task 3 and Task 4 (done by hand in JupyterLab)
#
#   Task 1 + 2: saves both Agent Studio prompts with the lab's model, region and image (automatic)
#   Task 3 + 4: prints the exact code to paste into each notebook (manual, about 5 minutes)
#   Safe to re-run: saved prompts are kept, nothing is duplicated.

SCRIPT_PATH=$(readlink -f "$0")
PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
if [ -z "$PROJECT_ID" ] || [ "$PROJECT_ID" = "(unset)" ]; then
  echo "ERROR: no project set. Run: gcloud config set project <Project ID from the lab panel>"; exit 1
fi
echo "Project: $PROJECT_ID"
STATE_DIR="$HOME/.gsp519"; mkdir -p "$STATE_DIR"
VARS_FILE="$STATE_DIR/vars2"; [ -f "$VARS_FILE" ] && . "$VARS_FILE"
fail() { echo; echo "FAIL: $1"; echo "Fix the error above, then re-run: bash $SCRIPT_PATH"; exit 1; }
ask() { local a; read -r -p "$1 [$2]: " a; a=$(echo "${a:-$2}" | xargs); printf -v "$3" '%s' "$a"; }

# ---------- detect defaults ----------
meta() { gcloud compute project-info describe --quiet --format="value(commonInstanceMetadata.items[$1])" 2>/dev/null </dev/null; }
DET_REGION=$(gcloud config get-value compute/region 2>/dev/null)
[ -z "$DET_REGION" ] || [ "$DET_REGION" = "(unset)" ] && DET_REGION=$(meta google-compute-default-region)
[ -z "$DET_REGION" ] && DET_REGION="us-central1"
echo "Looking for the product image..."
DET_IMAGE=""
for b in $(gcloud storage buckets list --format="value(name)" 2>/dev/null </dev/null); do
  DET_IMAGE=$(gcloud storage ls -r "gs://$b/**" 2>/dev/null </dev/null | grep -iE '\.(jpe?g|png|webp)$' | head -n 1)
  [ -n "$DET_IMAGE" ] && break
done

echo
echo "=== Copy each value from the lab text (yellow), or press ENTER to accept the default ==="
ask "Model      -> Task 1, step 1 (yellow model name, e.g. gemini-3.6-flash)          " "${MODEL:-gemini-3.6-flash}" MODEL
ask "Region     -> Task 1, step 5 ('Save ... select the <yellow> region')              " "${REGION:-$DET_REGION}" REGION
ask "Image path -> Task 1 ('The image for this task is located at: ...')               " "${IMAGE_URI:-$DET_IMAGE}" IMAGE_URI
case "$IMAGE_URI" in gs://*) ;; "") fail "image path is empty - copy it from Task 1" ;; *) IMAGE_URI="gs://$IMAGE_URI" ;; esac
case "${IMAGE_URI,,}" in *.png) MIME="image/png" ;; *.webp) MIME="image/webp" ;; *) MIME="image/jpeg" ;; esac
{ for v in MODEL REGION IMAGE_URI; do printf '%s=%q\n' "$v" "${!v}"; done; } > "$VARS_FILE"
gcloud config set compute/region "$REGION" >/dev/null 2>&1
gcloud services enable aiplatform.googleapis.com notebooks.googleapis.com --quiet </dev/null >/dev/null 2>&1

# ---------- where does this model answer? (newer Gemini models are often 'global' only) ----------
echo; echo "=== Checking where $MODEL answers ==="
TOKEN=$(gcloud auth print-access-token </dev/null)
try_model() {   # try_model LOCATION
  local host="${1}-aiplatform.googleapis.com"; [ "$1" = global ] && host="aiplatform.googleapis.com"
  curl -s -m 90 -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    "https://${host}/v1/projects/${PROJECT_ID}/locations/${1}/publishers/google/models/${MODEL}:generateContent" \
    -d '{"contents":[{"role":"user","parts":[{"text":"Say OK"}]}]}' | grep -q '"candidates"'
}
GEN_LOCATION=""
for loc in "$REGION" global; do
  if try_model "$loc"; then GEN_LOCATION="$loc"; break; fi
  echo "   $MODEL does not answer in $loc"
done
if [ -z "$GEN_LOCATION" ]; then
  echo "   WARNING: $MODEL did not answer anywhere - check the model name in Task 1. Using 'global' for the notebooks."
  GEN_LOCATION="global"
else
  echo "   OK: $MODEL answers in $GEN_LOCATION"
fi

# ---------- install the Vertex AI SDK (once) ----------
VENV="$HOME/.gsp519-venv"
if ! "$VENV/bin/python" -c "import vertexai; vertexai.Client" >/dev/null 2>&1; then
  echo; echo "=== Installing the Vertex AI SDK (about 1-2 minutes) ==="
  python3 -m venv "$VENV" || fail "could not create a Python environment"
  "$VENV/bin/pip" install -q --upgrade pip >/dev/null 2>&1
  "$VENV/bin/pip" install -q "google-cloud-aiplatform>=1.110,<3" google-genai || fail "pip install failed"
fi

# ---------- TASK 1 + TASK 2: save the two prompts in Agent Studio ----------
# Saved prompts are Vertex AI datasets. For each name we keep exactly ONE copy saved with the lab's model:
# copies with another model (or extra duplicates) are deleted, a missing one is (re)created.
API="https://${REGION}-aiplatform.googleapis.com/v1"
list_prompts() {   # list_prompts NAME  ->  "<resource name> <model>" per saved copy
  curl -s -H "Authorization: Bearer $(gcloud auth print-access-token </dev/null)" \
    "$API/projects/${PROJECT_ID}/locations/${REGION}/datasets?pageSize=100" |
  python3 -c 'import sys,json
d=json.load(sys.stdin)
if "error" in d: sys.stderr.write("   (list error: %s)\n" % d["error"].get("message","")[:150]); sys.exit(0)
for x in d.get("datasets",[]):
    if x.get("displayName")==sys.argv[1]: print(x["name"], x.get("modelReference",""))' "$1"
}
NEED=""
for n in "Cymbal Product Analysis" "Cymbal Tagline Generator Template"; do
  keep=""
  while read -r id model; do
    [ -z "$id" ] && continue
    if [ -z "$keep" ] && [ "${model##*/}" = "$MODEL" ]; then keep="$id"; continue; fi
    curl -s -X DELETE -H "Authorization: Bearer $(gcloud auth print-access-token </dev/null)" "$API/$id" >/dev/null
    echo "   removed an old copy of '$n' (model: ${model:-none})"
  done < <(list_prompts "$n")
  if [ -n "$keep" ]; then echo "   '$n' is already saved with $MODEL - keeping it"; else NEED="$NEED|$n"; fi
done

if [ -n "$NEED" ]; then
  echo; echo "=== TASK 1 + TASK 2: saving the prompts in Agent Studio with $MODEL in $REGION ==="
  PYTHONWARNINGS=ignore NEED="$NEED" PROJECT_ID="$PROJECT_ID" REGION="$REGION" MODEL="$MODEL" IMAGE_URI="$IMAGE_URI" MIME="$MIME" \
  "$VENV/bin/python" - <<'PYEOF' || echo "   (the save call reported an error above - checking what was saved anyway)"
import os, vertexai
from google.genai import types
P, R, M = os.environ["PROJECT_ID"], os.environ["REGION"], os.environ["MODEL"]
NEED = os.environ["NEED"]
client = vertexai.Client(project=P, location=R)
T = lambda s: types.Part.from_text(text=s)

def save(name, prompt):
    if name not in NEED:
        return
    try:
        client.prompts.create(prompt=prompt, config={"prompt_display_name": name, "timeout": 900})
        print("   saved: " + name)
    except TimeoutError:
        print("   still saving in the background: " + name + " (checked below)")

# Task 1 - image analysis prompt: the product image + a request for the 3 kinds of text
save("Cymbal Product Analysis", {"prompt_data": {
    "model": M,
    "contents": [types.Content(role="user", parts=[
        types.Part.from_uri(file_uri=os.environ["IMAGE_URI"], mime_type=os.environ["MIME"]),
        T("Describe this image with a focus on colors, textures, and the feeling it evokes. Generate: "
          "1) short, descriptive text inspired by the image, 2) catchy phrases suitable for advertisements, "
          "3) a poetic description for a nature-focused campaign.")])],
    "generation_config": types.GenerationConfig(temperature=1.0, top_p=0.95, max_output_tokens=8192),
}})

# Task 2 - tagline generator: system instructions, 2 examples, 3 parameters
SYS = ("Cymbal Direct is partnering with an outdoor gear retailer. They're launching a new line of products "
       "designed to encourage young people to explore the outdoors. Help them create catchy taglines for this "
       "product line.")
save("Cymbal Tagline Generator Template", {"prompt_data": {
    "model": M,
    "system_instruction": types.Content(parts=[T(SYS)]),
    "contents": [types.Content(role="user", parts=[T(
        "input: Write a tagline for a durable backpack designed for hikers that makes them feel prepared. "
        "Consider styles like minimalist.\n"
        "output: Built for the Journey: Your Adventure Essentials.\n\n"
        "input: Write a tagline for a lightweight tent designed for families that makes them feel connected. "
        "Consider styles like playful.\n"
        "output: Pitch Together, Dream Together.\n\n"
        "input: Write a tagline for a {product_attributes} product designed for {target_audience} "
        "that makes them feel {emotional_resonance}.\n"
        "output:")])],
    "variables": [{"product_attributes": T("lightweight, durable"),
                   "target_audience": T("young adventurers"),
                   "emotional_resonance": T("empowered")}],
    "generation_config": types.GenerationConfig(temperature=1.0, top_p=0.95, max_output_tokens=8192),
}})
PYEOF
fi

echo; echo "=== Checking that both prompts are saved with $MODEL (waits up to 10 minutes) ==="
for n in "Cymbal Product Analysis" "Cymbal Tagline Generator Template"; do
  ok=0
  for i in $(seq 1 40); do
    if list_prompts "$n" | awk '{print $2}' | grep -q "${MODEL}\$"; then ok=1; break; fi
    echo "   '$n' not visible yet ($i/40) - waiting 15s"; sleep 15
  done
  [ "$ok" = 1 ] && echo "   OK: '$n' is saved" || fail "'$n' was not saved - re-run the script, it only saves what is missing"
done
echo
read -r -p ">>> Lab page: click 'Check my progress' for Task 1 and Task 2, then press ENTER here " _

# ---------- TASK 3 + TASK 4: print the ready-to-paste notebook code ----------
OUT="$HOME/gsp519_notebook_code"; mkdir -p "$OUT"
PROJECT_ID="$PROJECT_ID" GEN_LOCATION="$GEN_LOCATION" MODEL="$MODEL" IMAGE_URI="$IMAGE_URI" MIME="$MIME" OUT="$OUT" \
python3 - <<'PYEOF'
import os
PROJECT_ID = os.environ["PROJECT_ID"]; LOCATION = os.environ["GEN_LOCATION"]; MODEL = os.environ["MODEL"]
IMAGE_URI = os.environ["IMAGE_URI"]; MIME = os.environ["MIME"]
IMAGE_CODE = '''from google import genai
from google.genai import types

def generate():
  client = genai.Client(vertexai=True, project=PROJECT_ID, location=LOCATION)

  msg1_image1 = types.Part.from_uri(
      file_uri="%(I)s",
      mime_type="%(MI)s",
  )
  msg1_text1 = types.Part.from_text(text="""In less than 10 words, describe this image with the most creative, unusual and unexpected description you can think of, focusing on its colors, textures and the feeling it evokes.""")

  model = "%(M)s"
  contents = [
    types.Content(
      role="user",
      parts=[
        msg1_image1,
        msg1_text1
      ]
    ),
  ]

  generate_content_config = types.GenerateContentConfig(
    temperature = 2,
    top_p = 0.95,
    max_output_tokens = 8192,
  )

  for chunk in client.models.generate_content_stream(
    model = model,
    contents = contents,
    config = generate_content_config,
    ):
    if chunk.text:
      print(chunk.text, end="")

generate()
''' % dict(P=PROJECT_ID, L=LOCATION, I=IMAGE_URI, MI=MIME, M=MODEL)

TAGLINE_CODE = '''from google import genai
from google.genai import types

def generate():
  client = genai.Client(vertexai=True, project=PROJECT_ID, location=LOCATION)

  si_text1 = """Cymbal Direct is partnering with an outdoor gear retailer. They're launching a new line of products designed to encourage young people to explore the outdoors. Help them create catchy taglines for this product line."""

  model = "%(M)s"
  contents = [
    types.Content(role="user", parts=[types.Part.from_text(text="""input: Write a tagline for a durable backpack designed for hikers that makes them feel prepared. Consider styles like minimalist.""")]),
    types.Content(role="model", parts=[types.Part.from_text(text="""output: Built for the Journey: Your Adventure Essentials.""")]),
    types.Content(role="user", parts=[types.Part.from_text(text="""input: Write a tagline for a lightweight tent designed for families that makes them feel connected. Consider styles like playful.""")]),
    types.Content(role="model", parts=[types.Part.from_text(text="""output: Pitch Together, Dream Together.""")]),
    types.Content(
      role="user",
      parts=[
        types.Part.from_text(text="""input: Write a tagline for a lightweight, durable product designed for young adventurers that makes them feel empowered.
The tagline must include the keyword "nature".
output:""")
      ]
    ),
  ]

  generate_content_config = types.GenerateContentConfig(
    temperature = 1,
    top_p = 0.95,
    max_output_tokens = 8192,
    system_instruction=[types.Part.from_text(text=si_text1)],
  )

  for chunk in client.models.generate_content_stream(
    model = model,
    contents = contents,
    config = generate_content_config,
    ):
    if chunk.text:
      print(chunk.text, end="")

generate()
''' % dict(P=PROJECT_ID, L=LOCATION, M=MODEL)


open(os.path.join(os.environ["OUT"], "task3_image_analysis.py"), "w").write(IMAGE_CODE)
open(os.path.join(os.environ["OUT"], "task4_tagline_generator.py"), "w").write(TAGLINE_CODE)
PYEOF

line() { printf '%.0s=' $(seq 1 90); echo; }
echo
line
echo " TASK 3 - paste this into image-analysis.ipynb (the cell that says 'PASTE YOUR CODE BELOW')"
line
cat "$OUT/task3_image_analysis.py"
line
echo " TASK 4 - paste this into tagline-generator.ipynb (the cell that says 'PASTE YOUR CODE BELOW')"
line
cat "$OUT/task4_tagline_generator.py"
line
cat <<'STEPS'
 HOW TO DO TASK 3 + 4 (about 5 minutes):
  1. Console > Vertex AI > Workbench > your instance > OPEN JUPYTERLAB
  2. Open image-analysis.ipynb. Run the %pip install cell (Shift+Enter), then
     Kernel > Restart Kernel (IMPORTANT - otherwise you get a GCE_METADATA_TIMEOUT error).
  3. Run the PROJECT_ID/LOCATION cell and the client cell (Shift+Enter each).
  4. In the cell with 'PASTE YOUR CODE BELOW', paste the TASK 3 code above (below that line),
     then press Shift+Enter and wait until text is printed under the cell.
  5. Press Ctrl+S to save the notebook. Click 'Check my progress' for Task 3.
  6. Open tagline-generator.ipynb and do steps 2-5 with the TASK 4 code. Ctrl+S, then 'Check my progress'.
  (The code is also saved in Cloud Shell: ~/gsp519_notebook_code/)
STEPS

LABEOF
```


bash promptdesign_tasks12.sh
