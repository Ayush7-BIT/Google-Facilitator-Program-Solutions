Task 2.  ----------------------------------------------------------------------------

```/text
from google.genai.types import GenerateContentConfig, Tool, ToolCodeExecution

# 1. Define the code execution tool
code_execution_tool = Tool(code_execution=ToolCodeExecution())

sneaker_prices = [120, 150, 110, 180, 135, 95, 210, 170, 140, 165]

# 2. Define the prompt with the code to be executed
PROMPT = f"""Calculate the average price of the sneakers using this list of sneaker prices: {sneaker_prices}.
Generate and run code for the calculation."""

response = client.models.generate_content(
    model=MODEL_ID,
    contents=PROMPT,
    config=GenerateContentConfig(
        tools=[code_execution_tool],
        temperature=0,
    ),
)
```


Task 3 -----------------------------------------------------------------------------------


```/text
from google.genai.types import GenerateContentConfig, GoogleSearch, Tool

# 1. Define the Google Search tool
google_search_tool = Tool(google_search=GoogleSearch())

# 2. Define the prompt with grounding
prompt = "What are the key features of the Nike Air Jordan XXXVI?"

# 3. Generate a response with grounding
response = client.models.generate_content(
    model=MODEL_ID,
    contents=prompt,
    config=GenerateContentConfig(tools=[google_search_tool]),
)

# Print the response
print(response.text)
```



Task 4 -------------------------------------------------------------------------------------

```/text
from google.genai.types import GenerateContentConfig, GoogleSearch, Tool
from IPython.display import Markdown, display
import json

# 1. Define the basketball sneaker models
sneaker_models = ["Under Armour Curry Flow 9", "Sketchers Slip-ins: Glide-Step Pro"]

# 2. Define the online retailers
retailers = ["Foot Locker", "Nordstrom"]

# 3. Initialize an empty list to store the extracted data
extracted_data = []

# 4. Loop through the sneaker models and retailers to extract pricing information
for model in sneaker_models:
    for retailer in retailers:
        # 5. Construct the search query
        query = f"Find the current price of the {model} basketball sneaker sold by {retailer}. If the price found is 0, return the regular retail price listed by {retailer} for this product."

        # 6. Use Response Schema to extract the data
        response = client.models.generate_content(
            model=MODEL_ID,
            contents=query,
            config=GenerateContentConfig(
                response_schema=response_schema,
                response_mime_type="application/json"
            )
        )

        print(response.text)
```

