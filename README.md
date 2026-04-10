# Email Phishing Detection

An advanced, asynchronous email phishing analyzer that combines email authentication verification, VirusTotal threat intelligence, OCR-based image analysis, and AI-powered scoring to deliver comprehensive phishing assessments.

---

## Features

- **Email Authentication Verification** -- Validates SPF, DKIM, and DMARC records by parsing authentication headers and performing live DNS lookups to confirm sender legitimacy.
- **VirusTotal Integration** -- Submits extracted URLs, IP addresses, and file hashes to the VirusTotal API for real-time threat intelligence, with results cached locally in SQLite to respect API rate limits.
- **OCR Image Analysis** -- Extracts text from image attachments (JPEG, PNG, TIFF, BMP, GIF) using Tesseract OCR to detect phishing content hidden inside screenshots or embedded graphics.
- **AI-Powered Verdict** -- Sends the full analysis payload to a configurable LLM endpoint (OpenRouter / DeepSeek by default) and receives a structured JSON verdict including phishing score, confidence level, suspicious elements, and actionable recommendations.
- **Typosquatting Detection** -- Uses Levenshtein distance calculations against a curated list of known brand domains to flag look-alike sender addresses and link domains.
- **Brand Impersonation Analysis** -- Identifies brand names mentioned in the email body and cross-references them with actual link destinations to surface domain mismatches.
- **Attachment Security Scanning** -- Generates MD5, SHA1, and SHA256 hashes for every attachment and checks them against VirusTotal; flags suspicious file extensions and double extensions.
- **Asynchronous Architecture** -- Built on `asyncio` and `aiohttp` for concurrent header, body, and attachment analysis, significantly reducing total analysis time.
- **Result Caching** -- Persists VirusTotal lookup results in an async SQLite database (`aiosqlite`) with configurable TTL to avoid redundant API calls across analysis runs.
- **Colored Console Reports** -- Renders detailed, color-coded terminal reports using `colorama` with box-drawn section headers, aligned key-value output, and configurable verbosity levels.
- **Structured JSON Export** -- Saves the complete analysis results as a JSON file for integration with downstream tooling, SIEM ingestion, or archival.
- **Multi-Format Support** -- Parses both `.eml` (RFC 5322) and `.msg` (Microsoft Outlook) email formats.

---

## Tech Stack

| Component | Technology |
|---|---|
| Language | Python 3.8+ |
| Async Runtime | `asyncio`, `aiohttp` |
| Email Parsing | `email` (stdlib), `extract-msg` |
| Threat Intelligence | VirusTotal API v3 |
| AI Analysis | OpenRouter API (DeepSeek Chat default) |
| OCR Engine | Tesseract OCR via `pytesseract`, `Pillow` |
| DNS Lookups | `dnspython` |
| String Similarity | `python-Levenshtein` |
| Database / Caching | SQLite via `aiosqlite` |
| Terminal Formatting | `colorama` |

---

## Architecture / How It Works

The analyzer follows a multi-stage pipeline orchestrated by `main.py`:

```
                          +------------------+
                          |   Email File     |
                          |  (.eml / .msg)   |
                          +--------+---------+
                                   |
                          +--------v---------+
                          |  Email Parser    |
                          |  (Validate &     |
                          |   Parse)         |
                          +--------+---------+
                                   |
                    +--------------+--------------+
                    |              |              |
           +-------v------+ +----v-------+ +----v-----------+
           | Header       | | Body       | | Attachment     |
           | Analyzer     | | Analyzer   | | Analyzer       |
           | - SPF/DKIM/  | | - URL      | | - File Hashes  |
           |   DMARC      | |   Extract  | | - OCR on       |
           | - IP Extract | | - Brand    | |   Images       |
           | - Typosquat  | |   Check    | | - Suspicious   |
           |   Check      | | - Typosquat| |   Extension    |
           +-------+------+ +----+-------+ +----+-----------+
                    |              |              |
                    +----+---------+------+-------+
                         |                |
                  +------v------+  +------v--------+
                  | VirusTotal  |  | SQLite Cache  |
                  | API Checks  |<>| (aiosqlite)   |
                  +------+------+  +---------------+
                         |
                  +------v------+
                  | AI Analysis | (optional)
                  | (OpenRouter)|
                  +------+------+
                         |
              +----------v-----------+
              |  Report Generator    |
              |  - Console (colored) |
              |  - JSON file export  |
              +----------------------+
```

1. **Parse** -- The email file is validated (size, extension) and parsed into a structured `EmailMessage` object. `.msg` files are converted via `extract-msg`.
2. **Analyze (concurrent)** -- Header, body, and attachment analyzers run concurrently using `asyncio.gather`. Each analyzer extracts indicators (IPs, URLs, hashes, text) and queries VirusTotal through the shared `aiohttp` session. The cache layer intercepts repeated lookups.
3. **AI Scoring (optional)** -- If enabled, the combined analysis payload is sent to the configured LLM which returns a structured JSON verdict with a 0-10 phishing score, confidence rating, and explanatory reasoning.
4. **Report** -- Results are rendered to the console with color-coded formatting and optionally saved as a JSON file.

