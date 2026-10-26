## task 1 --------------

```text
cat > lab4.sh <<'LABEOF'
#!/bin/bash
# Task 1 solver: Explore Generative AI with the Gemini API (Agent Platform / Vertex AI)
ask() { local r; read -e -r -p "$2 [$3]: " r; printf -v "$1" '%s' "${r:-$3}"; }

PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
[ -z "$PROJECT_ID" ] && { echo "STOP: no project set. Run: gcloud config set project <Project ID from the lab panel>"; exit 1; }
echo "Project: $PROJECT_ID"

echo
echo "=== Values from the lab's environment-variable box (press Enter to keep the default) ==="
ask LOCATION "LOCATION" "global"
ask MODEL_ID "MODEL_ID (the model named in step 3)" "gemini-3.5-flash"
ask QUESTION "Question to ask" "Why is the sky blue?"
if [ "$LOCATION" = "global" ]; then API_ENDPOINT="aiplatform.googleapis.com"; else API_ENDPOINT="${LOCATION}-aiplatform.googleapis.com"; fi

echo
echo "=== Enabling the APIs ==="
gcloud services enable aiplatform.googleapis.com notebooks.googleapis.com compute.googleapis.com storage.googleapis.com bigquery.googleapis.com

URL="https://${API_ENDPOINT}/v1/projects/${PROJECT_ID}/locations/${LOCATION}/publishers/google/models/${MODEL_ID}"
BODY=$(jq -n --arg q "$QUESTION" '{contents:[{role:"user",parts:[{text:$q}]}]}')

call() {   # call <method> <output file>
  for i in 1 2 3 4 5 6; do
    curl -s -X POST -H "Authorization: Bearer $(gcloud auth print-access-token)" \
      -H "Content-Type: application/json" "${URL}:$1" -d "$BODY" -o "$2"
    grep -q '"error": *{' "$2" || return 0
    echo "   attempt $i failed, retrying in 15s (a newly enabled API can take a minute)..."
    sleep 15
  done
  return 1
}

echo
echo "=== Task 1: calling $MODEL_ID ==="
call generateContent gemini-response.json || { echo "STOP: the call failed:"; head -c 800 gemini-response.json; exit 1; }
call streamGenerateContent gemini-stream-response.json || true
echo "--- Answer ---"
jq -r '[.candidates[0].content.parts[]?.text] | join("")' gemini-response.json | head -c 1200
echo
echo ">>> Task 1 done. Click 'Check my progress' for Task 1 on the lab page."
LABEOF
```

bash lab4.sh



## TASK 3 --------------------------------------------------------


```text
# Task 3.1
# use the following documentation to assist you complete this cell
# https://cloud.google.com/vertex-ai/generative-ai/docs/model-reference/function-calling
# Load Gemini 3.5 Flash Model
model_id = "gemini-3.5-flash"
```


```text
# Task 3.2
# use the following documentation to assist you complete this cell
# https://cloud.google.com/vertex-ai/generative-ai/docs/model-reference/function-calling
get_current_weather_func = FunctionDeclaration(
    name="get_current_weather",
    description="Get the current weather in a given location",
    parameters={
        "type": "object",
        "properties": {
            "location": {
                "type": "string",
                "description": "Location"
            }
        }
    },
)
```


```text
# Task 3.3
# use the following documentation to assist you complete this cell
# https://cloud.google.com/vertex-ai/generative-ai/docs/model-reference/function-calling
weather_tool = Tool(
    function_declarations=[get_current_weather_func],
)
```



```text
# Task 3.4
# use the following documentation to assist you complete this cell
# https://cloud.google.com/vertex-ai/generative-ai/docs/model-reference/function-calling
prompt = "What is the weather like in Boston?"
response = client.models.generate_content(
    model=model_id,
    contents=prompt,
    config=GenerateContentConfig(
        tools=[weather_tool],
        temperature=0,
    ),
)
response
```


```text
# Task 4.1
# Load the correct Gemini model use the following documentation to assist:
# https://cloud.google.com/vertex-ai/docs/generative-ai/multimodal/overview#supported-use-cases
# Load Gemini 3.5 Flash Model
multimodal_model = "gemini-3.5-flash"
```


```text
# Task 4.2 Generate a video description
# In this cell, update the prompt to ask Gemini to describe the video URL referenced.
# You can use the documentation at the following link to assist.
# https://cloud.google.com/vertex-ai/docs/generative-ai/multimodal/sdk-for-gemini/gemini-sdk-overview-reference#generate-content-from-video
# https://cloud.google.com/vertex-ai/generative-ai/docs/model-reference/inference#sample-requests-text-stream-response
# Video URI: gs://github-repo/img/gemini/multimodality_usecases_overview/mediterraneansea.mp4

prompt = """
Describe this video.
What is shown in this video?
Where should I go to see it?
What are the top 5 places in the world that look like this?
"""
video = Part.from_uri(
    file_uri="gs://github-repo/img/gemini/multimodality_usecases_overview/mediterraneansea.mp4",
    mime_type="video/mp4",
)
contents = [prompt, video]

responses = client.models.generate_content_stream(
    model=multimodal_model,
    contents=contents
)

print("-------Prompt--------")
print_multimodal_prompt(contents)

print("\n-------Response--------")
for response in responses:
    print(response.text, end="")
```
