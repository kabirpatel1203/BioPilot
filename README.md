# BioPilot

# 🏗️ **BioPilot Architecture**

---

## **COMPLETE SYSTEM ARCHITECTURE**

```
┌────────────────────────────────────────────────────────────┐
│                      USER INTERFACE                        │
│                   (Next.js Frontend)                       │
│                                                            │
│  • Symptom input form                                     │
│  • Multi-turn conversation UI                             │
│  • Cure plan display                                      │
│  • Video player embedded                                  │
└─────────────────────┬──────────────────────────────────────┘
                      │
                      ▼
┌────────────────────────────────────────────────────────────┐
│                  API LAYER (Next.js)                       │
│                                                            │
│  /api/triage/route.ts    → Main orchestration             │
│  /api/analyze/route.ts   → Symptom collection             │
│  /api/cure/route.ts      → Treatment generation           │
└─────────────────────┬──────────────────────────────────────┘
                      │
                      ▼
┌────────────────────────────────────────────────────────────┐
│                   PROCESSING FLOW                          │
└────────────────────────────────────────────────────────────┘

                      ▼

┌────────────────────────────────────────────────────────────┐
│  STEP 1: SYMPTOM COLLECTION (Multi-turn)                  │
│  ──────────────────────────────────────────────            │
│                                                            │
│  BioMistral AI (HuggingFace - FREE)                       │
│  • Asks clarifying questions                              │
│  • Extracts: location, duration, severity, quality        │
│  • Continues until all required data collected            │
│                                                            │
│  Required Data Checklist:                                 │
│  □ Primary symptom                                        │
│  □ Location (if applicable)                               │
│  □ Duration (when it started)                             │
│  □ Severity (1-10 scale)                                  │
│  □ Associated symptoms                                    │
│  □ Red flag questions answered                            │
└─────────────────────┬──────────────────────────────────────┘
                      │
                      ▼
┌────────────────────────────────────────────────────────────┐
│  STEP 2: RED FLAG CHECK (Code-based rules)                │
│  ───────────────────────────────────────                  │
│                                                            │
│  Hard-coded emergency detection:                          │
│                                                            │
│  EMERGENCY (Call 911):                                    │
│  • Chest pain + shortness of breath                      │
│  • Sudden severe headache ("thunderclap")                 │
│  • Slurred speech / facial drooping                       │
│  • Difficulty breathing                                   │
│  • Loss of consciousness                                  │
│                                                            │
│  URGENT (See doctor today):                               │
│  • High fever (>103°F) + stiff neck                       │
│  • Severe abdominal pain                                  │
│  • Blood in stool/urine                                   │
│                                                            │
│  If RED FLAG detected → STOP and redirect to ER/doctor    │
│  If SAFE → Continue to diagnosis                          │
└─────────────────────┬──────────────────────────────────────┘
                      │
                      ▼
┌────────────────────────────────────────────────────────────┐
│  STEP 3: DIAGNOSIS (Symptoma API - FREE 10k/month)        │
│  ────────────────────────────────────────────             │
│                                                            │
│  API Call: POST https://api.symptoma.com/v1/analyze       │
│                                                            │
│  Input:                                                   │
│  {                                                        │
│    "symptoms": [                                          │
│      "headache",                                          │
│      "bilateral_pain",                                    │
│      "pressing_quality"                                   │
│    ],                                                     │
│    "age": 30,                                             │
│    "sex": "male",                                         │
│    "severity": 5                                          │
│  }                                                        │
│                                                            │
│  Output:                                                  │
│  {                                                        │
│    "conditions": [                                        │
│      {                                                    │
│        "name": "Tension-type headache",                   │
│        "icd10": "G44.209",                                │
│        "probability": 0.87,                               │
│        "evidence": "Bilateral pressing pain without..."   │
│      },                                                   │
│      {                                                    │
│        "name": "Migraine",                                │
│        "icd10": "G43.909",                                │
│        "probability": 0.13                                │
│      }                                                    │
│    ]                                                      │
│  }                                                        │
│                                                            │
│  Take top condition (highest probability)                 │
└─────────────────────┬──────────────────────────────────────┘
                      │
                      ▼
┌────────────────────────────────────────────────────────────┐
│  STEP 4: GET OFFICIAL TREATMENT (MedlinePlus - FREE)      │
│  ──────────────────────────────────────────────────       │
│                                                            │
│  API Call: GET MedlinePlus Connect API                    │
│  URL: https://connect.medlineplus.gov/service             │
│                                                            │
│  Parameters:                                              │
│  • mainSearchCriteria.v.c = G44.209 (ICD-10 code)         │
│  • mainSearchCriteria.v.cs = 2.16.840.1.113883.6.103      │
│  • knowledgeResponseType = application/json               │
│                                                            │
│  Output:                                                  │
│  {                                                        │
│    "feed": {                                              │
│      "entry": [{                                          │
│        "title": "Tension Headache",                       │
│        "summary": "Treatment overview...",                │
│        "link": [{                                         │
│          "href": "https://medlineplus.gov/ency/...",      │
│          "title": "Tension Headache - MedlinePlus"        │
│        }],                                                │
│        "content": "Full HTML article with:                │
│          • Medication recommendations                     │
│          • Dosages                                        │
│          • Non-drug treatments                            │
│          • When to see doctor                             │
│          • Timeline for improvement"                      │
│      }]                                                   │
│    }                                                      │
│  }                                                        │
└─────────────────────┬──────────────────────────────────────┘
                      │
                      ▼
┌────────────────────────────────────────────────────────────┐
│  STEP 5: PARSE TREATMENT INTO CURE PLAN (BioMistral)      │
│  ───────────────────────────────────────────────────      │
│                                                            │
│  BioMistral AI (HuggingFace - FREE)                       │
│                                                            │
│  Prompt:                                                  │
│  "You are a medical assistant. Parse this NIH article     │
│   into a structured 3-day cure plan.                      │
│                                                            │
│   ARTICLE: [MedlinePlus HTML content]                     │
│                                                            │
│   USER: 30yo male, severity 5/10, 2 hours duration        │
│                                                            │
│   Extract ONLY what's in the article. Output JSON:        │
│   {                                                       │
│     day1: [actions with specific steps],                  │
│     day2_3: [continuation actions],                       │
│     medications: [{name, dose, frequency, warnings}],     │
│     non_drug: [physical treatments],                      │
│     timeline: expected improvement,                       │
│     see_doctor_if: [red flags]                            │
│   }"                                                      │
│                                                            │
│  Output:                                                  │
│  {                                                        │
│    "day1": [                                              │
│      "Take Ibuprofen 400mg with food",                    │
│      "Apply heat pack to neck for 15 minutes",            │
│      "Do gentle neck stretches (3 reps)",                 │
│      "Drink 8oz water immediately"                        │
│    ],                                                     │
│    "day2_3": [                                            │
│      "Continue Ibuprofen every 6 hours if needed",        │
│      "Repeat neck stretches 2-3x daily",                  │
│      "Maintain hydration (64oz/day)"                      │
│    ],                                                     │
│    "medications": [{                                      │
│      "name": "Ibuprofen",                                 │
│      "dose": "400mg",                                     │
│      "frequency": "Every 6 hours as needed",              │
│      "max_daily": "1200mg",                               │
│      "warnings": "Don't take if stomach ulcers"           │
│    }],                                                    │
│    "timeline": "50% better in 2 hours, 90% in 24 hours",  │
│    "see_doctor_if": [                                     │
│      "Sudden severe headache",                            │
│      "Fever + stiff neck",                                │
│      "Vision changes"                                     │
│    ]                                                      │
│  }                                                        │
└─────────────────────┬──────────────────────────────────────┘
                      │
                      ▼
┌────────────────────────────────────────────────────────────┐
│  STEP 6: VALIDATE MEDICATIONS (OpenFDA - FREE)            │
│  ────────────────────────────────────────────────         │
│                                                            │
│  For each medication in cure plan:                        │
│                                                            │
│  API Call: GET OpenFDA Drug Label API                     │
│  URL: https://api.fda.gov/drug/label.json                 │
│  Query: ?search=openfda.generic_name:"ibuprofen"          │
│                                                            │
│  Output:                                                  │
│  {                                                        │
│    "results": [{                                          │
│      "dosage_and_administration": [                       │
│        "Adults: 200-400mg every 4-6 hours..."             │
│      ],                                                   │
│      "warnings": [                                        │
│        "Do not use if allergic to aspirin",               │
│        "Ask doctor if history of stomach problems"        │
│      ],                                                   │
│      "drug_interactions": [                               │
│        "May increase effects of blood thinners"           │
│      ]                                                    │
│    }]                                                     │
│  }                                                        │
│                                                            │
│  Validation:                                              │
│  ✓ Confirm 400mg is within FDA-approved range            │
│  ✓ Add official warnings to display                      │
│  ✓ Flag any discrepancies                                │
└─────────────────────┬──────────────────────────────────────┘
                      │
                      ▼
┌────────────────────────────────────────────────────────────┐
│  STEP 7: FIND OFFICIAL VIDEOS (YouTube API - FREE)        │
│  ─────────────────────────────────────────────────        │
│                                                            │
│  Search for videos from official channels only            │
│                                                            │
│  Official Channel IDs:                                    │
│  • NIH: UCQwVLe19iQ_8qXssRXNf7tg                          │
│  • Mayo Clinic: UCrIffHd7khfQIVU2dCzU8Cw                  │
│  • Cleveland Clinic: UC_xWz_rVs3OswnYPNDlCewA             │
│                                                            │
│  API Call: GET YouTube Data API v3                        │
│  https://www.googleapis.com/youtube/v3/search             │
│                                                            │
│  Parameters:                                              │
│  • q = "tension headache exercises"                       │
│  • channelId = UCQwVLe19iQ_8qXssRXNf7tg (NIH)             │
│  • type = video                                           │
│  • maxResults = 3                                         │
│                                                            │
│  Output:                                                  │
│  {                                                        │
│    "items": [                                             │
│      {                                                    │
│        "id": { "videoId": "abc123" },                     │
│        "snippet": {                                       │
│          "title": "Neck Stretches for Tension Headache",  │
│          "channelTitle": "MedlinePlus",                   │
│          "thumbnails": { "high": { "url": "..." } }       │
│        }                                                  │
│      }                                                    │
│    ]                                                      │
│  }                                                        │
│                                                            │
│  Repeat for Mayo Clinic and Cleveland Clinic channels     │
│  Collect 2-3 relevant videos total                        │
└─────────────────────┬──────────────────────────────────────┘
                      │
                      ▼
┌────────────────────────────────────────────────────────────┐
│  STEP 8: ASSEMBLE FINAL RESPONSE                          │
│  ──────────────────────────────────                       │
│                                                            │
│  Combine all data into structured output:                 │
│                                                            │
│  {                                                        │
│    "diagnosis": {                                         │
│      "condition": "Tension-type headache",                │
│      "icd10_code": "G44.209",                             │
│      "confidence": 0.87,                                  │
│      "evidence": "Based on bilateral pressing pain...",   │
│      "source": "Symptoma Medical AI"                      │
│    },                                                     │
│                                                            │
│    "cure_plan": {                                         │
│      "day1": [...actions...],                             │
│      "day2_3": [...actions...],                           │
│      "medications": [...with FDA validation...],          │
│      "timeline": "Expected improvement...",               │
│      "see_doctor_if": [...red flags...]                   │
│    },                                                     │
│                                                            │
│    "references": {                                        │
│      "primary_source": {                                  │
│        "title": "Tension Headache - MedlinePlus",         │
│        "url": "https://medlineplus.gov/ency/...",         │
│        "organization": "NIH National Library of Medicine" │
│      },                                                   │
│      "diagnosis_source": {                                │
│        "name": "Symptoma Medical AI",                     │
│        "icd10_validated": true                            │
│      },                                                   │
│      "drug_info": {                                       │
│        "source": "FDA Drug Labels Database",              │
│        "url": "https://dailymed.nlm.nih.gov/..."          │
│      }                                                    │
│    },                                                     │
│                                                            │
│    "videos": [                                            │
│      {                                                    │
│        "title": "Neck Stretches for Tension Headache",    │
│        "url": "https://youtube.com/watch?v=abc123",       │
│        "channel": "MedlinePlus (NIH)",                    │
│        "thumbnail": "https://i.ytimg.com/..."             │
│      },                                                   │
│      {                                                    │
│        "title": "Heat Therapy for Headaches",             │
│        "url": "https://youtube.com/watch?v=def456",       │
│        "channel": "Mayo Clinic",                          │
│        "thumbnail": "https://i.ytimg.com/..."             │
│      }                                                    │
│    ]                                                      │
│  }                                                        │
└─────────────────────┬──────────────────────────────────────┘
                      │
                      ▼
┌────────────────────────────────────────────────────────────┐
│  STEP 9: DISPLAY TO USER                                  │
│  ──────────────────────────                               │
│                                                            │
│  Frontend renders:                                        │
│                                                            │
│  ┌──────────────────────────────────────────────┐         │
│  │ 🎯 DIAGNOSIS                                 │         │
│  │ Tension-type headache (87% confidence)       │         │
│  │ ICD-10: G44.209                              │         │
│  │ Source: Symptoma Medical AI                  │         │
│  └──────────────────────────────────────────────┘         │
│                                                            │
│  ┌──────────────────────────────────────────────┐         │
│  │ 💊 YOUR 3-DAY CURE PLAN                      │         │
│  │ Source: NIH MedlinePlus                      │         │
│  │                                              │         │
│  │ DAY 1 (Today):                               │         │
│  │ ☐ Take Ibuprofen 400mg with food            │         │
│  │   ⚠️ FDA Warning: Don't take if ulcers       │         │
│  │ ☐ Apply heat pack to neck (15 min)          │         │
│  │ ☐ Do neck stretches (see video below)       │         │
│  │ ☐ Drink 8oz water now                       │         │
│  │                                              │         │
│  │ DAY 2-3:                                     │         │
│  │ ☐ Continue Ibuprofen every 6 hrs if needed  │         │
│  │ ☐ Neck stretches 2-3x daily                 │         │
│  │ ☐ Stay hydrated (64oz/day)                  │         │
│  └──────────────────────────────────────────────┘         │
│                                                            │
│  ┌──────────────────────────────────────────────┐         │
│  │ 🎥 INSTRUCTIONAL VIDEOS                      │         │
│  │                                              │         │
│  │ [Video Player]                               │         │
│  │ "Neck Stretches for Tension Headache"        │         │
│  │ MedlinePlus (NIH)                            │         │
│  │                                              │         │
│  │ [Video Player]                               │         │
│  │ "Heat Therapy for Headaches"                 │         │
│  │ Mayo Clinic                                  │         │
│  └──────────────────────────────────────────────┘         │
│                                                            │
│  ┌──────────────────────────────────────────────┐         │
│  │ ⏱️ EXPECTED TIMELINE                         │         │
│  │ • 50% improvement: 2 hours                   │         │
│  │ • 90% improvement: 24 hours                  │         │
│  └──────────────────────────────────────────────┘         │
│                                                            │
│  ┌──────────────────────────────────────────────┐         │
│  │ 🚨 SEE A DOCTOR IF:                          │         │
│  │ • Sudden severe "thunderclap" headache       │         │
│  │ • Fever with stiff neck                      │         │
│  │ • Vision changes or confusion                │         │
│  │ • No improvement after 48 hours              │         │
│  └──────────────────────────────────────────────┘         │
│                                                            │
│  ┌──────────────────────────────────────────────┐         │
│  │ 📚 REFERENCES                                │         │
│  │ • MedlinePlus: Tension Headache              │         │
│  │   https://medlineplus.gov/...                │         │
│  │ • FDA Drug Label: Ibuprofen                  │         │
│  │   https://dailymed.nlm.nih.gov/...           │         │
│  └──────────────────────────────────────────────┘         │
└────────────────────────────────────────────────────────────┘
```

