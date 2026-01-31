## RentAdvisor
An intelligent Telegram chatbot that provides rental affordability insights for US locations. Users can query by city name or ZIP code to get average rent data. The system uses a multi-model council architecture for ranking recommendations, ensuring balanced and transparent advice for financial decisions. 

## Tech Stack 
- n8n to automate workflow 
- Telegram Bot API for user interaction 
- Google Sheets for data storage
- Ollama (Llama 3.1, Mistral) --> We can use a cloud provider instead of a local host.
- Zillow Research (Zillow ZORI data)
- Python for n8n code blocks

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



## How to use 
Import `rent_advisor_v1.json` into n8n
2. Configure credentials (Telegram, Google Sheets, Ollama)
3. Activate workflow
## Screenshots
Overall flow with Ollama
<img width="1326" height="544" alt="image" src="https://github.com/user-attachments/assets/906615aa-6476-4821-ba05-0e778a0f92ca" />


## Status: In Progress

