# AI Resume Screening Automation

**Automate resume evaluation using n8n, LLMs, and Google APIs**

## The Problem

When you're hiring, reviewing resumes manually is slow and inconsistent. You need to:
- Quickly assess if skills match the job
- Identify what's missing
- Score candidates fairly

Doing this for 50+ resumes takes hours. This project automates it.

---

## What This Does

1. **Watches** a Google Drive folder for new resume PDFs
2. **Extracts** text from the resume
3. **Sends** resume + job description to an LLM
4. **Gets back** structured scoring (skills match %, gaps, relevance)
5. **Saves** results to Google Sheets for review

**Result:** Upload resume → get scored automatically.

---

## How It Works

### Architecture

```
┌─────────────────┐
│  Google Drive   │
│  /Resumes/      │ ← Upload PDF
└────────┬────────┘
         │ (New file detected)
         ▼
┌────────────────────────┐
│   n8n Workflow         │
│                        │
│  1. Download Resume    │
│  2. Extract Text       │
│  3. Call LLM API       │
│  4. Parse JSON Output  │
│  5. Validate Schema    │
└───────────┬────────────┘
            │
            ▼
┌─────────────────┐
│  Google Sheets  │ ← Scored data
│  Results        │
└─────────────────┘
```

### Step-by-Step Flow

**Step 1: Google Drive Trigger**
- Monitors specific folder every minute
- Triggers when new PDF is uploaded

**Step 2: Download & Extract**
- Downloads the PDF file
- Uses n8n's Extract node to get text content

**Step 3: LLM Scoring**
- Sends prompt with:
  - Resume text
  - Job description
  - Scoring instructions
- Gets structured JSON response

**Step 4: Parse & Validate**
- Checks JSON format is correct
- Extracts key fields
- Handles errors if LLM returns bad data

**Step 5: Save to Sheets**
- Appends row with:
  - Candidate name
  - Skills match %
  - Missing skills
  - Recommendation

---

## The LLM Prompt

Here's what I send to the LLM:

```
You are a resume evaluator. Score this resume against the job description.

JOB DESCRIPTION:
{job_description}

RESUME:
{resume_text}

Return ONLY valid JSON in this format:
{
  "candidate_name": "string",
  "skills_match_percentage": number (0-100),
  "matched_skills": ["skill1", "skill2"],
  "missing_skills": ["skill1", "skill2"],
  "relevance_score": number (0-10),
  "key_strengths": "brief summary",
  "concerns": "brief summary",
  "recommendation": "Highly Recommended | Recommended | Consider | Reject"
}
```

**Why JSON schema matters:**  
Without it, LLMs return inconsistent formats. Sometimes paragraphs, sometimes broken JSON. Schema validation ensures every response is usable.

---

## Tech Stack

| Component | Technology | Why |
|-----------|-----------|-----|
| Automation | n8n | Visual workflow builder, easy to debug |
| LLM | Groq (Llama) | Fast, supports structured outputs |
| PDF Extraction | n8n Extract node | Built-in, works well |
| Storage | Google Sheets | Easy to review, no database needed |
| Trigger | Google Drive API | Automatic file monitoring |

---

## Setup Guide

### Prerequisites

- n8n (self-hosted or cloud)
- Google Cloud project
- Groq API key (free tier works)

### Step 1: Google Cloud Setup