---

## **📁 FILE STRUCTURE**

biopilot/
│
├── backend/                              # Python FastAPI backend
│   ├── main.py                          # FastAPI app entry point
│   ├── config.py                        # Environment variables & settings
│   ├── requirements.txt                 # Python dependencies
│   │
│   ├── api/                             # API routes
│   │   ├── __init__.py
│   │   ├── triage.py                   # Main orchestration endpoint
│   │   ├── analyze.py                  # Symptom collection endpoint
│   │   ├── diagnose.py                 # Diagnosis endpoint
│   │   ├── treatment.py                # Treatment retrieval endpoint
│   │   ├── cure_plan.py                # Cure plan generation endpoint
│   │   └── videos.py                   # Video finder endpoint
│   │
│   ├── services/                        # External API clients
│   │   ├── __init__.py
│   │   ├── symptoma.py                 # Symptoma API client
│   │   ├── medlineplus.py              # MedlinePlus API client
│   │   ├── biomistral.py               # HuggingFace BioMistral client
│   │   ├── openfda.py                  # OpenFDA API client
│   │   └── youtube.py                  # YouTube Data API client
│   │
│   ├── utils/                           # Utility functions
│   │   ├── __init__.py
│   │   ├── redflags.py                 # Emergency detection rules
│   │   ├── validators.py               # Input validation
│   │   └── formatters.py               # Response formatting
│   │
│   ├── models/                          # Pydantic models (data schemas)
│   │   ├── __init__.py
│   │   ├── symptom.py                  # Symptom data models
│   │   ├── diagnosis.py                # Diagnosis data models
│   │   ├── treatment.py                # Treatment data models
│   │   └── response.py                 # API response models
│   │
│   └── tests/                           # Unit tests
│       ├── __init__.py
│       ├── test_triage.py
│       ├── test_symptoma.py
│       └── test_redflags.py
│
├── frontend/                            # Next.js frontend (separate)
│   ├── app/
│   ├── components/
│   └── package.json
│
├── .env                                 # Environment variables
├── docker-compose.yml                   # Docker setup (optional)
└── README.md


