# LinkedIn Job Search Automation with n8n

This is an [n8n](https://n8n.io) workflow that looks for jobs on LinkedIn for you. It checks every job against your resume using AI, gives it a match score, writes a cover letter, saves everything in Google Sheets, and sends you a Telegram message when it finds a good match.

It started as a weekend project and turned into a fully automated job-matching machine.

## What it does

- Runs on a schedule (every day at 5 PM by default).
- Downloads your resume (PDF) from Google Drive.
- Reads your search filters from a Google Sheet.
- Searches LinkedIn and opens each job to get the title, company, location, and description.
- Sends the job description and your resume to an AI model.
- Gets back a match score (0 to 100) and a cover letter written for that job.
- Saves everything in a Google Sheet.
- Sends you a Telegram message if the score is 50 or higher.

## What you need

- **n8n.** Install it on your computer (recommended) or use the cloud version. Docker is the easiest way to run it locally, and [this YouTube search](https://www.youtube.com/results?search_query=n8n+docker+setup) has plenty of step-by-step guides.
- **Google Cloud credentials.** An OAuth2 client ID and secret, with the Google Drive API (to download your resume) and the Google Sheets API (to read your filters and save results) turned on.
- **An AI API key.** OpenAI, OpenRouter, or any compatible provider. If you need a free key, check out [OpenRouter](https://openrouter.ai).
- **A Telegram bot.** Create one with [@BotFather](https://t.me/BotFather). To get your user ID, message [@userinfobot](https://t.me/userinfobot).
- **A Google Sheet** with two tabs (see the next section).

If you want a quick way to start n8n with Docker, run this:

```bash
docker volume create n8n_data

docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -e GENERIC_TIMEZONE="Your/Timezone" \
  -e TZ="Your/Timezone" \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

Change `Your/Timezone` to your own (for example `Europe/Berlin`) so the 5 PM schedule runs at your local time, then open `http://localhost:5678`.

## Google Sheet setup

Make one spreadsheet with two tabs.

### Tab 1: Filter

This is where you say what kind of jobs you want. Name the tab `Filter` and use these exact column names:

| Keyword | Location | Experience Level | Remote | Job Type | Easy Apply |
|---------|----------|------------------|--------|----------|------------|
| Software Engineer | Berlin | Mid-Senior level | Remote | Full-time | true |

What goes in each column:

- **Keyword:** the job title or search term.
- **Location:** where to search.
- **Experience Level:** `Internship`, `Entry level`, `Associate`, `Mid-Senior level`, `Director`, or `Executive`. Use commas if you want more than one.
- **Remote:** `On-Site`, `Remote`, or `Hybrid`. Use commas if you want more than one.
- **Job Type:** `Full-time`, `Part-time`, `Contract`, `Temporary`, `Other`, or `Internship`.
- **Easy Apply:** `true`, or leave it empty.

### Tab 2: Results

Call it `Sheet1` (or anything you like) and add these headers in the first row:

`link`, `Title`, `Company`, `Location`, `score`, `description`, `Cover Letter`

The workflow adds or updates rows here. Keep the header names exactly as written.

## Setup

1. **Import the workflow.** Copy the JSON from this repo. In n8n, go to **Workflows → Import from Clipboard**, paste it, and save.

2. **Add your credentials** in n8n:
   - Google Drive OAuth2
   - Google Sheets OAuth2
   - OpenAI API (or your provider)
   - Telegram API

   Using OpenRouter? Create an OpenAI credential, paste your OpenRouter key, set the Base URL to `https://openrouter.ai/api/v1`, and pick your model in the **OpenAI Chat Model** node.

3. **Fill in the placeholders:**
   - **Download file:** pick your resume PDF from Google Drive.
   - **Get row(s) in sheet:** pick your spreadsheet and the `Filter` tab.
   - **Append or update row in sheet:** pick your spreadsheet and the results tab.
   - **Send a text message:** replace `TELEGRAM_CHAT_ID` with your chat ID.
   - **AI Agent:** change the prompt if you want. It gets your resume text from `$('Extract from File').item.json.text`.

4. **Test it.** Click **Execute Workflow**, then check your results sheet and Telegram. For the first run, use narrow filters so it doesn't go through too many jobs.

5. **Turn it on.** Switch the workflow to **Active** so it runs on schedule.

## What each node does

1. **Schedule Trigger:** starts the workflow (daily at 5 PM).
2. **Download file:** gets your resume PDF from Google Drive.
3. **Extract from File:** turns the PDF into plain text.
4. **Get row(s) in sheet:** reads your filters from the `Filter` tab.
5. **Create search URL:** builds the LinkedIn search link from your filters.
6. **Fetch Jobs from Linkedin:** loads the search results page.
7. **Extract Job Links:** pulls the job links out of that page.
8. **Split Out:** turns the list of links into separate items.
9. **Loop Over Items:** goes through the jobs one by one.
10. **Wait:** pauses 10 seconds between jobs to avoid rate limits.
11. **Fetch Job Page:** opens each job page.
12. **Parse Job Attributes:** grabs the title, company, location, description, and job ID.
13. **Modify Job Attributes:** cleans up the description, gets the job ID, and builds the apply link.
14. **AI Agent:** compares your resume with the job and returns a score and a cover letter.
15. **OpenAI Chat Model:** the AI model the agent uses (you can swap it for another one).
16. **Parse AI Output:** cleans up the AI's reply and turns it into JSON.
17. **Append or update row in sheet:** saves everything to your results sheet.
18. **Score Filter:** checks if the score is 50 or higher.
19. **Send a text message:** sends the Telegram alert.

After that, the workflow goes back to **Loop Over Items** for the next job. Both the "no" branch of Score Filter and the Telegram node loop back.

## Settings you can change

- **Schedule:** the **Schedule Trigger** node (default: daily at 5 PM).
- **Search filters:** the `Filter` tab in your sheet.
- **Wait time:** the **Wait** node (default: 10 seconds). Raise it if LinkedIn starts blocking requests.
- **AI model:** the **OpenAI Chat Model** node (default: `gpt-4.1-mini`). You can switch to OpenRouter or another compatible model.
- **Score cut-off:** the **Score Filter** node (default: 50).
- **AI prompt:** the **AI Agent** node.

## If something goes wrong

- **No jobs found, or the fields are empty.** LinkedIn might have changed its page layout or blocked the request. Check the output of the HTTP Request nodes, and update the CSS selectors in the HTML nodes if needed. Also make sure your filter values match the options listed above.
- **Errors like 429.** LinkedIn is rate limiting you. Increase the Wait time, use narrower filters, or run the workflow less often.
- **The AI output can't be parsed.** The model didn't reply with clean JSON. Try a better model and make sure the prompt asks for JSON only. The Parse AI Output node removes markdown fences, but it can't fix broken JSON.
- **Sheet columns are empty or in the wrong place.** Check that your header names match exactly.
- **Google login fails.** Make sure the Drive and Sheets APIs are enabled, your OAuth consent screen is set up, your account is added as a test user (if the app is in testing mode), and the redirect URL from n8n is added to your OAuth client.
- **No Telegram messages.** Send your bot a message first, double-check your chat ID, and make sure a job actually scored high enough.
- **It doesn't run on schedule.** Make sure the workflow is Active, n8n is running at that time, and your timezone is set correctly.

## Good to know

- Every job means one AI request, so more jobs means more cost.
- Your resume and the job descriptions go to whichever AI provider you use. Check their privacy policy, and keep your resume file private in Google Drive.
- Never upload your API keys or tokens to GitHub.

## Disclaimer

This is for personal and learning use. LinkedIn's terms may not allow automated access, so use it responsibly and keep the request rate low.

## Credits

Built by Nandu as a weekend experiment with [n8n](https://n8n.io). Feel free to fork it, change it, and make it better.