1. Go to [Google Cloud Console](https://console.cloud.google.com)
2. Create project
3. Enable these APIs:
   - Google Drive API
   - Google Sheets API
4. Create OAuth 2.0 credentials
5. Download JSON credentials

### Step 2: n8n Setup

**Install n8n** (if you don't have it):
```bash
npm install -g n8n
n8n start
```

**Import workflow:**
1. Open n8n at `http://localhost:5678`
2. Click "Import from File"
3. Upload `n8n-resume-scoring.json`

**Add credentials:**
1. Go to Credentials tab
2. Add "Google Drive OAuth2"
3. Add "Google Sheets OAuth2"
4. Authenticate with your Google account

**Add Groq API:**
1. Get API key from [Groq Console](https://console.groq.com)
2. Add as HTTP Header Auth in n8n
3. Header: `Authorization`
4. Value: `Bearer YOUR_API_KEY`

### Step 3: Configure Folders

**Google Drive:**
1. Create folder "Resumes_uploaded"
2. Copy folder ID from URL:
   ```
   https://drive.google.com/drive/folders/FOLDER_ID_HERE
   ```
3. Paste in n8n trigger node

**Google Sheets:**
1. Create new sheet "Resume Scores"
2. Add headers: `Name, Skills Match %, Matched Skills, Missing Skills, Score, Recommendation, Date`
3. Copy sheet ID from URL
4. Paste in n8n output node

### Step 4: Customize Job Description

In the LLM prompt node, update:

```javascript
const jobDescription = `
Role: Data Analyst
Required Skills: SQL, Python, Power BI
Experience: 2+ years
...
`;
```

### Step 5: Test

1. Upload test resume to Drive folder
2. Check n8n execution log
3. Verify output in Google Sheets

If it fails, check:
- API credentials are valid
- Folder IDs are correct
- Groq API key works

---

## Example Output

| Name | Skills Match | Matched | Missing | Score | Recommendation |
|------|--------------|---------|---------|-------|----------------|
| John Doe | 75% | SQL, Python | Power BI, AWS | 7/10 | Recommended |
| Jane Smith | 90% | SQL, Python, Power BI | None | 9/10 | Highly Recommended |

---

## Workflow JSON Structure

The `n8n-resume-scoring.json` file contains:

```json
{
  "nodes": [
    {
      "type": "googleDriveTrigger",
      "name": "Watch Resumes Folder"
    },
    {
      "type": "googleDrive", 
      "name": "Download PDF"
    },
    {
      "type": "extractFromFile",
      "name": "Extract Text"
    },
    {
      "type": "httpRequest",
      "name": "Score with LLM"
    },
    {
      "type": "googleSheets",
      "name": "Save Results"
    }
  ]
}
```

Each node has:
- **Parameters**: Configuration (folder IDs, API endpoints)
- **Credentials**: Auth tokens
- **Position**: Visual layout coordinates

---

## What I Learned

**API Integration:**
- How to authenticate with OAuth 2.0
- Handling async API responses
- Error handling for failed requests

**LLM Prompt Engineering:**
- Structured outputs are way more reliable than free-form
- JSON schema validation is essential
- Token limits matter (had to trim long resumes)

**n8n Workflows:**
- Visual debugging is helpful
- Error nodes prevent whole workflow from failing
- Logging is crucial for troubleshooting

---

## Limitations

**Current:**
- Only works with PDF resumes (no DOCX)
- Can't analyze formatting/design
- LLM might hallucinate skills
- No batch processing (one at a time)

**Future improvements:**
- Add DOCX support
- Batch process multiple resumes
- Email candidates automatically
- Compare against multiple JDs
- Add human review step

---

## Files in This Repo

```
resume-screening-automation/
├── n8n-resume-scoring.json    # Complete workflow
├── README.md                   # This file
├── docs/

```

---

## Troubleshooting

**Resume not being detected:**
- Check folder ID is correct
- Verify Drive API is enabled
- Make sure trigger is activated

**LLM returns invalid JSON:**
- Check prompt includes schema
- Verify Groq API key works
- Try with shorter resume

**Sheets not updating:**
- Check sheet ID is correct
- Verify column headers match
- Test Sheets API credentials

---

## Why I Built This

**Skills demonstrated:**
- API integration (3 different APIs)
- Workflow automation
- LLM prompt engineering
- JSON validation
- Error handling

**Real use case:**
- Saves hours on resume screening
- Provides consistent evaluation
- Scales to hundreds of candidates
- Frees up time for actual interviews

---

Built by Shivarjun Reddy | arjunreddy.inc@gmail.com

*This is a learning project, not production hiring software. Use responsibly.*
