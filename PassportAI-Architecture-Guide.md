# PassportAI — Complete Architecture & Setup Guide

## Project Structure

```
passport-ai/
├── frontend/                    # Next.js App
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── page.tsx             # Main chat + form page
│   │   ├── api/
│   │   │   ├── ocr/route.ts     # OCR proxy endpoint
│   │   │   ├── agent/route.ts   # AI agent endpoint
│   │   │   └── submit/route.ts  # Form submission
│   ├── components/
│   │   ├── ChatPanel/
│   │   │   ├── ChatPanel.tsx
│   │   │   ├── Message.tsx
│   │   │   ├── UploadWidget.tsx
│   │   │   ├── ExtractionCard.tsx
│   │   │   └── TypingIndicator.tsx
│   │   ├── FormPanel/
│   │   │   ├── PassportForm.tsx
│   │   │   ├── FormSection.tsx
│   │   │   └── FieldInput.tsx
│   │   └── ui/                  # shadcn components
│   ├── lib/
│   │   ├── agent.ts             # AI agent logic
│   │   ├── ocr/
│   │   │   ├── index.ts         # OCR factory
│   │   │   ├── tesseract.ts
│   │   │   ├── google-vision.ts
│   │   │   ├── azure-di.ts
│   │   │   └── aws-textract.ts
│   │   └── types.ts
│   ├── store/
│   │   └── formStore.ts         # Zustand state
│   ├── public/
│   ├── tailwind.config.ts
│   ├── next.config.ts
│   └── package.json
│
├── backend/                     # FastAPI
│   ├── main.py
│   ├── routers/
│   │   ├── ocr.py
│   │   ├── agent.py
│   │   └── forms.py
│   ├── services/
│   │   ├── ocr_service.py
│   │   ├── agent_service.py
│   │   └── form_service.py
│   ├── models/
│   │   ├── schemas.py
│   │   └── database.py
│   ├── requirements.txt
│   └── Dockerfile
│
├── docker-compose.yml
├── .env.example
└── README.md
```

---

## Database Schema (PostgreSQL)

```sql
-- Sessions table
CREATE TABLE sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  status VARCHAR(20) DEFAULT 'active' -- active | submitted | expired
);

-- Conversation history
CREATE TABLE messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id UUID REFERENCES sessions(id) ON DELETE CASCADE,
  role VARCHAR(10) NOT NULL, -- user | assistant
  content TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Document uploads
CREATE TABLE documents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id UUID REFERENCES sessions(id) ON DELETE CASCADE,
  filename VARCHAR(255) NOT NULL,
  file_type VARCHAR(50),
  file_size INTEGER,
  document_type VARCHAR(50), -- Aadhaar | PAN | Passport | DL
  ocr_provider VARCHAR(50),  -- tesseract | google | azure | aws
  ocr_raw JSONB,             -- raw OCR output
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Extracted fields
CREATE TABLE extracted_fields (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  document_id UUID REFERENCES documents(id) ON DELETE CASCADE,
  field_name VARCHAR(100) NOT NULL,
  field_value TEXT,
  confidence DECIMAL(4,3),   -- 0.000 to 1.000
  manually_corrected BOOLEAN DEFAULT FALSE,
  corrected_value TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Form submissions
CREATE TABLE form_submissions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id UUID REFERENCES sessions(id),
  reference_number VARCHAR(20) UNIQUE,
  first_name VARCHAR(100),
  last_name VARCHAR(100),
  date_of_birth DATE,
  gender VARCHAR(20),
  nationality VARCHAR(100),
  email VARCHAR(255),
  phone VARCHAR(50),
  address TEXT,
  city VARCHAR(100),
  state VARCHAR(100),
  country VARCHAR(100),
  postal_code VARCHAR(20),
  id_number VARCHAR(100),
  source_document_type VARCHAR(50),
  submitted_at TIMESTAMPTZ DEFAULT NOW(),
  ip_address INET
);

-- Indexes
CREATE INDEX idx_sessions_created ON sessions(created_at);
CREATE INDEX idx_messages_session ON messages(session_id);
CREATE INDEX idx_documents_session ON documents(session_id);
CREATE INDEX idx_submissions_ref ON form_submissions(reference_number);
```