---

## Project Structure

```
email-phishing-detection/
|-- main.py                    # CLI entry point and async orchestrator
|-- requirements.txt           # Python dependencies
|-- report.json                # Sample analysis output
|-- .gitignore
|
|-- config/
|   |-- __init__.py
|   |-- config.py              # Centralized configuration (API keys, timeouts,
|                              #   OCR settings, file limits, logging)
|
|-- src/
|   |-- __init__.py
|   |-- email_parser.py        # .eml/.msg parsing and validation
|   |-- security_analyzer.py   # Header, body, attachment analysis; VirusTotal
|   |                          #   client; typosquatting; OCR; hash generation
|   |-- ai_integration.py      # AI prompt construction and LLM API interaction
|   |-- report_generator.py    # Color-coded console report and formatting
|   |-- database_manager.py    # Async SQLite cache for VirusTotal results
|
|-- cache/                     # Auto-created directory for vt_cache.db
```

---

## Getting Started

### Prerequisites

- **Python 3.8+**
- **Tesseract OCR** (required only if OCR is enabled in config)
  - macOS: `brew install tesseract`
  - Ubuntu/Debian: `sudo apt install tesseract-ocr`
  - Windows: Download the installer from [UB Mannheim](https://github.com/UB-Mannheim/tesseract/wiki)
- **API Keys** (optional but recommended):
  - [VirusTotal API Key](https://www.virustotal.com/) -- free tier available
  - [OpenRouter API Key](https://openrouter.ai/) -- for AI-powered analysis

### Installation

```bash
# Clone the repository
git clone https://github.com/akshay1389/email-phishing-detection.git
cd email-phishing-detection

# Create and activate a virtual environment
python -m venv venv
source venv/bin/activate        # Linux / macOS
# venv\Scripts\activate         # Windows

# Install dependencies
pip install -r requirements.txt
```

### Configuration

Set API keys as environment variables before running:

```bash
export VIRUSTOTAL_API_KEY="your_virustotal_api_key"
export OPENROUTER_API_KEY="your_openrouter_api_key"
```

Additional settings (timeouts, cache duration, OCR languages, AI model, file size limits) can be adjusted directly in `config/config.py`.

If Tesseract is installed in a non-standard location, set the `TESSERACT_CMD` value in `config/config.py` to the full path of the `tesseract` executable.

### Running the Analyzer

```bash
python main.py -f /path/to/email.eml
```

---

## Usage Examples

**Basic analysis of an .eml file:**

```bash
python main.py -f suspicious_email.eml
```

**Enable AI-powered analysis:**

```bash
python main.py -f suspicious_email.eml --ai
```

**Save results to a JSON file:**

```bash
python main.py -f suspicious_email.eml --ai -o results.json
```

**Verbose console output (includes text snippets, full hashes, received chain):**

```bash
python main.py -f suspicious_email.eml --ai -o results.json -v
```

**Analyze a Microsoft Outlook .msg file:**

```bash
python main.py -f outlook_message.msg --ai -v
```

### Command-Line Options

| Flag | Description |
|---|---|
| `-f`, `--file` | **(Required)** Path to the email file (`.eml` or `.msg`) |
| `--ai` | Enable AI-based analysis (requires `OPENROUTER_API_KEY`) |
| `-o`, `--output` | Path to save full analysis results as JSON |
| `-v`, `--verbose` | Enable verbose report output with additional detail |

---

## Future Enhancements

- **Batch Processing** -- Support for analyzing multiple email files or entire mailbox directories in a single run.
- **IMAP/POP3 Integration** -- Connect directly to email servers to pull and analyze messages in real-time.
- **Web Dashboard** -- Browser-based interface for uploading emails, viewing reports, and tracking analysis history.
- **Custom Rule Engine** -- User-definable detection rules (YARA-style) for organization-specific phishing indicators.
- **Expanded Threat Intelligence** -- Integration with additional feeds such as AbuseIPDB, URLhaus, and PhishTank.
- **Machine Learning Model** -- Train a local classifier on labeled phishing datasets to reduce dependency on external AI APIs.
- **Internationalization** -- Extend OCR language support and body text analysis beyond English.
- **Docker Packaging** -- Provide a containerized deployment with Tesseract and all dependencies pre-installed.
- **Webhook / SOAR Integration** -- Push analysis results to Slack, Microsoft Teams, or SOAR platforms for automated incident response workflows.

---

## License

This project does not currently specify a license. Please contact the repository owner for usage terms.
