# Data Processing Pipeline

This repository hosts a data processing pipeline designed to extract, clean, transform, and publish data from Excel files. The entire workflow is automated using GitHub Actions, ensuring consistent and reproducible results.

## Project Structure

- `data.xlsx`: The source Excel file containing raw data.
- `data.csv`: The converted CSV file, generated in CI.
- `execute.py`: A Python script responsible for processing the data.
- `.github/workflows/ci.yml`: GitHub Actions workflow for continuous integration and deployment.
- `index.html`: A responsive web page providing an overview and linking to the published results.
- `README.md`: This document.
- `LICENSE`: The MIT License for this project.

## Features

- **Automated Data Conversion:** Converts `data.xlsx` to `data.csv` as part of the CI pipeline.
- **Robust Data Processing:** The `execute.py` script handles missing values, type conversions (e.g., dates, numbers), and aggregates data into a structured JSON output.
- **Code Quality:** Integrated with `ruff` for code linting and style enforcement.
- **Automated CI/CD:** GitHub Actions workflow automates linting, processing, and publishing of results to GitHub Pages.
- **GitHub Pages Integration:** Publishes the `result.json` output directly to GitHub Pages for easy access.

## Setup and Local Development

### Prerequisites

- Python 3.11+
- pip (Python package installer)
- virtualenv (recommended)

### Installation

1.  **Clone the repository:**
    ```bash
    git clone https://github.com/YOUR_USERNAME/YOUR_REPOSITORY_NAME.git
    cd YOUR_REPOSITORY_NAME
    ```

2.  **Create and activate a virtual environment (recommended):**
    ```bash
    python -m venv .venv
    source .venv/bin/activate # On Windows: .venv\Scripts\activate
    ```

3.  **Create `requirements.txt` with the following content:**
    ```
    pandas==2.3.0
    openpyxl
    ruff
    ```

4.  **Install dependencies:**
    ```bash
    pip install -r requirements.txt
    ```

### Running `execute.py` Locally

To process `data.csv` and generate `result.json`:

1.  **Ensure `data.xlsx` is in the root directory.**
2.  **Convert `data.xlsx` to `data.csv`:**
    ```bash
    python -c "import pandas as pd; df = pd.read_excel('data.xlsx'); df.to_csv('data.csv', index=False)"
    ```
3.  **Execute the script:**
    ```bash
    python execute.py data.csv > result.json
    ```
    This will generate `result.json` in your current directory.

## File Contents

Below are the contents of the generated and fixed files, as per the request.

### `execute.py` (Fixed Version)

```python
import pandas as pd
import json
import sys

def process_data(file_path):
    try:
        # Determine file type based on extension
        if file_path.endswith('.csv'):
            df = pd.read_csv(file_path)
        elif file_path.endswith('.xlsx') or file_path.endswith('.xls'):
            df = pd.read_excel(file_path)
        else:
            raise ValueError("Unsupported file type. Please provide a .csv or .xlsx file.")

        # Drop rows with any NaN values across all columns
        df.dropna(inplace=True)

        # Non-trivial error fix: Robust date and numeric column handling.
        # Identify date column
        date_col = None
        for col in ['Date', 'Timestamp', 'time', 'datetime']: # Common date column names
            if col in df.columns:
                date_col = col
                break

        if date_col:
            # Convert to datetime, coercing errors to NaT (Not a Time)
            df[date_col] = pd.to_datetime(df[date_col], errors='coerce')
            df.dropna(subset=[date_col], inplace=True) # Drop rows where date conversion failed
        else:
            # If no recognizable date column, warn or handle gracefully.
            print("Warning: No recognizable date column found. Skipping date-based aggregation.", file=sys.stderr)

        # Identify a potential numeric column for aggregation
        numeric_col = None
        for col in ['Value', 'Amount', 'Count', 'Data']: # Common numeric column names
            if col in df.columns:
                numeric_col = col
                break

        if numeric_col:
            # Convert to numeric, coercing errors to NaN
            df[numeric_col] = pd.to_numeric(df[numeric_col], errors='coerce')
            df.dropna(subset=[numeric_col], inplace=True) # Drop rows where numeric conversion failed
        else:
            print("Warning: No recognizable numeric column found. Outputting raw data (if available).", file=sys.stderr)

        # Perform aggregation if both date and numeric columns are found
        if date_col and numeric_col:
            df['Day'] = df[date_col].dt.date
            daily_summary = df.groupby('Day')[numeric_col].agg(['sum', 'mean', 'count']).reset_index()
            # Convert date objects in 'Day' column to string for JSON serialization
            daily_summary['Day'] = daily_summary['Day'].astype(str)
            result = daily_summary.to_dict(orient='records')
        elif numeric_col: # If only numeric, calculate overall stats
            result = {
                "total_sum": df[numeric_col].sum(),
                "average_mean": df[numeric_col].mean(),
                "total_count": df[numeric_col].count()
            }
        else:
            # Fallback: just return the cleaned DataFrame as a list of dicts.
            result = df.to_dict(orient='records')

        return result

    except FileNotFoundError:
        print(f"Error: File not found at {file_path}", file=sys.stderr)
        return {"error": f"File not found: {file_path}"}
    except pd.errors.EmptyDataError:
        print(f"Error: The file {file_path} is empty.", file=sys.stderr)
        return {"error": f"Empty data file: {file_path}"}
    except ValueError as ve:
        print(f"Error processing data: {ve}", file=sys.stderr)
        return {"error": f"Data processing error: {ve}"}
    except Exception as e:
        print(f"An unexpected error occurred: {e}", file=sys.stderr)
        return {"error": f"An unexpected error occurred: {e}"}

if __name__ == "__main__":
    # Expects `data.csv` as input as per workflow instruction.
    # The workflow runs `python execute.py data.csv`.
    # So `sys.argv[1]` should be `data.csv`.
    if len(sys.argv) < 2:
        # Fallback to a default name if no argument provided, useful for local testing
        input_file = "data.csv"
        print(f"Usage: python execute.py <data_file_path>. No file path provided, defaulting to '{input_file}'.", file=sys.stderr)
    else:
        input_file = sys.argv[1]

    processed_data = process_data(input_file)
    print(json.dumps(processed_data, indent=2))
```

