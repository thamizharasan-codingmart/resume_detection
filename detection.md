# Detection Logic Documentation

This document provides a comprehensive overview of the Resume Detection and Job Description (JD) Detection logic implemented in the dataflow service.

---

## Table of Contents

1. [Resume Detection Logic](#resume-detection-logic)
2. [JD Detection Logic](#jd-detection-logic)
3. [Common Architecture](#common-architecture)
4. [AI Classification](#ai-classification)

---

## Resume Detection Logic

### Overview

The Resume Detection service identifies resume/CV documents from email attachments using a **three-stage filtering and classification system**. The system is designed to minimize unnecessary downloads while maximizing detection accuracy.

### Architecture

The detection process follows a **multi-stage pipeline**:

```
Email Received
    ↓
Stage 1: Metadata Filtering (No Download)
    ↓
Stage 2: Filename & Context Analysis (No Download)
    ↓
Stage 3: AI Content Analysis (Download & Analyze)
    ↓
Final Result
```

### Stage 1: Metadata Filtering

**Purpose**: Filter emails based on headers and structure **without downloading attachments**.

**Location**: `resume-detection.service.ts` → `stage1MetadataFiltering()`

**Filtering Criteria**:

1. **Attachment Check**
   - Email must have at least one attachment
   - Rejects emails with no attachments

2. **Automated Sender Detection**
   - Filters out emails from automated senders using patterns:
     - `noreply@`, `no-reply@`, `automated@`, `notification@`, `system@`
   - Rejects if sender matches any pattern

3. **Negative Subject Keywords**
   - Filters out emails with negative keywords in subject:
     - `invoice`, `receipt`, `order confirmation`, `shipping`, `newsletter`, `promotion`, `marketing`, `unsubscribe`
   - Rejects if subject contains any negative keyword

4. **MIME Type Validation**
   - Accepts only resume-compatible MIME types:
     - `application/pdf`
     - `application/msword` (Word .doc)
     - `application/vnd.openxmlformats-officedocument.wordprocessingml.document` (Word .docx)
     - `application/rtf`
     - `text/plain`
     - `application/vnd.oasis.opendocument.text`
     - `image/jpeg`, `image/jpg`, `image/png`

5. **File Size Validation**
   - Accepts files between **10KB and 10MB**
   - Rejects files outside this range

**Output**: 
- `shouldProcess: boolean` - Whether to proceed to Stage 2
- `reason: string` - Reason for pass/fail
- `metadata: EmailMetadata` - Filtered email metadata with valid attachments

### Stage 2: Filename & Context Analysis

**Purpose**: Score attachments based on filename, subject, and sender patterns **without downloading**.

**Location**: `resume-detection.service.ts` → `stage2FilenameAnalysis()`

**Scoring System**: Each attachment receives a confidence score (0-100) based on:

#### 1. Filename Scoring (`scoreFilename()`)

| Pattern | Score | Description |
|---------|-------|-------------|
| Contains `resume`, `cv`, `curriculum`, `vitae` | **40** | Strong resume indicator |
| Contains `profile`, `bio` | **25** | Moderate resume indicator |
| Pattern: `[Name]_resume.pdf` or `[Name] cv.pdf` | **15** | Name + resume pattern |
| Contains `document`, `scan`, `attachment`, or 6+ digits | **0** | Negative indicator |
| Default | **8** | Neutral score |

#### 2. Subject Line Scoring (`scoreSubject()`)

| Pattern | Score | Description |
|---------|-------|-------------|
| Contains `applying for`, `application for`, `job application` | **30** | Strong application context |
| Contains `job`, `position`, `role`, `career`, `opportunity` | **15** | Moderate job-related context |
| Contains `invoice`, `receipt`, `order`, `newsletter`, `shipping` | **0** | Negative indicator |
| Default | **5** | Neutral score |

#### 3. Attachment Properties Scoring (`scoreAttachmentProperties()`)

| Criteria | Score | Description |
|----------|-------|-------------|
| Valid MIME type (from Stage 1 list) | **12** | Resume-compatible format |
| Size between 5KB and 2MB | **8** | Typical resume size range |
| **Total** | **0-20** | Combined property score |

#### 4. Sender Scoring (`scoreSender()`)

| Criteria | Score | Description |
|----------|-------|-------------|
| Personal email domain (gmail.com, yahoo.com, outlook.com, etc.) | **10** | Likely personal resume |
| Non-automated sender | **5** | Human sender |
| Automated sender | **0** | Automated system |

**Total Score Calculation**:
```
Total Score = Filename Score + Subject Score + Properties Score + Sender Score
Final Score = min(100, max(0, Total Score))
```

**Confidence Thresholds**:

| Score Range | Classification | Action |
|-------------|----------------|--------|
| **≥ 70** | High Confidence | ✅ Resume detected, no download needed |
| **40-69** | Medium Confidence | ⚠️ Requires AI analysis (Stage 3) |
| **< 40** | Low Confidence | ❌ Not a resume, reject |

**Output**: Array of `ResumeDetectionResult` objects with:
- `isResume: boolean` - Whether detected as resume (score ≥ 70)
- `confidence: number` - Confidence score (0-100)
- `reason: string` - Explanation of score
- `attachment: AttachmentMetadata` - Attachment details
- `shouldDownload: boolean` - Whether to download for Stage 3 (score ≥ 40)

### Stage 3: AI Content Analysis

**Purpose**: For attachments with confidence scores between 40-69, download and analyze content using AI.

**Location**: `resume-detection.service.ts` → `stage3AIClassification()`

**Process**:

1. **Filter Candidates**
   - Only processes attachments with `confidence >= 40 && confidence < 70`
   - Only processes if `shouldDownload === true`

2. **Download & Extract**
   - Downloads attachment from Gmail
   - Extracts text using `parseBuffer()` utility
   - Requires minimum 50 characters of extracted text

3. **AI Classification**
   - Uses OpenAI GPT-4.1-nano model
   - Analyzes extracted text for resume characteristics
   - Returns AI confidence (0.0 - 1.0)

4. **Confidence Adjustment**

   **If AI confirms resume** (isResume = true, confidence ≥ 0.7):
   ```
   Final Confidence = min(100, Stage2Score + (AIConfidence × 30))
   ```

   **If AI rejects resume** (isResume = false OR confidence < 0.7):
   ```
   Final Confidence = max(0, Stage2Score - ((1 - AIConfidence) × 30))
   ```

   **If insufficient text**:
   ```
   Final Confidence = Stage2Score × 0.5
   ```

5. **Final Classification**
   - `isResume = true` if final confidence ≥ 70
   - `isResume = false` otherwise

**AI Classification Details**: See [AI Classification](#ai-classification) section.

**Output**: Updated `ResumeDetectionResult[]` with AI-adjusted confidence scores.

### Main Entry Point

**Method**: `processEmailForResumeDetection()`

**Flow**:
1. Execute Stage 1 (Metadata Filtering)
2. If passed, execute Stage 2 (Filename Analysis)
3. If Stage 2 results have candidates (40-69 score), execute Stage 3 (AI Classification)
4. Return final results

**Output**: `EmailProcessingResult` containing:
- `resumeDetected: boolean` - Whether any attachment is a resume
- `attachments: ResumeDetectionResult[]` - Detailed results for each attachment
- `processingTime: number` - Total processing time in milliseconds

### File-Based Detection

**Method**: `processFileForResumeDetection()`

**Purpose**: Detect resume from a file buffer (used for WhatsApp uploads, direct file processing).

**Process**:
1. Validates MIME type and file size
2. Performs Stage 2 scoring (filename + properties only)
3. Extracts text from file
4. Performs AI classification
5. Returns final result

**Note**: Skips Stage 1 (email metadata) since no email context is available.

---

## JD Detection Logic

### Overview

The JD Detection service identifies Job Description documents from email attachments using a **three-stage filtering and classification system**, similar to resume detection but with JD-specific criteria.

### Architecture

Same multi-stage pipeline as Resume Detection:

```
Email Received
    ↓
Stage 1: Metadata Filtering (No Download)
    ↓
Stage 2: Filename & Context Analysis (No Download)
    ↓
Stage 3: AI Content Analysis (Download & Analyze)
    ↓
Final Result
```

### Stage 1: Metadata Filtering

**Purpose**: Filter emails based on headers and structure **without downloading attachments**.

**Location**: `jd-detection.service.ts` → `stage1MetadataFiltering()`

**Filtering Criteria**:

1. **Attachment Check**
   - Email must have at least one attachment
   - Rejects emails with no attachments

2. **Automated Sender Detection**
   - Same patterns as Resume Detection:
     - `noreply@`, `no-reply@`, `automated@`, `notification@`, `system@`
   - Rejects if sender matches any pattern

3. **Negative Subject Keywords** (JD-Specific)
   - Filters out emails with resume/application keywords:
     - `resume`, `cv`, `curriculum vitae`, `application`, `cover letter`, `applying for`, `application for`, `job application`
   - Also filters: `invoice`, `receipt`, `order confirmation`, `shipping`, `newsletter`, `promotion`, `marketing`, `unsubscribe`
   - **Key Difference**: Rejects emails that look like resume submissions

4. **MIME Type Validation**
   - Same MIME types as Resume Detection:
     - `application/pdf`
     - `application/msword` (Word .doc)
     - `application/vnd.openxmlformats-officedocument.wordprocessingml.document` (Word .docx)
     - `application/rtf`
     - `text/plain`
     - `application/vnd.oasis.opendocument.text`
     - `image/jpeg`, `image/jpg`, `image/png`

5. **File Size Validation**
   - Accepts files between **1KB and 10MB** (more lenient than resume detection)

**Output**: 
- `shouldProcess: boolean` - Whether to proceed to Stage 2
- `reason: string` - Reason for pass/fail
- `metadata: EmailMetadata` - Filtered email metadata with valid attachments
- `detailedResult` - Additional statistics about filtering

### Stage 2: Filename & Context Analysis

**Purpose**: Score attachments based on filename, subject, and sender patterns **without downloading**.

**Location**: `jd-detection.service.ts` → `stage2FilenameAnalysis()`

**Scoring System**: Each attachment receives a confidence score (0-100) based on:

#### 1. Filename Scoring (`scoreFilename()`)

| Pattern | Score | Description |
|---------|-------|-------------|
| Contains `jd`, `job-description`, `job_description`, `jobdescription`, `job-desc`, `job_desc`, `jobdesc` | **40** | Strong JD indicator |
| Contains `position-description`, `position_description`, `positiondescription`, `position-desc`, `position_desc`, `positionspec` | **40** | Position description |
| Contains `job-spec`, `job_spec`, `jobspec`, `job-specification`, `job_specification`, `jobspecification` | **40** | Job specification |
| Contains `role-description`, `role_description`, `roledescription`, `role-desc`, `role_desc`, `rolespec` | **40** | Role description |
| Contains `vacancy-description`, `vacancy_description`, `vacancydescription`, `vacancy-desc`, `vacancy_desc` | **40** | Vacancy description |
| Contains `hiring`, `hiring-requirements`, `hiring_requirements`, `hiringrequirements`, `we-are-hiring` | **40** | Hiring-related |
| Contains `job-posting`, `job_posting`, `jobposting`, `job-ad`, `job_ad`, `jobad` | **40** | Job posting/ad |
| Contains `job`, `position`, `role`, `vacancy`, `opportunity`, `opening`, `career` | **15** | Moderate JD indicator |
| Pattern: `job-123`, `position-456`, `jd-789` | **15** | Job ID pattern |
| Contains `description`, `requirements`, `specification`, `details`, `spec` | **8** | Generic document terms |
| Contains `resume`, `cv`, `curriculum`, `vitae`, `application`, `cover-letter` | **0** | Negative indicator (resume-related) |
| Contains `document`, `scan`, `attachment`, or 6+ digits | **0** | Negative indicator |
| Default | **8** | Neutral score |

#### 2. Subject Line Scoring (`scoreSubject()`)

| Pattern | Score | Description |
|---------|-------|-------------|
| Contains `job description`, `jd for`, `position description`, `hiring for`, `we are hiring`, `job opening`, `job opportunity`, `vacancy description`, `role description` | **30** | Strong JD context |
| Contains `job spec`, `job specification`, `position spec`, `position specification` | **30** | Job specification context |
| Contains `job`, `position`, `role`, `vacancy`, `opening`, `opportunity`, `career opportunity`, `new position`, `hiring` | **15** | Moderate job-related context |
| Contains `description`, `requirements`, `specification` | **5** | Generic terms |
| Contains `application for`, `applying for`, `resume`, `cv submission`, `cover letter` | **0** | Negative indicator (application-related) |
| Default | **5** | Neutral score |

#### 3. Attachment Properties Scoring (`scoreAttachmentProperties()`)

Same as Resume Detection:
- Valid MIME type: **12 points**
- Size between 5KB and 2MB: **8 points**
- **Total: 0-20 points**

#### 4. Sender Scoring (`scoreSender()`)

| Criteria | Score | Description |
|----------|-------|-------------|
| Non-personal email domain (corporate/company email) | **10** | Likely company JD |
| Personal email domain (gmail.com, yahoo.com, etc.) | **5** | Possible personal JD |
| Automated sender | **0** | Automated system |

**Key Difference**: Corporate emails score higher (companies post JDs, individuals submit resumes).

**Total Score Calculation**: Same as Resume Detection.

**Confidence Thresholds**: Same as Resume Detection:
- **≥ 70**: High Confidence ✅
- **40-69**: Medium Confidence ⚠️ (requires AI)
- **< 40**: Low Confidence ❌

**Output**: Array of `JDDetectionResult` objects with detailed scoring breakdown.

### Stage 3: AI Content Analysis

**Purpose**: For attachments with confidence scores between 40-69, download and analyze content using AI.

**Location**: `jd-detection.service.ts` → `stage3AIClassification()`

**Process**: Same as Resume Detection Stage 3, but uses JD-specific AI classifier.

**AI Classification Details**: See [AI Classification](#ai-classification) section.

### Main Entry Point

**Method**: `processEmailForJDDetection()`

**Flow**: Same as Resume Detection.

**Output**: `JDProcessingResult` containing:
- `jdDetected: boolean` - Whether any attachment is a JD
- `attachments: JDDetectionResult[]` - Detailed results for each attachment
- `processingTime: number` - Total processing time
- `stage1Result` - Detailed Stage 1 filtering statistics
- `stage2Results` - Detailed Stage 2 scoring breakdown
- `stage3Results` - Stage 3 AI classification results
- `detectionSummary` - Summary statistics (high/medium/low confidence counts, best attachment)

---

## Common Architecture

### Shared Components

Both detection services share:

1. **Multi-Stage Pipeline**: Three-stage filtering approach
2. **Scoring System**: 0-100 confidence score
3. **AI Integration**: OpenAI GPT-4.1-nano for borderline cases
4. **MIME Type Support**: Same supported file formats
5. **Error Handling**: Graceful degradation on errors

### Key Differences

| Aspect | Resume Detection | JD Detection |
|--------|-----------------|--------------|
| **Negative Keywords** | Invoice, receipt, marketing | Resume, CV, application (opposite) |
| **Filename Patterns** | `resume`, `cv`, `curriculum` | `jd`, `job-description`, `hiring` |
| **Subject Patterns** | `applying for`, `job application` | `job description`, `we are hiring` |
| **Sender Scoring** | Personal emails score higher | Corporate emails score higher |
| **File Size Min** | 10KB | 1KB |
| **AI Classifier** | Resume-specific prompts | JD-specific prompts |

### Performance Optimization

1. **No Download Until Necessary**: Stages 1 & 2 work with metadata only
2. **Selective AI Analysis**: Only borderline cases (40-69 score) trigger AI
3. **Early Rejection**: Low-confidence attachments rejected immediately
4. **Batch Processing**: Multiple attachments processed in parallel where possible

---

## AI Classification

### Overview

Both services use OpenAI's GPT-4.1-nano model for content analysis when Stage 2 scoring is ambiguous (40-69 confidence).

### Resume AI Classifier

**Location**: `resume-ai-classifier.service.ts`

**Model**: `gpt-4.1-nano`

**Text Processing**:
- Maximum text length: **10,000 characters**
- Prompt word limit: **500 words**
- Extracts first page only (using form feed `\f` or triple newline `\n\n\n`)

**Prompt Characteristics**:
- Analyzes for personal information (name, email, phone)
- Looks for professional experience sections
- Checks for education history
- Identifies skills sections
- Detects work history with dates
- Rejects job descriptions, cover letters, invoices

**Output Format**:
```json
{
  "isResume": boolean,
  "confidence": number (0.0 to 1.0),
  "reason": "short explanation",
  "insufficientText": boolean
}
```

**Confidence Adjustment**:
- AI confidence ≥ 0.7: Adds up to 30 points to Stage 2 score
- AI confidence < 0.7: Subtracts up to 30 points from Stage 2 score
- Insufficient text: Reduces Stage 2 score by 50%

### JD AI Classifier

**Location**: `jd-ai-classifier.service.ts`

**Model**: `gpt-4.1-nano`

**Text Processing**: Same as Resume AI Classifier.

**Prompt Characteristics**:
- Analyzes for job title and position information
- Looks for company/role descriptions ("We are looking for", "We are seeking")
- Checks for requirements and qualifications
- Identifies responsibilities and duties
- Detects experience requirements
- Rejects resumes, cover letters, invoices

**Output Format**: Same as Resume AI Classifier (with `isJD` instead of `isResume`).

**Confidence Adjustment**: Same as Resume AI Classifier.

### Error Handling

1. **JSON Parsing**: Cleans markdown code blocks, extracts JSON from response
2. **Retry Logic**: Falls back to simplified prompt on failure
3. **Validation**: Ensures confidence is between 0.0 and 1.0
4. **Graceful Degradation**: Returns low confidence on errors

---

## Usage Examples

### Resume Detection

```typescript
// Email-based detection
const result = await resumeDetectionService.processEmailForResumeDetection(
  token,
  messageId,
  threadId,
  mailboxId,
  mailboxEmail
);

if (result.resumeDetected) {
  const highConfidenceResumes = result.attachments.filter(
    att => att.isResume && att.confidence >= 70
  );
  // Process high confidence resumes
}

// File-based detection (WhatsApp, direct upload)
const fileResult = await resumeDetectionService.processFileForResumeDetection(
  fileBuffer,
  filename,
  mimeType,
  fileSize
);

if (fileResult.isResume && fileResult.confidence >= 70) {
  // Process resume file
}
```

### JD Detection

```typescript
// Email-based detection
const result = await jdDetectionService.processEmailForJDDetection(
  token,
  messageId,
  threadId,
  mailboxId,
  mailboxEmail
);

if (result.jdDetected) {
  const bestJD = result.detectionSummary?.bestAttachment;
  // Process best JD attachment
}

// Access detailed stage results
const stage1Stats = result.stage1Result;
const stage2Scores = result.stage2Results;
const stage3AI = result.stage3Results;
```

---

## Configuration

### Supported MIME Types

Both services support:
- `application/pdf`
- `application/msword` (.doc)
- `application/vnd.openxmlformats-officedocument.wordprocessingml.document` (.docx)
- `application/rtf`
- `text/plain`
- `application/vnd.oasis.opendocument.text`
- `image/jpeg`, `image/jpg`, `image/png`

### File Size Limits

- **Resume Detection**: 10KB - 10MB
- **JD Detection**: 1KB - 10MB

### Confidence Thresholds

- **High Confidence**: ≥ 70 (no AI needed)
- **Medium Confidence**: 40-69 (requires AI)
- **Low Confidence**: < 40 (rejected)

### AI Model Configuration

- **Model**: `gpt-4.1-nano`
- **Max Text Length**: 10,000 characters
- **Prompt Word Limit**: 500 words
- **Max Tokens**: 150
- **Temperature**: 0 (deterministic)

---

## Future Enhancements

Potential improvements:

1. **Machine Learning Model**: Train custom ML model for faster classification
2. **Caching**: Cache AI results for similar documents
3. **Batch AI Processing**: Process multiple borderline cases in single AI call
4. **Confidence Calibration**: Fine-tune confidence thresholds based on historical data
5. **Multi-language Support**: Detect resumes/JDs in multiple languages
6. **Image OCR Enhancement**: Better text extraction from image-based documents

---

## References

- **Resume Detection Service**: `src/services/resume-automation/resume-detection.service.ts`
- **JD Detection Service**: `src/services/resume-automation/jd-detection.service.ts`
- **Resume AI Classifier**: `src/services/resume-automation/resume-ai-classifier.service.ts`
- **JD AI Classifier**: `src/services/resume-automation/jd-ai-classifier.service.ts`

---

*Last Updated: 2024*

