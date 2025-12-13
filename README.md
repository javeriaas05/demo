================================================
# 1. INSTALL REQUIRED LIBRARIES
# ================================================
!pip install requests beautifulsoup4 ipywidgets --quiet

import requests
from bs4 import BeautifulSoup
import re
import json
import ipywidgets as widgets
from IPython.display import display
import os
import time

# ================================================
# 2. SAFELY INPUT GROQ API KEY
# ================================================
API_KEY = os.environ.get("GROQ_API_KEY")
if not API_KEY:
    API_KEY = input("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").strip()
os.environ["GROQ_API_KEY"] = API_KEY

# CORRECT GROQ CHAT COMPLETION ENDPOINT
API_URL = "https://api.groq.com/openai/v1/chat/completions"


# ================================================
# 3. TEXT CLEANING FUNCTION
# ================================================
def clean_text(text):
    text = re.sub(r"\s+", " ", text)
    text = re.sub(r"http\S+", "", text)
    text = re.sub(r"[^A-Za-z0-9 .,-]", "", text)
    return text.strip()


# ================================================
# 4. WEB SCRAPING FUNCTION
# ================================================
def scrape_web(query):
    sources = [
        f"https://www.bbc.co.uk/search?q={query}",
        f"https://www.reuters.com/site-search/?query={query}"
    ]

    combined_text = ""
    for url in sources:
        try:
            res = requests.get(url, timeout=6)
            soup = BeautifulSoup(res.text, "html.parser")
            paragraphs = soup.find_all("p")
            extracted = " ".join([p.get_text() for p in paragraphs[:5]])
            combined_text += extracted + " "
        except Exception:
            pass
        time.sleep(0.5)
    return combined_text.strip()


# ================================================
# 5. CALL LLM USING GROQ API (updated model)
# ================================================
def verify_with_llm(news_text, scraped_text):
    prompt = f"""
Compare the following news with trusted scraped information.

NEWS PROVIDED BY USER:
{news_text}

INFORMATION FROM RELIABLE SOURCES:
{scraped_text}

Give the result in this format:
Authenticity Score: (0-100)
Final Decision: REAL / FAKE / POSSIBLY MISLEADING
Explanation: (2–3 lines)
"""

    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json"
    }

    data = {
        # Use a supported Groq production model. Change if your account prefers another model.
        "model": "llama-3.3-70b-versatile",
        "messages": [
            {"role": "user", "content": prompt}
        ],
        "temperature": 0.2,
        # You can set max_output_tokens if desired, e.g. "max_output_tokens": 512
    }

    try:
        response = requests.post(API_URL, headers=headers, json=data, timeout=30)

        if response.status_code != 200:
            # Return full response text for debugging (useful in Colab)
            return f"API Request Failed: Status code {response.status_code}, {response.text}"

        resp = response.json()

        # Robust parsing for chat completions
        if "choices" in resp and len(resp["choices"]) > 0:
            # Many Groq responses follow OpenAI-like shape:
            # resp["choices"][0]["message"]["content"]
            choice = resp["choices"][0]
            # handle a few possible formats defensively
            if "message" in choice and "content" in choice["message"]:
                return choice["message"]["content"].strip()
            elif "text" in choice:
                return choice["text"].strip()
            else:
                return json.dumps(choice, indent=2)
        elif "error" in resp:
            return "API Error: " + resp["error"].get("message", str(resp["error"]))
        else:
            return "Unexpected API response: " + json.dumps(resp)

    except requests.exceptions.RequestException as e:
        return f"Network/API request failed: {e}"


# ================================================
# 6. MARKDOWN TO HTML
# ================================================
def md_to_html(md):
    html = md.replace("\n", "<br>")
    html = html.replace("### ", "<h3>")
    html = html.replace("## ", "<h2>")
    html = html.replace("# ", "<h1>")
    return html


# ================================================
# 7. DASHBOARD UI
# ================================================
title_md = """
# 🔍 AI-Based Content Verification Tool
### Fake News Detector (Colab Dashboard)
"""
title_html = widgets.HTML(value=md_to_html(title_md))

input_box = widgets.Textarea(
    value="Enter news here...",
    description="News:",
    layout=widgets.Layout(width="100%", height="120px")
)

run_button = widgets.Button(
    description="Verify",
    button_style='success',
    icon='search'
)

output_area = widgets.HTML(value="")


# ================================================
# 8. BUTTON LOGIC
# ================================================
def on_button_click(b):
    news = input_box.value.strip()
    if len(news) < 10:
        output_area.value = "<b style='color:red;'>Please enter valid news text.</b>"
        return

    output_area.value = "<b>Scraping reliable sources...</b>"
    scraped = scrape_web(news[:40])  # increase query length sent to scrapers

    output_area.value = "<b>Analyzing with AI...</b>"
    cleaned = clean_text(news)
    result = verify_with_llm(cleaned, scraped)

    output_area.value = f"""
    <h3>✅ RESULT</h3>
    <div style='padding:12px; background:#f5f5f5; border-radius:10px;'>
        {result.replace("\n","<br>")}
    </div>
    """

run_button.on_click(on_button_click)


# ================================================
# 9. DISPLAY UI
# ================================================
display(widgets.VBox([title_html, input_box, run_button, output_area]))
