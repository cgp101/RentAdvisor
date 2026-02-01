## RentAdvisor
An intelligent Telegram chatbot that provides rental affordability insights for US locations. Users can query by city name or ZIP code to get average rent data. The system uses a multi-model council architecture for ranking recommendations, ensuring balanced and transparent advice for financial decisions. 

## Tech Stack 
- n8n to automate workflow 
- Telegram Bot API for user interaction 
- Google Sheets for data storage
- Ollama  --> We can use a cloud provider instead of a local host.
- Zillow Research (Zillow ZORI data)
- Python for n8n code blocks
- Ngrok for tunnel between ollama and n8n

## Data Source 
Zillow ZORI Dataset -- Zillow Observed Rent Index dataset
link: https://www.zillow.com/research/data/
This dataset is smoothed and seasonally adjusted This means removes random nosies and unexpected one-time spikes from data. Smoothing the data gives us cleaner and more stable trends. 
Seasonally adjusted means removes seasonal patterns such high rent prices in summer and rent dips in winter, giving us accurate market trends rather than displaying cyclic patterns. 

Why this dataset:
1. No anomalies to worry about. 
2. Helps us identity market movements rather, finding out cyclic trends. 

### Data Schema 
| Column Name                 | Data Type | Description                                |
| --------------------------- | --------- | ------------------------------------------ |
| `RegionID`                  | Integer   | Zillow’s internal region identifier        |
| `SizeRank`                  | Integer   | Rank by population (0 = largest)           |
| `RegionName`                | String    | ZIP code (for example, `"94107"`)          |
| `RegionType`                | String    | Always `"zip"`                             |
| `StateName`                 | String    | State abbreviation (for example, `"CA"`)   |
| `City`                      | String    | City name (for example, `"San Francisco"`) |
| `Metro`                     | String    | Metropolitan area                          |
| `CountyName`                | String    | County name                                |
| `2015-01-31` … `2024-12-31` | Float     | Monthly rent values in USD                 |

## Overall automation flow and about each node 
### Node 1: Telegram Trigger

**Purpose:** Receives incoming messages from users

**Configuration:**
- Update type: Message
- Bot credentials: Rent_looker bot via BotFather

**Output:**
```json
{
  "message": {
    "text": "San Francisco",
    "chat": {
      "id": 8259885354,
      "first_name": "Charit"
    }
  }
}
```

### Node 2: Name_Normalization 
**Purpose:** Receives incoming messages from users and extracts city/zipcode from the text 
| Input | Output | Confidence |
|-------|--------|------------|
| `10467` | zip: 10467 | high |
| `rent in 10467-1234` | zip: 10467 | high |
| `Bronx` | city: Bronx | high |
| `the bronx` | city: Bronx | high |
| `bx` | city: Bronx | high |
| `nyc` | city: New York City | high |
| `what's rent in queens` | city: Queens | high |
| `brooklyn ny` | city: Brooklyn | high |
| `astoria` | city: Queens | high |
| `sf` | city: San Francisco | high |
| `chicago il` | city: Chicago | high |
| `CA` | state: California | high |
| `affordable apartments in denver` | city: Denver | medium |

## Node 3: Google Sheets Lookup

**Purpose:** Fetches rent data matching user's search term

**Configuration:**
- Operation: Get Row(s)
- Document: Zillow ZORI spreadsheet
- Sheet: Zip_Zori_willow_city
- Filter Column: RegionName (for ZIP) or City (for city name)
- Filter Value: `{{ $json.search_term }}`

---

### Node 4: Average Rent Calculator

**Purpose:** Aggregates rent data and handles errors

```json
return [{"json": {
    "error": None,
    "city": city_name,
    "metro": metro_name,
    "county": county_name,
    "avg_rent": avg_rent,
    "zip_count": c,
    "zips_included": zips
}}]
```

### Node 5: IF Node (Error Router-- Error_usr_inp)

**Condition:** `{{ $json.error }}` is empty

| Branch | Condition | Action |
|--------|-----------|--------|
| TRUE | No error | Send rent data to user |
| FALSE | Has error | Send error message |

### Node 6: Telegram Response Message
Either it sends Success Message or error message 

## Council Layer

### Node 7: LLM-if triggers Node

**Purpose:** Routes to council layer when user responds "yes"

| Field | Value |
|-------|-------|
| value1 | `{{$('Telegram Trigger').first().json.message.text.toLowerCase()}}` |
| Condition | matches regex |
| value2 | ^(yes|yeah|yea|yep|yup|sure|okay|ok|for sure|of course|cool)$ |

---
## Iteration 1 → Mistral 7B + Llama 3.1 8B Council (rent_advisor_v3.json)

### Mistral 7B (RentAdvisor)
| Attribute | Value |
|-----------|-------|
| Name | RentAdvisor |
| Focus | Livability |
| Perspective | "Is this a good place to live?" |
| Tone | Conversational, analytical, practical, friendly |
| Temperature | 0.30 |

**Analyzes:** Affordability, neighborhoods, safety, money-saving tips, alternative cities

### Llama 3.1 8B (Market Analyst)
| Attribute | Value |
|-----------|-------|
| Name | Market Analyst |
| Focus | Investment |
| Perspective | "Is this a good deal right now?" |
| Tone | Analytical, numbers-focused, direct, objective |
| Temperature | 0.25 |

**Analyzes:** Market valuation, supply-demand, market cycle, timing verdict (NOW/WAIT), arbitrage opportunities

### Techniques Used (Both Models)
- Persona prompting
- Chain of thought
- Self-evaluation verification

