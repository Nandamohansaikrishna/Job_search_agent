# LinkedIn Job Search Automation with n8n

Automatically find the LinkedIn jobs that fit your profile. This [n8n](https://n8n.io) workflow searches LinkedIn using your filters, scores every listing against your resume with AI, writes a tailored cover letter, logs the results to Google Sheets, and sends a Telegram alert whenever a strong match appears.

What started as a weekend project is now a fully automated job-matching pipeline.
## Overview

Job hunting involves a lot of repetition: run a search, open each listing, decide whether it fits, then tailor an application. This workflow automates that loop from start to finish.

On a schedule, it reads your search criteria from a Google Sheet, builds a LinkedIn search, and processes each result one at a time. For every job, it extracts the key details, compares the description against your resume using an AI model, and produces a match score (0–100) together with a custom cover letter. All results are saved to Google Sheets, and any job that meets your score threshold triggers an instant Telegram notification.

## Features

- **Scheduled execution** — runs daily at 5 PM by default; the schedule is fully adjustable.
- **Resume-aware matching** — downloads your resume (PDF) from Google Drive and converts it to text for analysis.
- **Sheet-driven filters** — reads keyword, location, experience level, remote type, job type, and Easy Apply preferences from a Google Sheet, so you can change your search without editing the workflow.
- **Automated job collection** — builds a LinkedIn search URL, scrapes the listings, and opens each job page to extract the title, company, location, and description.
- **AI match scoring** — sends the job description and your resume to a chat model that returns a match score from 0 to 100.
- **Custom cover letters** — generates a tailored cover letter for every job it evaluates.
- **Central tracking** — saves every result to a Google Sheet for easy sorting and review.
- **Instant Telegram alerts** — notifies you when a job meets your score threshold (default: 50).
- **Model-agnostic** — works with OpenAI, OpenRouter, or any compatible chat model provider.

## How It Works

```text
┌───────────────────┐   ┌──────────────────────┐   ┌─────────────────────────┐
│ 1. Trigger        │──►│ 2. Prepare           │──►│ 3. Search LinkedIn      │
│ Daily at 5 PM     │   │ Resume + filters     │   │ Build URL, get links    │
└───────────────────┘   └──────────────────────┘   └────────────┬────────────┘
                                                                │
                                                                ▼
┌───────────────────┐   ┌──────────────────────┐   ┌─────────────────────────┐
│ 6. Save + Alert   │◄──│ 5. AI analysis       │◄──│ 4. Process each job     │
│ Sheets, Telegram  │   │ Score + cover letter │   │ Fetch, parse, clean     │
└───────────────────┘   └──────────────────────┘   └─────────────────────────┘
```

Steps 4 to 6 repeat for every job found.

1. **Trigger** — The Schedule Trigger starts the run (default: daily at 5 PM).
2. **Prepare** — Your resume PDF is downloaded from Google Drive and converted to text, and your search filters are read from the `Filter` tab.
3. **Search LinkedIn** — A LinkedIn search URL is built from your filters, the results page is fetched, and the job links are extracted.
4. **Process each job** — Jobs are handled one at a time with a 10-second pause between requests. Each job page is fetched and parsed for its title, company, location, description, and job ID.
5. **AI analysis** — The AI agent compares the job description with your resume and returns a match score (0–100) plus a tailored cover letter.
6. **Save and alert** — The result is written to Google Sheets. If the score meets the threshold (default: 50), you receive a Telegram message. The workflow then moves on to the next job.

## Tech Stack

| Component | Role |
|-----------|------|
| [n8n](https://n8n.io) | Open-source workflow automation platform that runs the pipeline |
| LinkedIn | Source of job listings (search and job pages, fetched over HTTP) |
| Google Drive | Stores your resume PDF |
| Google Sheets | Holds your search filters and the results log |
| OpenAI, OpenRouter, or a compatible provider | Chat model used for scoring and cover letter generation |
| Telegram Bot API | Delivers high-match notifications |

## Prerequisites

Make sure you have the following in place before you begin.

### n8n

Install n8n locally (recommended) or use the cloud version. Running it locally with Docker is the easiest route. For step-by-step help, see the [n8n hosting documentation](https://docs.n8n.io/hosting/) or search YouTube for an [n8n Docker setup guide](https://www.youtube.com/results?search_query=n8n+docker+setup).

Quick start with Docker:

```bash
docker volume create n8n_data

docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -e GENERIC_TIMEZONE="<YOUR_TIMEZONE>" \
  -e TZ="<YOUR_TIMEZONE>" \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

Replace `<YOUR_TIMEZONE>` with your own timezone (for example, `Europe/Berlin`) so the schedule fires at the correct local time, then open `http://localhost:5678` in your browser.

### Google Cloud credentials

You need OAuth2 credentials (client ID and client secret) from the [Google Cloud Console](https://console.cloud.google.com) with access to:

- **Google Drive API** — to download your resume.
- **Google Sheets API** — to read your filters and write the results.

### Chat model API key

An API key for OpenAI, [OpenRouter](https://openrouter.ai), or any compatible provider. If you need a free API key, OpenRouter is a good place to start.

### Telegram bot

1. Create a bot with [@BotFather](https://t.me/BotFather) and copy the bot token.
2. Message [@userinfobot](https://t.me/userinfobot) to get your user ID, which you will use as the chat ID.

### Google spreadsheet

One spreadsheet with two tabs: `Filter` for your search criteria and `Sheet1` (or any name) for the results. The exact layout is described below.

## Google Sheets Setup

Create a single spreadsheet containing two tabs.

### Filter tab

Name the tab `Filter` and use these column headers exactly as written:

| Keyword | Location | Experience Level | Remote | Job Type | Easy Apply |
|---------|----------|------------------|--------|----------|------------|
| Software Engineer | Berlin | Mid-Senior level | Remote | Full-time | true |

| Column | Description | Accepted values |
|--------|-------------|-----------------|
| `Keyword` | Job title or search term | Any text, e.g. `Software Engineer` |
| `Location` | Where to search | Any location, e.g. `Berlin` |
| `Experience Level` | Seniority filter | Comma-separated list of `Internship`, `Entry level`, `Associate`, `Mid-Senior level`, `Director`, `Executive` |
| `Remote` | Work arrangement | Comma-separated list of `On-Site`, `Remote`, `Hybrid` |
| `Job Type` | Employment type | `Full-time`, `Part-time`, `Contract`, `Temporary`, `Other`, `Internship` |
| `Easy Apply` | Limit results to LinkedIn Easy Apply jobs | `true`, or leave empty |

### Results tab

Name the tab `Sheet1` (or any name you prefer) and add these headers. The workflow appends or updates rows here.

| Column | Contents |
|--------|----------|
| `link` | Link to the job posting |
| `Title` | Job title |
| `Company` | Hiring company |
| `Location` | Job location |
| `score` | AI match score (0–100) |
| `description` | Cleaned job description |
| `Cover Letter` | AI-generated cover letter tailored to the job |

> **Note:** Keep the header names exactly as shown so the Google Sheets nodes map each column correctly.

## Workflow Reference

Importing the provided JSON gives you every node, connection, and setting automatically. Here is what each node does:

| # | Node | Type | Purpose |
|---|------|------|---------|
| 1 | Schedule Trigger | Trigger | Runs the workflow on a schedule (default: daily at 5 PM). |
| 2 | Download file | Google Drive | Downloads your resume PDF. |
| 3 | Extract from File | Extract from File | Converts the PDF to plain text. |
| 4 | Get row(s) in sheet | Google Sheets | Reads your search filters from the `Filter` tab. |
| 5 | Create search URL | Code | Builds a LinkedIn search URL from the filters. |
| 6 | Fetch Jobs from Linkedin | HTTP Request | Fetches the search results page. |
| 7 | Extract Job Links | HTML | Scrapes job links using CSS selectors. |
| 8 | Split Out | Split Out | Splits the array of links into individual items. |
| 9 | Loop Over Items | Split In Batches | Processes each job one by one. |
| 10 | Wait | Wait | Adds a 10-second delay to avoid rate limiting. |
| 11 | Fetch Job Page | HTTP Request | Downloads the individual job page. |
| 12 | Parse Job Attributes | HTML | Extracts title, company, location, description, and job ID. |
| 13 | Modify Job Attributes | Set | Cleans the description, extracts the job ID, and builds the apply link. |
| 14 | AI Agent | AI Agent | Compares your resume with the job description and returns a score and cover letter. |
| 15 | OpenAI Chat Model | Chat model | Provides the language model (swap in any compatible model). |
| 16 | Parse AI Output | Set | Cleans the AI response and converts it to JSON. |
| 17 | Append or update row in sheet | Google Sheets | Saves everything to your results sheet. |
| 18 | Score Filter | IF | Checks whether the score is ≥ 50. |
| 19 | Send a text message | Telegram | Sends a notification for high-scoring jobs. |

**Loop-back:** the false branch of **Score Filter** and the **Telegram** node both connect back to **Loop Over Items**, so the workflow continues with the next job.

## Installation

### 1. Import the workflow

1. Copy the workflow JSON from this repository.
2. In n8n, go to **Workflows → Import from Clipboard**.
3. Paste the JSON and save.

### 2. Configure credentials

Create these credentials in n8n and attach them to the matching nodes:

| Credential | Used by | Notes |
|------------|---------|-------|
| Google Drive OAuth2 | Download file | OAuth2 client ID and secret with the Google Drive API enabled |
| Google Sheets OAuth2 | Get row(s) in sheet, Append or update row in sheet | OAuth2 client ID and secret with the Google Sheets API enabled |
| OpenAI API (or compatible provider) | OpenAI Chat Model | API key from OpenAI, OpenRouter, or another compatible provider |
| Telegram API | Send a text message | Bot token from @BotFather |

> **Using OpenRouter?** Create an OpenAI credential in n8n, enter your OpenRouter API key, and set the Base URL to `https://openrouter.ai/api/v1`. Then choose your preferred model in the **OpenAI Chat Model** node.

### 3. Update placeholders

| Node | What to change |
|------|----------------|
| **Download file** | Select your resume PDF from Google Drive. |
| **Get row(s) in sheet** | Select your spreadsheet and the `Filter` tab. |
| **Append or update row in sheet** | Select your spreadsheet and the results tab. |
| **Send a text message** | Replace `TELEGRAM_CHAT_ID` with your actual chat ID. |
| **AI Agent** | Adjust the prompt if needed. It reads your resume text via `$('Extract from File').item.json.text`. |

### 4. Test the workflow

1. Click **Execute Workflow** to run it manually.
2. Check that new rows appear in your results sheet.
3. Confirm that you receive a Telegram message for any job that meets the score threshold.

> **Tip:** For your first test, use narrow filters so fewer jobs are processed. Each job adds a delay and one AI request.

### 5. Activate

Toggle the workflow to **Active** so it runs on its schedule.

## Configuration

| Setting | Where to change it | Default |
|---------|--------------------|---------|
| Run schedule | **Schedule Trigger** node | Daily at 5 PM |
| Search criteria | `Filter` tab in Google Sheets | Your own values |
| Delay between requests | **Wait** node | 10 seconds |
| AI model or provider | **OpenAI Chat Model** node | OpenAI `gpt-4.1-mini` |
| Matching and cover letter prompt | **AI Agent** node | Uses the extracted resume text |
| Notification threshold | **Score Filter** node | 50 |
| Telegram recipient | **Send a text message** node | `TELEGRAM_CHAT_ID` placeholder |

## Usage

Once the workflow is active, it runs on its own:

1. **Change what you search for** by editing the `Filter` tab. No workflow edits are needed.
2. **Review new jobs** in the results sheet. Sort by the `score` column to see the strongest matches first.
3. **Use the cover letters** in the `Cover Letter` column as a starting point, and review and personalize them before sending.
4. **Watch Telegram** for alerts on jobs that meet your threshold.

To run it on demand, open the workflow in n8n and click **Execute Workflow**.

## Troubleshooting

| Symptom | Likely cause | What to try |
|---------|--------------|-------------|
| No jobs found, or job fields are empty | LinkedIn changed its page markup, blocked the request, or a filter value is invalid | Inspect the output of the HTTP Request nodes, update the CSS selectors in the HTML nodes if needed, and check that your `Filter` values match the accepted options |
| HTTP errors or blocked requests (for example, 429) | LinkedIn rate limiting | Increase the **Wait** delay, narrow your filters, or run the workflow less often |
| AI output fails to parse | The model returned invalid JSON or extra text | Use a model that follows instructions reliably and keep the prompt strict about JSON-only output. **Parse AI Output** strips markdown fences but cannot repair malformed JSON |
| Sheet columns are empty or misaligned | Header names in the sheet do not match | Use the exact column names from [Google Sheets Setup](#google-sheets-setup) |
| Google authorization fails | APIs not enabled or OAuth not fully configured | Enable the Drive and Sheets APIs, configure the OAuth consent screen, add your account as a test user if the app is in testing mode, and add the redirect URL shown by n8n to your OAuth client |
| No Telegram notifications | Bot not started, wrong chat ID, or no job reached the threshold | Send your bot a message first, confirm your chat ID with @userinfobot, and check the score threshold |
| Workflow does not run on schedule | Workflow is inactive, n8n is not running, or the timezone is off | Toggle the workflow to **Active**, keep your n8n instance running, and check your timezone settings |

## Notes and Limitations

- The workflow uses OpenAI `gpt-4.1-mini` by default. You can change the model, or use OpenRouter, by editing the **OpenAI Chat Model** node.
- LinkedIn may rate-limit requests. The 10-second wait helps, and you can increase it if needed.
- The AI prompt expects a JSON response. The **Parse AI Output** node strips markdown fences before converting the response to JSON.
- The score threshold is set to 50. Change it in the **Score Filter** node.
- Scraping depends on LinkedIn's current page structure, so the CSS selectors in the HTML nodes may need updating over time.
- Each processed job triggers one AI request, so usage costs grow with the number of jobs found.
- Your resume text and the job descriptions are sent to the AI provider you configure. Review that provider's data-handling policy, keep your resume file private in Google Drive, and never commit API keys or tokens to version control.

## Disclaimer

This project is intended for personal and educational use. Automated access to LinkedIn may be restricted by its Terms of Service, and LinkedIn can block or rate-limit automated requests. You are responsible for how you use this workflow. Keep request rates modest and comply with the terms and laws that apply to you.

## Contributing

Contributions are welcome. Feel free to fork, modify, and improve the workflow.

1. Fork the repository.
2. Create a feature branch.
3. Commit your changes with a clear message.
4. Open a pull request describing what you changed and why.

## Credits

Built by Nandu as a weekend experiment with [n8n](https://n8n.io), the open-source workflow automation tool.