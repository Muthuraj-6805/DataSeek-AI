# DataSeek AI

DataSeek AI is a Flask-based web application that turns natural-language or voice questions into read-only SQLite queries. Upload a single CSV file, let the application build a temporary local database and dynamic schema, then use the Groq API to generate SQL, execute it, and summarize the results.

## Features

- Upload and process a CSV file containing one table.
- Automatically create a local SQLite database from the uploaded data.
- Normalize column names while preserving their original order.
- Inspect the generated table schema and row count.
- Ask questions using natural language or browser-based voice input.
- Generate structured SQLite `SELECT` queries with Groq.
- Validate generated SQL and reject database-modifying operations.
- Display generated SQL, query metadata, result rows, and row counts.
- Generate a concise natural-language summary of the query result.
- Keep result-summary prompts limited to a maximum of ten result rows.
- Provide a health-check endpoint for server monitoring.

## How it works

1. The browser uploads a CSV file to `POST /process-file`.
2. `database.py` validates the file, cleans column names, and loads the data into `database.db` using SQLite and pandas.
3. The application extracts the table name, column types, column order, and row count.
4. The user enters or speaks a database requirement.
5. `agent.py` sends the requirement, dynamic schema, and up to three sample rows to Groq.
6. Groq returns structured metadata and one SQLite `SELECT` statement.
7. The SQL is parsed and validated before execution.
8. `database.py` executes the query and returns columns, rows, and the total row count.
9. Groq optionally creates a concise result summary from the executed query and at most ten result rows.

## Tech stack

- **Backend:** Python, Flask
- **Data processing:** pandas
- **Database:** SQLite via Python's built-in `sqlite3` module
- **AI provider:** Groq API
- **SQL parsing and formatting:** sqlparse
- **Frontend:** HTML, CSS, and browser JavaScript
- **Voice input:** Browser Web Speech API / `SpeechRecognition`
- **Deployment server:** Gunicorn

## Project structure

```text
.
├── agent.py          # Groq SQL-generation and result-summary logic
├── app.py            # Flask application and HTTP API routes
├── database.py       # CSV processing, SQLite setup, schema, and query execution
├── index.html        # Browser interface and client-side application logic
├── style.css         # Frontend styling
├── requirements.txt  # Python dependencies
├── database.db       # Generated local SQLite database after processing a CSV
└── uploads/          # Temporary upload directory created by the Flask app
```

`database.db` and `uploads/` are runtime artifacts. The database is replaced when a new CSV is processed, and successfully processed CSV files are removed from the upload directory.

## Prerequisites

- Python 3.9 or newer
- A Groq API key
- A modern browser with Web Speech API support if voice input is required
  - Google Chrome and Microsoft Edge generally provide the best support for browser speech recognition

## Setup

Clone the repository and enter its directory:

```bash
git clone https://github.com/Muthuraj-6805/DataSeek-AI.git
cd DataSeek-AI
```

Create and activate a virtual environment:

### macOS/Linux

```bash
python3 -m venv .venv
source .venv/bin/activate
```