## **💰 COST BREAKDOWN**

| Service | Free Tier | Monthly Cost |
|---------|-----------|--------------|
| **Symptoma API** | 10,000 requests | $0 (until 10k users) |
| **BioMistral (HuggingFace)** | 1,000/day | $0 |
| **MedlinePlus Connect** | Unlimited | $0 (government) |
| **OpenFDA** | Unlimited | $0 (government) |
| **YouTube Data API** | 10,000 units/day | $0 |
| **Vercel Hosting** | 100GB bandwidth | $0 |
| **Supabase** | 500MB DB | $0 |
| **TOTAL** | | **$0/month** |

**Scales to 10,000 users/month completely free.**

---

## **🚀 DATA FLOW EXAMPLE**

### **User Journey: "I have a headache"**

```
1. User types: "I have a headache"
   ↓
2. BioMistral asks: "Where exactly? Both sides or one side?"
   User: "Both sides"
   ↓
3. BioMistral asks: "How would you describe it?"
   User: "Dull, pressing feeling"
   ↓
4. BioMistral asks: "Rate pain 1-10"
   User: "5"
   ↓
5. BioMistral asks: "Any nausea or light sensitivity?"
   User: "No"
   ↓
6. Red flag check: ✅ SAFE
   ↓
7. Symptoma API diagnoses:
   → Tension headache (87% confidence)
   → ICD-10: G44.209
   ↓
8. MedlinePlus API fetches treatment article
   ↓
9. BioMistral formats into 3-day plan
   ↓
10. OpenFDA validates Ibuprofen dosage
   ↓
11. YouTube API finds 2 NIH videos
   ↓
12. User sees complete cure plan with videos + references
```


This architecture gives you:
- ✅ Zero monthly cost
- ✅ Medical-grade diagnosis (Symptoma)
- ✅ Official treatment (NIH)
- ✅ Full references and videos
- ✅ Scales to 10,000 users free