---

## Backend (FastAPI) — `backend/main.py`

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from routers import ocr, agent, forms

app = FastAPI(title="PassportAI API", version="1.0.0")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(ocr.router, prefix="/api/ocr", tags=["OCR"])
app.include_router(agent.router, prefix="/api/agent", tags=["Agent"])
app.include_router(forms.router, prefix="/api/forms", tags=["Forms"])

@app.get("/health")
def health():
    return {"status": "ok"}
```

## OCR Service — `backend/services/ocr_service.py`

```python
import os
import base64
from abc import ABC, abstractmethod
from typing import Optional
import json

class OCRProvider(ABC):
    @abstractmethod
    async def extract(self, image_b64: str, mime_type: str) -> dict:
        pass

class TesseractProvider(OCRProvider):
    """Free, runs locally. Good for printed text."""
    async def extract(self, image_b64: str, mime_type: str) -> dict:
        import pytesseract
        from PIL import Image
        import io
        img_bytes = base64.b64decode(image_b64)
        img = Image.open(io.BytesIO(img_bytes))
        text = pytesseract.image_to_string(img, lang='eng+hin')
        data = pytesseract.image_to_data(img, output_type=pytesseract.Output.DICT)
        confidence = sum(c for c in data['conf'] if c > 0) / max(len([c for c in data['conf'] if c > 0]), 1)
        return {"raw_text": text, "confidence": confidence / 100, "provider": "tesseract"}

class GoogleVisionProvider(OCRProvider):
    """Best accuracy, handles multiple languages."""
    async def extract(self, image_b64: str, mime_type: str) -> dict:
        from google.cloud import vision
        client = vision.ImageAnnotatorClient()
        image = vision.Image(content=base64.b64decode(image_b64))
        response = client.document_text_detection(image=image)
        text = response.full_text_annotation.text
        confidence = response.full_text_annotation.pages[0].confidence if response.full_text_annotation.pages else 0.8
        return {"raw_text": text, "confidence": confidence, "provider": "google_vision"}

class AzureDocumentIntelligenceProvider(OCRProvider):
    """Best for structured forms and documents."""
    async def extract(self, image_b64: str, mime_type: str) -> dict:
        from azure.ai.documentintelligence import DocumentIntelligenceClient
        from azure.core.credentials import AzureKeyCredential
        client = DocumentIntelligenceClient(
            endpoint=os.environ["AZURE_DI_ENDPOINT"],
            credential=AzureKeyCredential(os.environ["AZURE_DI_KEY"])
        )
        body = {"base64Source": image_b64}
        poller = client.begin_analyze_document("prebuilt-idDocument", body)
        result = poller.result()
        fields = {}
        if result.documents:
            doc = result.documents[0]
            for name, field in doc.fields.items():
                if field.value_string:
                    fields[name] = {"value": field.value_string, "confidence": field.confidence}
        return {"fields": fields, "provider": "azure_di"}

class AWSTextractProvider(OCRProvider):
    """Best for complex layouts and tables."""
    async def extract(self, image_b64: str, mime_type: str) -> dict:
        import boto3
        client = boto3.client('textract', region_name=os.environ.get('AWS_REGION', 'us-east-1'))
        response = client.detect_document_text(
            Document={'Bytes': base64.b64decode(image_b64)}
        )
        lines = [b['Text'] for b in response['Blocks'] if b['BlockType'] == 'LINE']
        confidence = sum(b.get('Confidence', 0) for b in response['Blocks'] if b['BlockType'] == 'LINE')
        count = len(lines) or 1
        return {
            "raw_text": '\n'.join(lines),
            "confidence": (confidence / count) / 100,
            "provider": "aws_textract"
        }

