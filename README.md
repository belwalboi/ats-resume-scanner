# ATS Resume Scanner

An NLP-powered ATS Resume Scanner that compares resumes against job descriptions, identifies matching and missing skills, calculates an explainable compatibility score, and generates actionable resume improvement suggestions.

Built with Python, FastAPI, Streamlit, and NLP techniques.

## Features

- **Resume Parsing:** Extract text from PDF and DOCX resumes.
- **Skill Extraction:** Identify relevant technical skills using a curated skill taxonomy.
- **Keyword Matching:** Detect exact, related, and missing skills.
- **Semantic Similarity:** Compare resume content with job descriptions using TF-IDF and cosine similarity.
- **Explainable Scoring:** Calculate a compatibility score using skill coverage, text similarity, and contextual evidence.
- **Evidence-Based Analysis:** Display resume snippets supporting identified skill matches.
- **Improvement Recommendations:** Suggest truthful, actionable ways to improve resume alignment.
- **Input Validation:** Handle unsupported files, malformed documents, and oversized uploads.
- **REST API:** Expose resume analysis through FastAPI.
- **Interactive Dashboard:** Present analysis results through Streamlit.

## Technology Stack

| Component | Technology |
|---|---|
| Programming Language | Python |
| Frontend | Streamlit |
| Backend | FastAPI |
| NLP | scikit-learn |
| Text Similarity | TF-IDF and Cosine Similarity |
| Document Processing | PyMuPDF, python-docx |
| Database | SQLite, where persistence is required |
| API Testing | HTTPX |
| Automated Testing | Pytest |
| Deployment | Streamlit Community Cloud or another compatible hosting platform |

## Architecture


 # Resume Analysis Architecture

This system processes a user's resume against a specific job description to generate compatibility scores and actionable recommendations. The architecture is divided into four main layers:

## Architecture Diagram

```mermaid
flowchart TD
    User((User)) --> UI
    
    subgraph Client Layer
        UI[Streamlit UI]
    end
    
    subgraph API Layer
        API[FastAPI Server]
    end
    
    subgraph Processing Engine
        Parser[Resume Parser]
        NLP[NLP Analysis]
        Scoring[Scoring Engine]
    end
    
    subgraph Output
        Dash[Results Dashboard]
    end

    UI -- HTTP POST --> API
    API --> Parser
    Parser --> NLP
    NLP --> Scoring
    Scoring --> Dash


The backend handles all resume analysis and scoring. The frontend communicates with the backend through HTTP requests.

## How It Works

1. Upload a resume in PDF or DOCX format.
2. Enter the target job description.
3. Extract and preprocess resume text.
4. Identify relevant skills and job requirements.
5. Compare resume skills with the job description.
6. Calculate skill coverage and text similarity.
7. Generate a compatibility score and explain the results.
8. Display matched skills, missing skills, evidence, and recommendations.

## Scoring Methodology

The scanner calculates an overall compatibility score using a weighted combination of available components:

- **Skill Coverage:** Measures how well the resume covers relevant job requirements.
- **Text Similarity:** Measures textual similarity between the resume and job description.
- **Contextual Evidence:** Evaluates whether matched skills appear in meaningful resume content rather than only in a skills list.

Exact matches receive full credit, related skills receive partial credit, and missing skills receive no credit for that requirement.

The component weights and matching rules are configurable.

**Important:** The resulting score is a heuristic compatibility estimate. It is not an official ATS score, a hiring probability, or a validated prediction of recruitment outcomes.

## API Endpoints

| Method | Endpoint | Description |
|---|---|---|
| GET | `/health` | Check backend health |
| POST | `/analyze` | Analyze a resume against a job description |
| GET | `/docs` | Access interactive API documentation |

### Example API Request

```bash
curl -X POST http://127.0.0.1:8000/analyze \
  -F "file=@resume.pdf" \
  -F "job_description=Python developer with SQL and NLP experience"
```

The analysis endpoint returns structured results containing the compatibility score, matched skills, related skills, missing skills, recommendations, and relevant warnings.

## Installation and Setup

### Prerequisites

- Python 3.10 or later
- Git
- pip

### 1. Clone the Repository

```bash
git clone <your-repository-url>
cd ats-resume-scanner
```

Replace `<your-repository-url>` with your actual GitHub repository URL.

### 2. Create a Virtual Environment

Linux/macOS:

```bash
python -m venv .venv
source .venv/bin/activate
```

Windows:

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
```

### 3. Install Dependencies

```bash
pip install -r requirements-dev.txt
```

### 4. Run the Application

```bash
streamlit run app/frontend.py
```

If the application is configured for single-deployment mode, the frontend starts the local FastAPI backend automatically.

Alternatively, start the backend separately:

```bash
uvicorn app.main:app --host 127.0.0.1 --port 8000
```

Then open the API documentation at:

http://127.0.0.1:8000/docs

## Testing

Run the automated test suite:

```bash
python -m pytest tests -q
```

The tests should cover document parsing, input validation, skill matching, scoring behavior, API responses, and error handling.

## Deployment

The application can be deployed using a compatible Python hosting platform.

### Streamlit Community Cloud

1. Push the project to GitHub.
2. Open [Streamlit Community Cloud](https://share.streamlit.io/).
3. Create a new application.
4. Select the repository and branch.
5. Set the main file to `app/frontend.py`.
6. Configure the supported Python version.
7. Deploy and test the public application.

Before deployment, verify that the hosting environment supports the application's backend startup process and resource requirements.

Deployment is not considered complete until the public application has been tested successfully.

## Privacy and Security

- Validate uploaded documents before processing.
- Avoid logging resume contents.
- Minimize retention of uploaded files.
- Keep credentials out of source control.
- Handle malformed documents safely.
- Consider authentication and rate limiting before exposing the application to substantial public traffic.

Resumes contain personal information. Users should avoid uploading documents they are not comfortable processing on the selected hosting platform.

## Limitations

- Skill detection depends on the available skill taxonomy.
- Related-skill matching may award partial credit to skills that are not fully interchangeable.
- Scanned PDFs require OCR, which may not be supported.
- Complex document layouts can affect text extraction.
- Keyword matching may miss context, negation, or proficiency levels.
- Text similarity does not establish actual candidate suitability.
- The scoring methodology has not been calibrated against real hiring outcomes.

## Future Improvements

- OCR support for scanned resumes.
- Improved contextual skill extraction.
- More robust detection of experience and proficiency.
- Expanded skill taxonomy.
- Resume comparison across multiple job descriptions.
- Downloadable analysis reports.
- Performance and scoring evaluation using a labeled test dataset.
- Authentication and rate limiting for public deployment.

## Disclaimer

This project is intended for educational purposes and resume self-assessment. Its scores and recommendations should not be treated as definitive measures of candidate quality or hiring suitability.
