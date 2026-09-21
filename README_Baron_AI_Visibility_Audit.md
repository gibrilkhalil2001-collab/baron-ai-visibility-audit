# Baron AI — AI Visibility Audit

An n8n workflow that tests how a local business appears in answers from three AI providers, analyses the responses and emails a branded audit report.

## Why I built it

I built this for Baron AI Solutions LTD to answer a straightforward question: when a customer asks an AI tool for a local service, does the business appear in the answer?

A manual audit involves running questions across multiple tools, checking which businesses appear, recording the results and writing a report. That process is repetitive and difficult to compare if the questions change between audits.

This workflow brings those steps into a single pipeline. It gives me a consistent starting point for prospect research, client audits and follow-up assessments after work on a business’s online presence.

## What it fixes

The problem this project addresses is the audit process itself: repeated searches, scattered answers and manual reporting.

| Problem | Implementation |
| --- | --- |
| Running the same checks across different tools | A shared set of questions is submitted to Anthropic, Perplexity and OpenAI. |
| Different response formats | Provider-specific extraction nodes convert responses into a common structure. |
| Reading every answer to find business mentions | An analysis step extracts mentions, sentiment, list position and competitor names. |
| Calculating scores and compiling findings manually | JavaScript calculates mention rates and competitor counts. |
| Preparing each report from scratch | An HTML report is generated from the aggregated results and delivered through Gmail. |

It does not fix a business’s search visibility directly. It provides evidence to decide what to investigate and a baseline for later comparison. The export contains no measured business outcomes or time-saving benchmarks.

## How the workflow runs

1. **Intake:** an n8n form collects the business name, location, services, optional competitors and recipient email.
2. **Question generation:** a JavaScript node builds local service questions and direct questions about the business.
3. **Provider requests:** three branches query Claude, Perplexity Sonar and the OpenAI Responses API. The Claude and OpenAI requests include web-search tools.
4. **Response processing:** each branch extracts answer text. The merge node combines the answers by item position.
5. **Analysis:** Claude receives the three answers for each question and is instructed to return JSON with mentions, sentiment, recommendation position, competitors and a supporting quote.
6. **Aggregation:** JavaScript calculates per-provider mention rates and selects the eight most frequently mentioned competitor names.
7. **Reporting:** Claude generates an HTML report from the aggregated data. The final nodes extract the HTML and send it through Gmail.

## Design decisions

**Use the same questions across providers.** This keeps the comparison anchored to the same customer query. Provider behaviour still differs, so the results describe the sampled answers rather than a universal ranking.

**Separate language interpretation from arithmetic.** The model handles interpretation of the answers. JavaScript handles counts, percentages and sorting, keeping the scoring logic explicit and inspectable.

**Keep provider extraction separate.** Each API returns a different response structure. Dedicated extraction nodes keep those differences out of the aggregation code.

**Bound the workload.** The code accepts up to four services and creates three questions per service, one optional wider-area question and two brand questions. That gives a maximum of **15 questions** with the current templates. The code also contains an 18-question cap, which these templates do not reach.

**Batch requests.** The query and analysis nodes are configured for batches of two with a 1.5-second interval to moderate request volume. This is not a complete retry or recovery mechanism.

## Reading the score

Each provider’s score is calculated as:

`mention rate = questions marked as mentioning the business / total analysed questions × 100`

The result is rounded to a whole percentage. Six mentions across 15 questions gives a score of 40%.

A mention is not necessarily a recommendation. A negative answer can still count as a mention, and the two direct brand questions explicitly name the business. The current score combines these with the service-discovery questions. I would report those categories separately before using the score as a client performance metric.

## What the report includes

- Business details and audit date.
- A mention-rate score for each provider.
- Competitor names and mention counts.
- Question-by-question results and sentiment.
- A selected quote from the analysis.
- Suggested next steps for the business’s online presence.

The report is an HTML email. PDF export, scheduled monthly runs and historical reporting are not implemented in this version.

## Stack

| Component | Role |
| --- | --- |
| n8n | Intake, orchestration, API requests and delivery |
| JavaScript | Prompt generation, response transformation and scoring |
| Anthropic API | Search answers, response analysis and report generation |
| Perplexity API | Sonar answers |
| OpenAI Responses API | Answers from the node labelled “Ask ChatGPT” |
| Gmail OAuth2 | HTML email delivery |

The export specifies `claude-sonnet-4-6`, `sonar` and `gpt-4o`. These are configuration values from the file, not confirmation of current model availability or compatibility.

## Running it

1. Import the workflow JSON into n8n.
2. Configure Header Auth credentials for Anthropic (`x-api-key`), Perplexity (`Authorization: Bearer …`) and OpenAI (`Authorization: Bearer …`). Connect a Gmail OAuth2 account.
3. Assign the credentials to **Ask Claude**, **Ask Perplexity**, **Ask ChatGPT**, **Analyse Mentions**, **Generate HTML Report** and **Send Report**.
4. Replace the contact placeholder in **Generate HTML Report** with the correct business details.
5. Run a test using your own email address. Review the source answers, extracted analysis, calculated scores and rendered email.
6. Activate the workflow and use the intake form’s production URL.

API costs depend on request volume, response length and search usage. The cost estimate in the workflow notes has not been validated.

## Known limitations

- **Failed analysis affects the denominator.** If JSON parsing fails, the item contributes no mentions but remains in the total. Invalid results should be flagged and retried or excluded explicitly.
- **Output validation is limited.** The analysis prompt requests a JSON structure, but the aggregation code does not enforce a schema. Parsing successfully does not guarantee valid field types or accurate classifications.
- **Merging depends on item order.** Combining by position assumes the branches retain matching items in the same order. Joining on the existing question index would make that relationship explicit.
- **Evidence is reduced during processing.** The extraction nodes retain answer text, but do not separately preserve provider citation metadata for the final report. Storing full responses would make findings easier to review.
- **Report wording needs review.** The report prompt refers to recommendation frequency, while the calculation measures mentions. These should be aligned before client delivery.
- **API answers are a proxy.** They do not reproduce the exact experience of the consumer ChatGPT, Claude or Perplexity apps.
- **All three providers are assumed downstream.** Removing a branch also requires changes to analysis, aggregation and reporting; changing the merge input count alone is insufficient.

The next engineering priorities are validated analysis output, explicit failure handling, separate discovery and brand scores, and persistent audit records. These would make repeat comparisons more defensible and failures easier to diagnose.

## Project status

This README describes the supplied workflow configuration. The workflow has not been executed as part of this review; live API behaviour, email delivery and production reliability remain unverified.