class OCRService:
    PROVIDERS = {
        "tesseract": TesseractProvider,
        "google": GoogleVisionProvider,
        "azure": AzureDocumentIntelligenceProvider,
        "aws": AWSTextractProvider,
    }

    def get_provider(self, name: Optional[str] = None) -> OCRProvider:
        name = name or os.environ.get("OCR_PROVIDER", "tesseract")
        cls = self.PROVIDERS.get(name)
        if not cls:
            raise ValueError(f"Unknown OCR provider: {name}")
        return cls()

    async def extract_and_parse(self, image_b64: str, mime_type: str, provider: str = None) -> dict:
        prov = self.get_provider(provider)
        raw = await prov.extract(image_b64, mime_type)
        # Parse with Claude
        parsed = await self._parse_with_ai(raw)
        return {**raw, "parsed": parsed}

    async def _parse_with_ai(self, raw: dict) -> dict:
        """Use Claude to parse raw OCR into structured fields."""
        import anthropic
        client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])
        prompt = f"""Extract structured fields from this OCR text. Return ONLY valid JSON.
Fields: firstName, lastName, dob (YYYY-MM-DD), gender, nationality, address, city, state, country, postalCode, idNumber, fatherName, phone, email
Include confidence scores (0.0-1.0) for each field.

OCR Text:
{raw.get('raw_text', str(raw.get('fields', '')))}

JSON format:
{{
  "firstName": "", "lastName": "", "dob": "", "gender": "", "nationality": "",
  "address": "", "city": "", "state": "", "country": "", "postalCode": "",
  "idNumber": "", "fatherName": "", "phone": "", "email": "",
  "confidence": {{}},
  "documentType": ""
}}"""
        msg = client.messages.create(
            model="claude-sonnet-4-6", max_tokens=800,
            messages=[{"role": "user", "content": prompt}]
        )
        text = msg.content[0].text.strip()
        if text.startswith("```"):
            text = text.split("```")[1]
            if text.startswith("json"):
                text = text[4:]
        return json.loads(text)
```

## AI Agent — `backend/services/agent_service.py`

```python
import anthropic
import os
from typing import List, Dict

SYSTEM_PROMPT = """You are PassportAI, an expert passport application assistant...
[Same system prompt as in the HTML file]
"""

class AgentService:
    def __init__(self):
        self.client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

    async def chat(self, messages: List[Dict], extracted_data: dict = None) -> str:
        context = ""
        if extracted_data:
            context = f"\n\n[CONTEXT: Extracted data already available: {extracted_data}]"

        response = self.client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=1000,
            system=SYSTEM_PROMPT + context,
            messages=messages
        )
        return response.content[0].text

    async def analyze_document(self, image_b64: str, mime_type: str, messages: List[Dict]) -> dict:
        """Pass image directly to Claude Vision for extraction."""
        response = self.client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=1000,
            system=SYSTEM_PROMPT,
            messages=[
                *messages,
                {
                    "role": "user",
                    "content": [
                        {"type": "image", "source": {"type": "base64", "media_type": mime_type, "data": image_b64}},
                        {"type": "text", "text": "Extract all fields from this identity document and return in EXTRACTED_DATA format."}
                    ]
                }
            ]
        )
        return response.content[0].text
```

---

## Frontend — Key Components

### `frontend/lib/ocr/index.ts`

```typescript
export type OCRProvider = 'tesseract' | 'google' | 'azure' | 'aws';

export interface OCRResult {
  raw_text?: string;
  fields?: Record<string, { value: string; confidence: number }>;
  confidence: number;
  provider: OCRProvider;
  parsed: ExtractedData;
}

export interface ExtractedData {
  firstName: string;
  lastName: string;
  dob: string;
  gender: string;
  nationality: string;
  address: string;
  city: string;
  state: string;
  country: string;
  postalCode: string;
  idNumber: string;
  fatherName: string;
  phone: string;
  email: string;
  documentType: string;
  confidence: Record<string, number>;
}

