# n8n Job Hunter 🤖

An n8n workflow that finds relevant job offers every day and adapts your CV to each one,
so you only have to review and click "Apply".

## How it works

1. Reads your CV from Google Docs; Gemini extracts your role, level, country and search keywords.
2. Searches this week's job offers with the JSearch API (Google for Jobs: LinkedIn, Indeed, local sites).
3. Filters out irrelevant, too senior, internship and already-sent offers.
4. Gemini scores each offer out of 100 against your CV.
5. For the best offers, creates a copy of your CV adapted to the offer (reordered, never invented).
6. Adds each offer and its adapted CV to a Google Sheet and emails you the links.

It never applies for you: you review the CV and apply yourself.

## Tech stack

- n8n (self-hosted in a Docker container)
- Google Gemini (CV analysis, scoring, CV adaptation)
- JSearch API (job search)
- Google Docs, Drive and Sheets
- Gmail

## The problem

Adapting my CV manually to every job offer took too much time.
I wanted to spend my energy on applying and preparing interviews, not on copy-pasting.

## Design choices

- **Human in the loop.** The workflow never applies for me. AI can make mistakes,
  so I always review the adapted CV before applying.
- **API instead of scraping.** JSearch gives a stable, documented contract.
  Scraping breaks whenever a website changes its HTML and is hard to control.
- **Catch-up on startup instead of a cloud server.** If my PC is off at 14:00,
  the run is skipped. Since I am the only user, I chose a catch-up: when n8n starts,
  it checks whether today's run happened and runs it if not (once per day).
  Next step: move it to an Oracle Cloud Always Free VM so it runs 24/7.

## Results

What used to take me half a day now takes a few minutes: I open the email,
review the adapted CVs and click "Apply".

## What I learned

- Packaging the workflow in Docker so anyone can run it with one command,
  with the same result on every machine.
- Making it run automatically without failing: waiting for the network on startup,
  avoiding duplicate runs and staying within the free API quotas.


## Setup

See [docs-installation.md](docs-installation.md) (French).
