```/text
set +H
cat > adk_lab.sh <<'LABEOF'
#!/bin/bash
# Engineer AI Agents with Agent Development Kit (ADK): Challenge Lab (GSP540)
# Run: bash adk_lab.sh   (1 question, then runs by itself, ~5 min)
ask() { local r; read -e -r -p "$2 [$3]: " r; printf -v "$1" '%s' "${r:-$3}"; }
P=$(gcloud config get-value project 2>/dev/null)
[ -z "$P" ] && { echo "run: gcloud config set project <Project ID>"; exit 1; }
echo "=== YELLOW value from Task 2 (.env line MODEL=...) ==="
ask MODEL "MODEL" "gemini-3.6-flash"; export MODEL
export PATH=$PATH:"/home/${USER}/.local/bin"

# ================= Task 1 =================
echo; echo "=== Task 1: install ADK + project ==="
python3 -m pip install -q --user google-adk 2>&1 | tail -1
cd ~
[ -d adk_project ] || { gcloud storage cp "gs://$P-bucket/adk_project.zip" . \
  && unzip -q -o adk_project.zip; } || { echo "STOP: download failed"; exit 1; }
cd ~/adk_project || exit 1
python3 -m pip install -q --user -r requirements.txt 2>&1 | tail -1
for a in my_google_search_agent geo_validator llm_auditor; do
  cat > $a/.env <<EOF
GOOGLE_GENAI_USE_ENTERPRISE=true
GOOGLE_GENAI_USE_VERTEXAI=true
GOOGLE_CLOUD_PROJECT=$P
GOOGLE_CLOUD_LOCATION=global
MODEL=$MODEL
EOF
done
echo "   wrote .env for the 3 agents"

# ================= code edits =================
python3 - <<'PY'
import re, os, pathlib
def edit(path, fn):
    p = pathlib.Path(path); old = p.read_text(); new = fn(old)
    p.write_text(new)
    print(f"   {path}: " + ("updated" if new != old else "no change needed"))
def imp_end(lines):   # index just after the last top-level import
    last, i = -1, 0
    while i < len(lines):
        l = lines[i]
        if l.startswith(("import ", "from ")):
            if "(" in l and ")" not in l:
                while i < len(lines) and ")" not in lines[i]: i += 1
            last = i
        i += 1
    return last + 1
def add_import(t, line):
    if line in t: return t
    lines = t.split("\n"); lines.insert(imp_end(lines), line)
    return "\n".join(lines)

# Task 2: google_search tool
def search(t):
    t = add_import(t, "from google.adk.tools import google_search")
    if re.search(r"tools\s*=\s*\[\s*google_search\s*\]", t): return t
    t2 = re.sub(r"tools\s*=\s*\[[^\]]*\]", "tools=[google_search]", t, count=1)
    if t2 != t: return t2
    return re.sub(r"(root_agent\s*=\s*Agent\()", r"\1\n    tools=[google_search],", t, count=1)
edit("my_google_search_agent/agent.py", search)

# Task 4: CountryCapital output schema, no transfers
def geo(t):
    t = add_import(t, "from pydantic import BaseModel")
    # class goes right after the imports, so it exists before main() runs
    t = re.sub(r"\n*class CountryCapital\(BaseModel\):\n(?:[ \t]+.*(?:\n|$))+",
               "\n", t)
    lines = t.split("\n")
    lines.insert(imp_end(lines),
                 "\n\nclass CountryCapital(BaseModel):\n    capital: str\n\n")
    t = re.sub(r"\n{4,}", "\n\n\n", "\n".join(lines))
    M = os.environ.get("MODEL", "gemini-3.6-flash")
    t = re.sub(r'model\s*=\s*"gemini[^"]*"', f'model="{M}"', t)
    for k, v in (("output_schema", "CountryCapital"),
                 ("disallow_transfer_to_parent", "True"),
                 ("disallow_transfer_to_peers", "True")):
        if re.search(rf"{k}\s*=", t):
            t = re.sub(rf"{k}\s*=\s*[^,\n)]+", f"{k}={v}", t)
        else:
            t = re.sub(r"(\w+\s*=\s*(?:Llm)?Agent\()", rf"\1\n    {k}={v},", t, count=1)
    return t
edit("geo_validator/agent.py", geo)

# Task 5: add reviser_agent
def aud(t):
    t = re.sub(r"^#\s*(from\s+\S+\s+import\s+reviser_agent.*)$", r"\1", t, flags=re.M)
    if not re.search(r"^from\s+\S+\s+import\s+.*reviser_agent", t, re.M):
        t = add_import(t, "from .sub_agents.reviser import reviser_agent")
    if not re.search(r"sub_agents\s*=\s*\[[^\]]*reviser_agent", t):
        t = re.sub(r"(sub_agents\s*=\s*\[[^\]]*?)(\s*,?\s*\])",
                   r"\1, reviser_agent]", t, count=1)
        t = t.replace("[, reviser_agent]", "[reviser_agent]")
    return t
edit("llm_auditor/agent.py", aud)
PY
for f in my_google_search_agent/agent.py geo_validator/agent.py llm_auditor/agent.py; do
  python3 -m py_compile "$f" || { echo "STOP: $f has an error - send Claude: cat -n $f"; exit 1; }
done

# ================= Task 2 + 5: adk web, ask through its API =================
ask_web() {   # ask_web <app> <question>
  S=s$RANDOM
  curl -s -X POST "http://127.0.0.1:8000/apps/$1/users/user/sessions/$S" \
    -H "Content-Type: application/json" -d '{}' >/dev/null
  curl -s -m 300 -X POST http://127.0.0.1:8000/run -H "Content-Type: application/json" \
    -d "{\"app_name\":\"$1\",\"user_id\":\"user\",\"session_id\":\"$S\",\"new_message\":{\"role\":\"user\",\"parts\":[{\"text\":\"$2\"}]}}" \
  | python3 -c '
import sys, json
try: ev = json.load(sys.stdin)
except Exception: print("   (no answer)"); sys.exit()
for e in ev:
    for p in (e.get("content") or {}).get("parts", []):
        if p.get("text"): print("   [" + e.get("author","?") + "]", p["text"][:300].replace("\n"," "))'
}
echo; echo "=== Task 2: adk web + Travel Scout question ==="
pkill -f "adk web" 2>/dev/null; sleep 1
nohup adk web --allow_origins "regex:https://.*\.cloudshell\.dev" > ~/adk_web.log 2>&1 &
for i in $(seq 1 30); do curl -s -o /dev/null http://127.0.0.1:8000 && break; sleep 2; done
ask_web my_google_search_agent "What are some major events in Tokyo in 2025?"
echo ">>> Task 2 DONE - click 'Check my progress' (Run the agent using the ADK's Web UI)"

echo; echo "=== Task 3: adk run (command line) ==="
printf 'What is the currency exchange rate for Japan?\nexit\n' \
  | timeout 180 adk run my_google_search_agent 2>&1 | tail -8
echo ">>> Task 3 DONE - click 'Check my progress' (command-line interface)"

echo; echo "=== Task 4: run geo_validator programmatically ==="
timeout 180 python3 geo_validator/agent.py 2>&1 | tail -8
echo ">>> Task 4 DONE - click 'Check my progress' (Run an agent programmatically)"

echo; echo "=== Task 5: llm_auditor (Critic -> Reviser) ==="
ask_web llm_auditor "Double check this: You can take a direct train from Hawaii to Japan."
echo ">>> Task 5 DONE - click 'Check my progress' (Preview a multi-agent example)"
cat <<'MSG'

Optional (only if a check stays grey): the dev UI is still running.
 Cloud Shell > Web Preview > Change port > 8000, pick the agent and ask
 the same question in the browser, then click 'Check my progress' again.
>>> All done.
MSG
LABEOF
echo "SCRIPT SAVED OK - now run: bash adk_lab.sh"
```

bash adk_lab.sh
