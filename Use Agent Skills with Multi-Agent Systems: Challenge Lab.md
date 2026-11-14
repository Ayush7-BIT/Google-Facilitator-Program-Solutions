```/text
set +H
cat > agents_lab.sh <<'LABEOF'
#!/bin/bash
# Use Agent Skills with Multi-Agent Systems: Challenge Lab (GSP541)
# Run: bash agents_lab.sh   (2 questions with defaults, then runs by itself)
ask() { local r; read -e -r -p "$2 [$3]: " r; printf -v "$1" '%s' "${r:-$3}"; }
P=$(gcloud config get-value project 2>/dev/null)
[ -z "$P" ] && { echo "run: gcloud config set project <Project ID>"; exit 1; }
echo "=== YELLOW values from Task 1 step 6:  export REGION=\"...\"  export MODEL=\"...\" ==="
REGION=""
while [ -z "$REGION" ]; do read -e -r -p "REGION exactly as in Task 1 step 6 (e.g. us-west1): " REGION; done
ask MODEL  "MODEL"  "gemini-3.6-flash"
B="gs://${P}-agent-bucket"
D=~/multi-agent-system-skills-starter

# ================= Task 1 setup =================
echo; echo "=== setup: starter code, uv sync, .env ==="
cd ~
[ -d "$D" ] || gcloud storage cp -r "gs://${P}-assets/multi-agent-system-skills-starter" . \
  || { echo "STOP: could not download the starter code"; exit 1; }
cd "$D" || exit 1
# files copied from Cloud Storage lose their execute permission; without it the
# orchestrator container fails with "/app/run.sh: permission denied"
find . -name "*.sh" -exec chmod +x {} +
command -v uv >/dev/null || { pip install -q --user uv; export PATH="$HOME/.local/bin:$PATH"; }
uv sync -q 2>&1 | tail -2
cat > .env <<EOF
export GOOGLE_CLOUD_PROJECT="$P"
export GOOGLE_CLOUD_LOCATION="global"
export GOOGLE_GENAI_USE_VERTEXAI=true
export REGION=$REGION
export MODEL=$MODEL
EOF
source .env
gcloud services enable run.googleapis.com cloudbuild.googleapis.com \
  artifactregistry.googleapis.com aiplatform.googleapis.com >/dev/null 2>&1

# ================= code edits (Tasks 1-5) =================
python3 - <<'PY'
import re, pathlib
def edit(path, fn):
    p = pathlib.Path(path); old = p.read_text(); new = fn(old)
    if new == old: print(f"   {path}: no change (already done or pattern not found)")
    else: p.write_text(new); print(f"   {path}: updated")
def todo_text(text, rule):
    if rule.split(".")[0] in text: return text
    lines = text.split("\n")
    for i, l in enumerate(lines):
        if "TODO" in l:
            lines[i] = rule; return "\n".join(lines)
    return text.rstrip("\n") + "\n\n" + rule + "\n"
def tools(name):
    return lambda t: re.sub(r"tools\s*=\s*\[\s*None\s*\]", f"tools=[{name}]", t)

print("\n=== Task 1: researcher ===")
edit("agents/researcher/skills/SKILL.md", lambda t: todo_text(t,
  "Do not try to invoke any skills or tools directly. Do not output raw function "
  "calls or JSON payloads. Do not call any function named 'skills_search'. Provide "
  "your research findings directly as a plain text summary."))
edit("agents/researcher/agent.py", tools("researcher_toolset"))

print("\n=== Task 2: judge ===")
def judge(t):
    t = re.sub(r"output_schema\s*=\s*None", "output_schema=JudgeFeedback", t)
    t = re.sub(r"(disallow_transfer_to_parent\s*=\s*)(None|False)", r"\1True", t)
    t = re.sub(r"(disallow_transfer_to_peers\s*=\s*)(None|False)", r"\1True", t)
    return t
edit("agents/judge/agent.py", judge)

print("\n=== Task 3: content builder ===")
def fmt(t):
    if re.search(r"""['"]success['"]\s*:""", t): return t
    m = re.search(r"def check_format\([^)]*\)[^:]*:", t)
    if not m: return t
    body = t[m.end():]
    bools = re.findall(r"^\s+(\w+)\s*=\s*.*\bin\b.*$", body, re.M)
    cond = " and ".join(bools[:2]) if len(bools) >= 2 else None
    if not cond:
        print("   check_format.py: could not find the two heading checks - see file below")
        return t
    ret = ('return {"success": bool(%s), "message": "Both required headings are present." '
           'if (%s) else "Missing one or both required headings."}' % (cond, cond))
    new, n = re.subn(r"^(\s+)(return\s+None|pass|return\s*)\s*(#.*)?$",
                     lambda mm: mm.group(1) + ret, body, count=1, flags=re.M)
    if n == 0:
        new = body.rstrip("\n") + "\n    " + ret + "\n"
    return t[:m.end()] + new
edit("agents/content_builder/skills/scripts/check_format.py", fmt)
edit("agents/content_builder/skills/SKILL.md", lambda t: todo_text(t,
  "Before finalizing your output, you MUST run the validation script tool "
  "`check_format` to verify the formatting of the generated course module. If it "
  "returns success false, fix the formatting and run `check_format` again."))
edit("agents/content_builder/agent.py", tools("content_builder_toolset"))

print("\n=== Task 4: orchestrator ===")
def orch(t):
    i = t.find("judge")
    j = t.find("after_agent_callback", i if i >= 0 else 0)
    if j >= 0:
        seg = t[j:j+80]
        seg2 = re.sub(r"after_agent_callback\s*=\s*None",
                      'after_agent_callback=create_save_output_callback("judge_feedback")', seg, count=1)
        t = t[:j] + seg2 + t[j+80:]
    t = re.sub(r"actions\s*=\s*None", "actions=EventActions(escalate=True)", t, count=1)
    k = t.find("research_loop")
    k = t.find("research_loop", k + 1) if t.find("research_loop =") < 0 else t.find("research_loop =")
    if k >= 0:
        t = t[:k] + re.sub(r"\[\s*None\s*\]", "[researcher, judge, escalation_checker]", t[k:], count=1)
    k = t.find("root_agent =")
    if k >= 0:
        t = t[:k] + re.sub(r"\[\s*None\s*\]", "[research_loop, content_builder]", t[k:], count=1)
    return t
edit("agents/orchestrator/agent.py", orch)

print("\n=== Task 5: run_local.sh ===")
def runl(t):
    return (t.replace("YOUR_GCLOUD_COMMAND_HERE", "$(gcloud config get-value project)")
             .replace("YOUR_REGION_VARIABLE_HERE", "us-central1")
             .replace("YOUR_LOCAL_JUDGE_URL", "http://localhost:8002/a2a/agent/.well-known/agent-card.json")
             .replace("YOUR_LOCAL_CONTENT_BUILDER_URL", "http://localhost:8003/a2a/agent/.well-known/agent-card.json"))
edit("run_local.sh", runl)
PY

echo; echo "--- check: lines still containing None/TODO/YOUR_ in the edited files ---"
grep -nE "None|TODO|YOUR_" agents/*/agent.py agents/*/skills/SKILL.md \
  agents/content_builder/skills/scripts/check_format.py run_local.sh | grep -v "Optional\|is None\|= None)" | head -20
for f in agents/*/agent.py agents/content_builder/skills/scripts/check_format.py; do
  python3 -m py_compile "$f" 2>&1 | head -3; done

up() { gcloud storage cp "$1" "$B/$1" >/dev/null && echo "   uploaded $1"; }
echo; echo "=== uploading ==="
up agents/researcher/skills/SKILL.md; up agents/researcher/agent.py
echo ">>> Task 1 DONE - click 'Check my progress'"
up agents/judge/agent.py
echo ">>> Task 2 DONE - click 'Check my progress'"
up agents/content_builder/skills/scripts/check_format.py
up agents/content_builder/skills/SKILL.md; up agents/content_builder/agent.py
echo ">>> Task 3 DONE - click 'Check my progress'"
up agents/orchestrator/agent.py
echo ">>> Task 4 DONE - click 'Check my progress'"
up run_local.sh

# ================= Task 5: local run (background) =================
echo; echo "=== Task 5: starting the 4 local agents + web UI in the background ==="
chmod +x run_local.sh
pkill -f uvicorn 2>/dev/null; sleep 1
(set -a; source .env; nohup ./run_local.sh > ~/run_local.log 2>&1 &)
for i in $(seq 1 30); do
  curl -s -o /dev/null http://localhost:8080 && break; sleep 5; done
tail -5 ~/run_local.log
cat <<'MSG'
>>> Task 5 - DO THIS NOW (1 minute), the script keeps deploying meanwhile:
    Cloud Shell toolbar > Web Preview > Preview on port 8080
    type:  Create a course about the history of coffee.   and submit
    wait until the course appears, then click 'Check my progress' for Task 5
    (local log: tail -f ~/run_local.log)
MSG

# ================= Task 6: Cloud Run =================
echo; echo "=== Task 6: deploying researcher, judge, content-builder ==="
ENV="GOOGLE_CLOUD_PROJECT=$P,GOOGLE_CLOUD_LOCATION=global,GOOGLE_GENAI_USE_VERTEXAI=true,MODEL=$MODEL"
dep() {   # dep <service> <source> <env> : up to 3 tries, stop if it never deploys
  for i in 1 2 3; do
    gcloud run deploy "$1" --source "$2" --region "$REGION" --allow-unauthenticated \
      --memory 512Mi --set-env-vars "$3" --quiet > ~/deploy_$1.log 2>&1 \
      && { echo "   $1 deployed"; return 0; }
    echo "   $1 failed ($i/3):"; tail -3 ~/deploy_$1.log; sleep 20
  done
  echo "STOP: $1 did not deploy - send Claude ~/deploy_$1.log"; exit 1
}
U() { gcloud run services describe "$1" --region "$REGION" --format='value(status.url)' 2>/dev/null; }
# one at a time: parallel first deploys clash on the shared container repository
# ALWAYS deploy (never skip): the lab may pre-create placeholder services with these
# names, and the checker only counts services deployed from your code
for s in researcher judge content-builder; do
  dep $s "agents/$(echo $s | tr - _)/" "$ENV"
done
C="/a2a/agent/.well-known/agent-card.json"
echo "=== deploying orchestrator ==="
dep orchestrator agents/orchestrator/ \
 "RESEARCHER_AGENT_CARD_URL=$(U researcher)$C,JUDGE_AGENT_CARD_URL=$(U judge)$C,CONTENT_BUILDER_AGENT_CARD_URL=$(U content-builder)$C,$ENV"
echo "=== deploying course-creator frontend ==="
dep course-creator app "AGENT_SERVER_URL=$(U orchestrator),GOOGLE_CLOUD_PROJECT=$P"
gcloud run services list --region "$REGION" \
  --format="table(metadata.name,status.conditions[0].status:label=READY,status.url)"
FE=$(U course-creator)
cat <<MSG

>>> Task 6 - open this URL in a new tab:  $FE
    enter:  The history of cloud computing   and click Start Building
    wait for the course module, then click 'Check my progress' for Task 6
>>> All done.
MSG
LABEOF
echo "SCRIPT SAVED OK - now run: bash agents_lab.sh"
```


bash agents_lab.sh