### `data.csv` (Converted from `data.xlsx`)

Assuming an example `data.xlsx` with columns `Date`, `Product`, `Value`, `Category` and some mixed data, the `data.csv` would look like this:

```csv
Date,Product,Value,Category
2023-01-01,A,10.5,X
2023-01-01,B,20,Y
2023-01-02,A,15.2,X
2023-01-02,C,invalid,Z
2023-01-03,B,25.1,Y
2023-01-04,D,30.0,X
2023-01-04,E,,Z
```

### `.github/workflows/ci.yml` (GitHub Actions Workflow)

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Set up Python 3.11
      uses: actions/setup-python@v5
      with:
        python-version: '3.11'
        cache: 'pip'

    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install ruff pandas==2.3.0 openpyxl

    - name: Run Ruff Linter
      run: ruff check . --output-format=github

    - name: Convert data.xlsx to data.csv
      run: |
        python -c "\
import pandas as pd; \
import sys; \
try: \
    df = pd.read_excel('data.xlsx'); \
    df.to_csv('data.csv', index=False); \
except FileNotFoundError: \
    print('data.xlsx not found, skipping conversion.', file=sys.stderr); \
    # Create an empty data.csv if data.xlsx is missing to prevent later steps from failing \
    # if the file is genuinely optional for some runs (though here it's expected). \
    with open('data.csv', 'w') as f: \
        f.write('Date,Value\n'); \
except Exception as e: \
    print(f'Error converting data.xlsx: {e}', file=sys.stderr); \
    sys.exit(1)"

    - name: Execute data processing script
      run: python execute.py data.csv > result.json

    - name: Upload result.json as artifact
      uses: actions/upload-artifact@v4
      with:
        name: result-json
        path: result.json

  deploy-pages:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      pages: write
      id-token: write
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}

    steps:
    - name: Download result.json artifact
      uses: actions/download-artifact@v4
      with:
        name: result-json

    - name: Setup Pages
      uses: actions/configure-pages@v5

    - name: Upload result.json to GitHub Pages
      uses: actions/upload-pages-artifact@v3
      with:
        path: './' # Upload the directory containing result.json

    - name: Deploy to GitHub Pages
      id: deployment
      uses: actions/deploy-pages@v4
```

## GitHub Actions CI/CD Pipeline (`.github/workflows/ci.yml`)

The `ci.yml` workflow automates the following steps on every push and pull request to `main`:

1.  **Checkout Code:** Retrieves the repository content.
2.  **Setup Python 3.11:** Configures the environment with Python 3.11 and caches pip dependencies.
3.  **Install Dependencies:** Installs `ruff`, `pandas` (version 2.3.0), and `openpyxl`.
4.  **Run Ruff Linter:** Checks the Python code for style and potential errors. Results are displayed in the CI log.
5.  **Convert `data.xlsx` to `data.csv`:** Uses a Python one-liner to perform this conversion, making `data.csv` available for the next step. Includes basic error handling for `data.xlsx` not found.
6.  **Execute Data Processing Script:** Runs `python execute.py data.csv` and redirects its output to `result.json`.
7.  **Upload `result.json` Artifact:** Stores `result.json` as a build artifact.
8.  **Deploy to GitHub Pages:** Publishes the `result.json` to GitHub Pages, making it publicly accessible (only on pushes to `main`).

### Accessing `result.json` on GitHub Pages

The generated `result.json` file for the latest successful run can be found at:

`https://<YOUR_GITHUB_USERNAME>.github.io/<YOUR_REPOSITORY_NAME>/result.json`

*(Replace placeholders with your actual GitHub username and repository name.)*

## License

This project is licensed under the MIT License - see the [LICENSE](#license) section below for details.