export async function runOCR(
  file: File,
  provider: OCRProvider = 'google'
): Promise<OCRResult> {
  const formData = new FormData();
  formData.append('file', file);
  formData.append('provider', provider);

  const res = await fetch('/api/ocr', { method: 'POST', body: formData });
  if (!res.ok) throw new Error('OCR failed');
  return res.json();
}
```

### `frontend/store/formStore.ts`

```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface FormData {
  firstName: string; lastName: string; dob: string; gender: string;
  nationality: string; email: string; phone: string; address: string;
  city: string; state: string; country: string; postalCode: string;
}

interface FormStore {
  formData: FormData;
  autoFilled: Set<string>;
  messages: Message[];
  phase: 'greeting' | 'waiting-upload' | 'processing' | 'confirming' | 'done';
  setField: (key: keyof FormData, value: string) => void;
  autoFill: (data: Partial<FormData>, fields: string[]) => void;
  addMessage: (msg: Message) => void;
  setPhase: (phase: FormStore['phase']) => void;
  reset: () => void;
}

export const useFormStore = create<FormStore>()(
  persist(
    (set) => ({
      formData: { firstName:'', lastName:'', dob:'', gender:'', nationality:'',
                  email:'', phone:'', address:'', city:'', state:'', country:'', postalCode:'' },
      autoFilled: new Set(),
      messages: [],
      phase: 'greeting',
      setField: (key, value) => set(s => ({ formData: { ...s.formData, [key]: value } })),
      autoFill: (data, fields) => set(s => ({
        formData: { ...s.formData, ...data },
        autoFilled: new Set([...s.autoFilled, ...fields])
      })),
      addMessage: (msg) => set(s => ({ messages: [...s.messages, msg] })),
      setPhase: (phase) => set({ phase }),
      reset: () => set({ formData: {} as FormData, messages: [], phase: 'greeting', autoFilled: new Set() })
    }),
    { name: 'passport-ai-store' }
  )
);
```

---

## Docker Configuration

### `docker-compose.yml`

```yaml
version: '3.9'
services:
  frontend:
    build: ./frontend
    ports: ["3000:3000"]
    environment:
      - NEXT_PUBLIC_API_URL=http://backend:8000
    depends_on: [backend]

  backend:
    build: ./backend
    ports: ["8000:8000"]
    environment:
      - DATABASE_URL=postgresql://passport:secret@db:5432/passportai
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - OCR_PROVIDER=${OCR_PROVIDER:-tesseract}
      - GOOGLE_APPLICATION_CREDENTIALS=/app/gcp-creds.json
      - AZURE_DI_ENDPOINT=${AZURE_DI_ENDPOINT}
      - AZURE_DI_KEY=${AZURE_DI_KEY}
    depends_on: [db]
    volumes:
      - ./gcp-creds.json:/app/gcp-creds.json:ro

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: passportai
      POSTGRES_USER: passport
      POSTGRES_PASSWORD: secret
    volumes: [pgdata:/var/lib/postgresql/data]
    ports: ["5432:5432"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

volumes:
  pgdata:
```

### `backend/Dockerfile`

```dockerfile
FROM python:3.11-slim
WORKDIR /app
RUN apt-get update && apt-get install -y tesseract-ocr tesseract-ocr-hin libgl1 && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### `frontend/Dockerfile`

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json .
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
EXPOSE 3000
CMD ["node", "server.js"]
```

---

## Environment Variables — `.env.example`

```bash
# Core
ANTHROPIC_API_KEY=sk-ant-...
DATABASE_URL=postgresql://passport:secret@localhost:5432/passportai
REDIS_URL=redis://localhost:6379

# OCR Provider Selection (tesseract | google | azure | aws)
OCR_PROVIDER=google

# Google Vision API
GOOGLE_APPLICATION_CREDENTIALS=./gcp-creds.json

# Azure Document Intelligence
AZURE_DI_ENDPOINT=https://your-resource.cognitiveservices.azure.com/
AZURE_DI_KEY=your-azure-key

# AWS Textract
AWS_ACCESS_KEY_ID=your-key
AWS_SECRET_ACCESS_KEY=your-secret
AWS_REGION=us-east-1

# App
NEXT_PUBLIC_API_URL=http://localhost:8000
SESSION_SECRET=your-session-secret
MAX_FILE_SIZE_MB=10
```

---

## Step-by-Step Setup

### Prerequisites
- Node.js 20+, Python 3.11+, Docker & Docker Compose, PostgreSQL 15+

### Quick Start (Docker)

```bash
git clone https://github.com/your-org/passport-ai
cd passport-ai
cp .env.example .env
# Edit .env with your API keys
docker-compose up --build
# Open http://localhost:3000
```

### Manual Setup

**Frontend:**
```bash
cd frontend
npm install
cp ../.env.example .env.local
npm run dev   # http://localhost:3000
```

**Backend:**
```bash
cd backend
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
# Install Tesseract:
# macOS: brew install tesseract tesseract-lang
# Ubuntu: apt-get install tesseract-ocr tesseract-ocr-hin

# Database setup
psql -U postgres -c "CREATE DATABASE passportai;"
psql -U postgres -d passportai -f schema.sql

uvicorn main:app --reload   # http://localhost:8000
```

---

## API Endpoints

```
POST /api/ocr/analyze
  Body: multipart/form-data { file, provider? }
  Returns: { raw_text, confidence, parsed: ExtractedData }

POST /api/agent/chat
  Body: { messages: Message[], session_id?, extracted_data? }
  Returns: { response: string, extracted_data?: ExtractedData }

POST /api/agent/analyze-document
  Body: { image_b64, mime_type, session_id }
  Returns: { response: string, extracted_data: ExtractedData }

POST /api/forms/submit
  Body: FormData
  Returns: { reference_number, submitted_at }

GET  /api/forms/{reference_number}
  Returns: FormData + status

GET  /api/sessions/{session_id}/history
  Returns: Message[]
```

---

## Bonus Features Implementation

### Multi-language Support
```typescript
// Use i18next + next-intl
// languages: en, hi, ta, te, bn, mr, gu
import { useTranslations } from 'next-intl';
const t = useTranslations('PassportForm');
```

### Camera Scanning (Real-time)
```typescript
// Use getUserMedia + canvas frame capture every 2s
const stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: 'environment' } });
// Auto-detect document edges with OpenCV.js
// Trigger OCR on stable frames
```

### Voice Assistant
```typescript
// Web Speech API for input
const recognition = new webkitSpeechRecognition();
recognition.lang = 'en-IN';
recognition.onresult = (e) => sendMessage(e.results[0][0].transcript);
// Speech synthesis for output
speechSynthesis.speak(new SpeechSynthesisUtterance(response));
```

### Confidence-Based Verification Workflow
```python
# Fields with confidence < 0.7 trigger human verification
LOW_CONFIDENCE_THRESHOLD = 0.7