### Windows PowerShell

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
```

Install the dependencies:

```bash
pip install -r requirements.txt
```

Configure the Groq API key. You can export it in your shell:

### macOS/Linux

```bash
export GROQ_API_KEY="your-groq-api-key"
```

### Windows PowerShell

```powershell
$env:GROQ_API_KEY = "your-groq-api-key"
```

Alternatively, create a `.env` file in the project root:

```dotenv
GROQ_API_KEY=your-groq-api-key
# Optional; defaults to openai/gpt-oss-20b
GROQ_MODEL=openai/gpt-oss-20b
```

Do not commit `.env` or real API keys to the repository.

## Run locally

Start the Flask development server:

```bash
python app.py
```

The application is available at:

- Web interface: http://127.0.0.1:5000
- Health check: http://127.0.0.1:5000/health

The health endpoint returns the server status and whether a database has been processed:

```bash
curl http://127.0.0.1:5000/health
```

## Run with Gunicorn

For a production-style WSGI server:

```bash
gunicorn app:app
```

By default, Gunicorn listens on `127.0.0.1:8000`. Configure the bind address and worker count as needed:

```bash
gunicorn --bind 0.0.0.0:8000 --workers 2 app:app
```

Make sure `GROQ_API_KEY` is available in the environment of the Gunicorn process.

## Usage

1. Open the web interface.
2. Select a `.csv` file containing one table.
3. Select **Process File**.
4. Review the detected table name, row count, and schema.
5. Type a natural-language question in the requirement box, or select **Speak** and use browser voice input.
6. Select **Generate SQL**.
7. Review the generated SQL, metadata, result table, row count, and natural-language summary.

Example questions:

- `Show the five highest salaries.`
- `How many records are in each department?`
- `List customers from California ordered by name.`

## API

### `GET /`

Serves the main `index.html` interface.

### `GET /health`

Returns the service status.

Example response:

```json
{
  "status": "running",
  "service": "DataSeek AI",
  "database_ready": false
}
```

### `POST /process-file`

Processes one CSV upload and creates the current SQLite database.

Request: `multipart/form-data` with a `file` field.

```bash
curl -X POST \
  -F "file=@data.csv" \
  http://127.0.0.1:5000/process-file
```

Important response fields include:

- `success`
- `filename`
- `table_name`
- `columns`
- `row_count`
- `schema`
- `schema_text`

### `GET /database-status`

Returns whether a CSV has been processed and, if so, the current table and schema.

### `POST /generate-sql`

Generates and executes a read-only SQL query for the current uploaded dataset.

Request body:

```json
{
  "requirement": "Show the five highest salaries"
}
```

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"requirement":"Show the five highest salaries"}' \
  http://127.0.0.1:5000/generate-sql
```

Important response fields include:

- `success`
- `requirement`
- `provider`
- `table_name`
- `sql`
- `explanation`
- `tables_used`
- `columns_used`
- `operations`
- `result_summary`
- `result.columns`
- `result.rows`
- `result.row_count`

## Query safety

DataSeek AI is designed to execute read-only queries:

- Generated SQL must contain exactly one statement.
- The statement must begin with `SELECT`.
- Database-modifying operations such as `INSERT`, `UPDATE`, `DELETE`, `DROP`, `ALTER`, `CREATE`, `REPLACE`, `ATTACH`, `DETACH`, and related transaction or maintenance commands are rejected.
- SQL is validated before it is sent to SQLite and formatted SQL is validated again.
- The result-summary model receives at most ten result rows; the complete query result remains available to the application for display.

These safeguards reduce risk, but the application should still be deployed with appropriate access controls, resource limits, and operational monitoring.

## Current limitations

- Only `.csv` uploads are supported.
- Each upload represents one table; joins across multiple uploaded tables are not supported.
- The active database is stored locally as `database.db` and represents the most recently processed CSV.
- Processing a new CSV replaces the previous local database.
- Successfully processed CSV files are removed from `uploads/` after the database is created.
- The generated query must be a single read-only SQLite `SELECT` statement.
- Voice input depends on browser support for the Web Speech API and microphone permissions.
- The Groq API key is required for SQL generation and AI-generated summaries.
- The application currently has no built-in authentication or multi-user database isolation.

## Troubleshooting

### `GROQ_API_KEY is not configured`

Set `GROQ_API_KEY` in the shell environment or in a local `.env` file, then restart the Flask or Gunicorn process.

### Unsupported file type

Upload a file whose name ends in `.csv`. Other formats are rejected by both the Flask route and `database.py`.

### Empty CSV or missing columns

The uploaded CSV must contain at least one row and one column. Empty files and files without columns are rejected during processing.

### Voice input is unavailable

Use a browser with Web Speech API support, allow microphone access for the site, and check that a microphone is available. If speech recognition is unavailable, type the requirement into the text box instead.

### Query generation or summary fails

Verify that the Groq API key is valid, the process has network access to Groq, and the selected model is available. SQL results may still be useful when the optional AI result-summary request fails because the application provides a basic row-count fallback.

## License

No license file is currently included in the repository.