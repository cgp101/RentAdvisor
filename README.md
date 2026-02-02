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

- **Ollama Models**

<img width="592" height="111" alt="(base) cgp@Charit-MBPro RentAdvisor  ollama" src="https://github.com/user-attachments/assets/2a61631b-562c-47e5-95eb-efde2dba9ee8" />

<img width="595" height="203" alt="image" src="https://github.com/user-attachments/assets/be9696c5-3483-4bdd-8152-124743646ae1" />


### Screenshots of Iternation -1 

- **Iternation -1 Flow**
  <img width="1257" height="483" alt="Pasted Graphic 8" src="https://github.com/user-attachments/assets/f842ec09-9567-408f-b0b5-f6cca1600c70" />
- **City name**
  <img width="1051" height="552" alt="Avo sent in Toledo is 1120 0037 USD" src="https://github.com/user-attachments/assets/cc180924-6e8f-4c9c-af1c-46eb225222cc" />
  <img width="1051" height="552" alt="A RentAdviser (Mural 4 4 Gf) and Market Analyst (Lama 45" src="https://github.com/user-attachments/assets/84b70b48-c511-4d75-bc08-e375901b0db8" />

- **Zip Code**
  <img width="997" height="652" alt="saved on 1 Zip codes, ist of zip codes are" src="https://github.com/user-attachments/assets/21680c13-00ae-44d6-babc-f3ecbca06852" />

 <img width="997" height="652" alt="• Tming Wat for correction of find alternative" src="https://github.com/user-attachments/assets/e5d0632b-b2a5-467d-b822-bc6adf116085" />


   
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
#### Parameters: 
- phi4-mini:3.8b:
  <img width="993" height="610" alt="image" src="https://github.com/user-attachments/assets/f1ed7e0b-0c8d-4605-bd0c-4eb04692ff91" />

- phi3:3.8b:
  <img width="993" height="610" alt="image" src="https://github.com/user-attachments/assets/7de666c1-ef92-4c3d-8eea-c6119bf36ef8" />



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
- **City name**
  ￼<img width="1068" height="601" alt="Pasted Graphic 12" src="https://github.com/user-attachments/assets/57802c60-da93-4bd5-a25e-5f8a11d34d24" />

   <img width="1051" height="552" alt="H Councl Summar" src="https://github.com/user-attachments/assets/62e2cba3-61b4-40a5-9372-a5403946195a" />

- **Zip Code**
  <img width="1068" height="601" alt="This message was sere automatically with ne" src="https://github.com/user-attachments/assets/0dfd5744-fb13-4365-bae3-872d0509a422" />

  <img width="1068" height="601" alt="Verdiet Altardsele (hersatie" src="https://github.com/user-attachments/assets/456ca589-8c35-40fb-924c-8458d49cd644" />


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
2. Base URL: Paste the ngrok URL
3. Models auto-populate from your local Ollama

**Note:** ngrok URL changes on restart. Update n8n credentials accordingly.


## How to use 
1. Import `rent_advisor_v4.json` into n8n
2. Configure credentials (Telegram, Google Sheets, Ollama, and Ngrok)
3. Activate workflow
## Screenshots
Overall flow
<img width="1242" height="527" alt="image" src="https://github.com/user-attachments/assets/5eb3eb6f-746b-4e9b-926c-cbb92bff25be" />



## Status: In Progress