def get_verification_required(fields: dict) -> list[str]:
    return [
        field for field, conf in fields['confidence'].items()
        if conf < LOW_CONFIDENCE_THRESHOLD and fields.get(field)
    ]
```

### RAG-Based Document Understanding
```python
# Store document templates as vector embeddings
# Use similarity search to match detected document type
# Apply document-specific extraction rules
from anthropic import Anthropic
client = Anthropic()
# Use claude-sonnet-4-6 with extended context for RAG
```

---

## Security Considerations

1. **File Validation**: Validate MIME type server-side, not just extension
2. **File Size Limits**: Max 10MB, configurable via env
3. **Temp Storage**: Files stored in memory/tmp, never persisted permanently
4. **Rate Limiting**: 10 uploads/hour per IP via Redis
5. **CORS**: Restrict to your domain in production
6. **API Key Security**: Never expose Anthropic key to frontend
7. **PII Handling**: Hash/encrypt stored personal data, auto-delete after 30 days
8. **Input Sanitization**: Sanitize all form inputs server-side

---

## Performance Tips

- **OCR Caching**: Cache extracted results by file hash (Redis, 24h TTL)
- **Image Optimization**: Resize images to 1200px max before OCR
- **Streaming**: Use Server-Sent Events for real-time OCR progress
- **CDN**: Serve static assets via CloudFront/Vercel Edge
- **Connection Pooling**: Use asyncpg for PostgreSQL with pool of 10 connections