> 📄 **Full prompt engineering details:** See `Prompt_Eng_models.md`
> 📄 **Python node codes:** See `Python_node_codes.ipynb`

### Hardware Limitation Note

Mistral 7B (~4.4GB RAM) and Llama 3.1 8B (~4.9GB RAM) caused system crashes on local hardware. 

**Current workaround:** Telegram response message is hardcoded instead of LLM-generated while testing smaller models.

Council layer architecture remains intact — just swap models when hardware allows or deploy to cloud.
### Screenshots of Iternation -1 


## Iteration 2 → phi4-mini:3.8b + phi3:3.8b Council (rent_advisor_v4.json)

### What Changed from v3
- Swapped Mistral 7B → phi4-mini:3.8b (RentAdvisor)
- Swapped Llama 3.1 8B → phi3:3.8b (Market Analyst)
- Same council architecture, lighter models for local hardware

### Why These Models
- **Strong reasoning:** Both phi models are optimized for reasoning tasks despite small size
- **Lightweight:** 2.2GB and 2.5GB each vs 4-5GB+ for Mistral/Llama
- **Local-friendly:** Can run both models simultaneously without crashing

### Models
| Role | v3 Model | v4 Model | Size |
|------|----------|----------|------|
| RentAdvisor | Mistral 7B | phi4-mini:3.8b | ~2.5GB |
| Market Analyst | Llama 3.1 8B | phi3:3.8b | ~2.2GB |

### Merge Node
**Purpose:** Combines outputs from both LLM models into single message

**Configuration:**
- Mode: Append
- Number of Inputs: 2

**Output:** 2 items (phi4-mini content + phi3 content)

### Telegram Council Message Node
**Purpose:** Sends combined council analysis to user
**Text Field (Expression):**
```
📊 Council Summary
🏠 RentAdvisor:
{{ $('phi4-min').first().json.content.replace(/\*/g, '').replace(/#/g, '') }}
---
📈 Market Analyst:
{{ $('phi3:3').first().json.content.replace(/\*/g, '').replace(/#/g, '') }}
```
### Screenshots of Iternation - 2




## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         USER INTERFACE                              │
│                        (Telegram Bot)                               │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      INPUT PROCESSING                               │
│              ┌─────────────────────────────────────┐                │
│              │    Name Normalization (Enhanced)    │                │
│              │  • Regex ZIP extraction             │                │
│              │  • State abbreviation detection     │                │
│              │  • City aliases & boroughs          │                │
│              │  • Noise word removal               │                │
│              │  • Multi-word city detection        │                │
│              └─────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       DATA LAYER                                    │
│              ┌─────────────────────────────────────┐                │
│              │      Google Sheets                  │                │
│              │  • Zillow ZORI Dataset              │                │
│              │  • ZIP-level rent data              │                │
│              │  • City, Metro, County mapping      │                │
│              └─────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    PROCESSING LAYER                                 │
│              ┌─────────────────────────────────────┐                │
│              │   Average Rent Calculator           │                │
│              │  • Aggregates ZIP-level data        │                │
│              │  • Handles missing data             │                │
│              │  • Error handling                   │                │
│              └─────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     ERROR HANDLING                                  │
│              ┌─────────────────────────────────────┐                │
│              │       IF Node (Router)              │                │
│              │  • Success → Send rent data         │                │
│              │  • Error → Send error message       │                │
│              └─────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      OUTPUT LAYER                                   │
│                    (Telegram Response)                              │
│              "Avg rent is $X... Want more details?"                 │
└─────────────────────────────────────────────────────────────────────┘
                                │
                          User: "yes"
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      COUNCIL LAYER                                  │
│    ┌────────────────────┐       ┌────────────────────┐              │
│    │    LLM Model-1     │       │       LLM Model-2  │              │
│    │    (RentAdvisor)   │       │  (Market Analyst)  │              │
│    │                    │       │                    │              │
│    │  Livability Focus  │       │  Investment Focus  │              │
│    │  - Neighborhoods   │       │  - Market cycles   │              │
│    │  - Safety          │       │  - Supply/demand   │              │
│    │  - Lifestyle       │       │  - Timing verdict  │              │
│    └────────────────────┘       └────────────────────┘              │
│                    │                       │                        │
│                    └───────────┬───────────┘                        │
│                                ▼                                    │
│                           MERGE NODE                                |
|                                |                                    |
|                                ▼                                    |
|              Send a message back to user on Telegram                |
└─────────────────────────────────────────────────────────────────────┘
```

## Ollama + n8n Cloud Setup

### Step 1: Launch Ollama for External Traffic
**Linux/macOS:**
```bash
OLLAMA_HOST=0.0.0.0 ollama serve
## Check online for windows
```
### Step 2: Create ngrok Tunnel ( inna new terminak window)

```bash
ngrok http 11434
```
Copy the forwarding URL (e.g., `https://random-id.ngrok-free.app`)

### Step 3: Configure n8n Credentials

1. In n8n: Credentials → Add Credential → Ollama
2. Base URL: Paste ngrok URL
3. Models auto-populate from your local Ollama

**Note:** ngrok URL changes on restart. Update n8n credentials accordingly.


## How to use 
Import `rent_advisor_v4.json` into n8n
2. Configure credentials (Telegram, Google Sheets, Ollama)
3. Activate workflow
## Screenshots
Overall flow with Ollama
<img width="1326" height="544" alt="image" src="https://github.com/user-attachments/assets/906615aa-6476-4821-ba05-0e778a0f92ca" />


## Status: In Progress

