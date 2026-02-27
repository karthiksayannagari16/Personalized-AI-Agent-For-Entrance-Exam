# Feature Overview & Architecture

## 🎯 Three Major Features Added

### Feature 1: Recommendation Page with Free Resources
```
┌─────────────────────────────────────────────────────┐
│          RECOMMENDATION PAGE FLOW                    │
├─────────────────────────────────────────────────────┤
│                                                      │
│  User selects weak topic from dropdown               │
│         ↓                                            │
│  API: GET /api/resources/topic/{topic}              │
│         ↓                                            │
│  Returns: List of FreeResource objects              │
│         ↓                                            │
│  Display grouped by resource_type:                  │
│  ├─ Videos (YouTube, Khan Academy)                  │
│  ├─ Articles (Blog posts, Medium)                   │
│  ├─ PDFs (Study materials)                          │
│  ├─ Problem Sets (Practice questions)               │
│  └─ Practice Tests (Full exams)                     │
│         ↓                                            │
│  For each resource show:                            │
│  ├─ Title & Description                             │
│  ├─ Source badge                                    │
│  ├─ Difficulty: Beginner/Intermediate/Advanced      │
│  ├─ Duration (for videos)                           │
│  ├─ Visit button → Direct link                      │
│  └─ Track Progress button                           │
│                                                      │
└─────────────────────────────────────────────────────┘
```

**Key Features:**
- Dynamically fetches resources based on topic + exam type
- Sorted by difficulty level for progressive learning
- Direct links to external resources
- Integrated with progress tracking
- Can mark resources as completed

**Data Model:**
```
FreeResource {
  id: int
  topic: "Integration"           # Topic name
  exam_type: "JEE"              # Exam type
  resource_type: "video"        # video/article/pdf/problem_set/practice_test
  title: string
  description: text
  url: string
  duration_minutes: int (optional)
  difficulty_level: "intermediate"
  source: "Khan Academy"
  created_at: timestamp
}
```

---

### Feature 2: Student Dashboard & Progress Tracking
```
┌──────────────────────────────────────────────────────────┐
│              STUDENT DASHBOARD ARCHITECTURE              │
├──────────────────────────────────────────────────────────┤
│                                                           │
│  ┌─ STATISTICS SECTION ─────────────────────────────┐   │
│  │  ╔═══════════════════════════════════════════╗   │   │
│  │  ║ Total Topics │ Completed │ Hours │ Progress║   │   │
│  │  ║      8       │     3     │ 12.5  │  37.5%  ║   │   │
│  │  ╚═══════════════════════════════════════════╝   │   │
│  └────────────────────────────────────────────────────┘   │
│                                                           │
│  ┌─ TOPIC-WISE PROGRESS ──────────────────────────────┐  │
│  │ Topic 1: Integration                               │  │
│  │ ✅ Status: Completed | Progress: 100%              │  │
│  │ Performance Score: 85.5 | Last Studied: 2 hours ago │  │
│  │ 📚 3 resources available [Expand to view]          │  │
│  │                                                     │  │
│  │ Topic 2: Differentiation                           │  │
│  │ ⏳ Status: In Progress | Progress: 60%             │  │
│  │ Performance Score: 72.0 | Last Studied: 1 day ago  │  │
│  │ 📚 5 resources available [Expand to view]          │  │
│  │                                                     │  │
│  │ Topic 3: Vectors                                   │  │
│  │ ⏸️  Status: Pending | Progress: 0%                │  │
│  │ Performance Score: None | Last Studied: Never      │  │
│  │ 📚 2 resources available [Expand to view]          │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                           │
│  ┌─ RECENT Q&A ACTIVITY ──────────────────────────────┐  │
│  │ Q: What is integration by substitution?            │  │
│  │ A: [Answer preview...] [View full]                 │  │
│  │ [👍 Helpful] [👎 Not helpful]                       │  │
│  │                                                     │  │
│  │ Q: How to solve differential equations?            │  │
│  │ A: [Answer preview...] [View full]                 │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                           │
│  [View Resources] [Ask AI] [Refresh]                     │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

**Dashboard Data Structure:**
```
Dashboard {
  profile: StudentProfile
  stats: {
    total_topics: 8
    topics_completed: 3
    topics_in_progress: 2
    overall_progress_percentage: 37.5
    average_performance_score: 78.5
    total_study_hours: 12.5
    weak_topics: ["Differential Equations", "Series", ...]
    recent_chat_count: 5
  }
  topics: [
    {
      topic: "Integration",
      status: "completed",        # pending/in_progress/completed/mastered
      progress_percentage: 100,
      performance_score: 85.5,
      last_studied: "2024-02-20",
      resources_count: 3,
      free_resources: [...]       # Array of FreeResource
    },
    ...
  ]
  recent_chats: [...]             # Last 5 ChatHistory items
}
```

**Progress Tracking Model:**
```
ProgressTracking {
  id: int
  student_id: int (FK)
  topic: "Integration"
  status: "completed"
  study_date: timestamp
  time_spent_minutes: 120
  performance_score: 85.5        # 0-100
  notes: "Need more practice"
  resource_id: int (FK, optional)
}
```

**Key Metrics Calculated:**
- **Overall Progress %** = (Completed Topics / Total Topics) × 100
- **Avg Performance** = Mean of all performance_score values
- **Total Study Hours** = Sum(time_spent_minutes) / 60
- **Status Indicators:**
  - ✅ Mastered: 100% score
  - ✅ Completed: 75-99% progress
  - ⏳ In Progress: 1-74% progress
  - ⏸️ Pending: 0% progress

---

### Feature 3: AI Chatbot for Q&A
```
┌────────────────────────────────────────────────────────┐
│             AI CHATBOT INTERACTION FLOW               │
├────────────────────────────────────────────────────────┤
│                                                        │
│  USER INPUT:                                           │
│  ┌──────────────────────────────────────────────────┐ │
│  │ Question: "How to solve integration by parts?"   │ │
│  │ Topic (optional): "Calculus"                     │ │
│  │              [Get Answer Button]                 │ │
│  └──────────────────────────────────────────────────┘ │
│                                                        │
│          API: POST /api/chat/ask                       │
│          ├─ student_id                                │
│          ├─ question                                  │
│          └─ topic (optional)                          │
│                                                        │
│                    ↓                                   │
│                                                        │
│          LLM Processing (Claude):                      │
│          ├─ Context: Exam type + Target               │
│          ├─ System prompt: Expert tutor role          │
│          └─ Generate detailed answer                  │
│                                                        │
│                    ↓                                   │
│                                                        │
│  RESPONSE STORED:                                      │
│  ┌──────────────────────────────────────────────────┐ │
│  │ ChatHistory entry created:                       │ │
│  │ - question: "How to solve integration by parts?" │ │
│  │ - answer: "Integration by parts uses LIATE..."  │ │
│  │ - topic: "Calculus"                             │ │
│  │ - helpful: null (awaiting feedback)             │ │
│  │ - created_at: timestamp                         │ │
│  └──────────────────────────────────────────────────┘ │
│                                                        │
│  DISPLAY ANSWER:                                       │
│  ┌──────────────────────────────────────────────────┐ │
│  │ Q: How to solve integration by parts?            │ │
│  │                                                  │ │
│  │ A: Integration by parts is a technique used...   │ │
│  │    Formula: ∫u dv = uv - ∫v du                   │ │
│  │    LIATE rule for choosing u and dv...          │ │
│  │    Example: ∫x·e^x dx = ...                     │ │
│  │                                                  │ │
│  │                  [👍 Helpful] [👎 Not helpful]  │ │
│  └──────────────────────────────────────────────────┘ │
│                                                        │
│  FEEDBACK:                                             │
│  POST /api/chat/feedback                              │
│  ├─ chat_id: 42                                       │
│  └─ helpful: true                                     │
│                                                        │
│  CHAT HISTORY:                                         │
│  GET /api/chat/history/{student_id}                   │
│  Returns: [chat1, chat2, chat3, ...]                  │
│                                                        │
└────────────────────────────────────────────────────────┘
```

**ChatHistory Data Model:**
```
ChatHistory {
  id: int
  student_id: int (FK)
  question: "How to solve integration by parts?"
  answer: "Integration by parts is a technique..."
  topic: "Calculus" (optional)
  helpful: true/false/null
  created_at: timestamp
}
```

**AI Answer Generation Features:**
The LLM generates answers with:
- Clear explanations of concepts
- Relevant formulas and equations
- Practical tips and tricks
- Common mistakes to avoid
- Worked examples
- Exam-specific strategies

**Context-Aware Responses:**
- Adjusts complexity based on exam type (JEE, NEET, GRE)
- Aligns with target performance level
- Uses exam-relevant examples
- Includes time-saving tips

---

## 🏗️ Complete System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        FRONTEND                             │
│                    (Streamlit App)                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │   Analyze   │  │    7-Day     │  │  Recommend   │       │
│  │   Tests     │  │    Plan      │  │  Resources   │       │
│  └──────┬──────┘  └──────┬───────┘  └──────┬───────┘       │
│         │                 │                  │              │
│  ┌──────┴─────┬──────────┴──────┬───────────┴──────┐       │
│  │                                                  │       │
│  │   ┌─────────────────┐    ┌─────────────────┐   │       │
│  │   │    Dashboard    │    │   AI Chat Bot   │   │       │
│  │   │                 │    │                 │   │       │
│  │   │ - Progress View │    │ - Q&A System    │   │       │
│  │   │ - Metrics       │    │ - History       │   │       │
│  │   │ - Resources     │    │ - Feedback      │   │       │
│  │   └────────┬────────┘    └────────┬────────┘   │       │
│  └────────────┼───────────────────────┼────────────┘       │
│               │                       │                    │
└───────────────┼───────────────────────┼────────────────────┘
                │                       │
        HTTP/REST API Calls            │
                │                       │
┌───────────────┴───────────────────────┴────────────────────┐
│                    BACKEND                                 │
│                (FastAPI Server)                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  API Endpoints:                                              │
│  ├─ POST   /api/analyze-test        (Analyze mock tests)    │
│  ├─ POST   /api/revision-plan       (Generate 7-day plan)   │
│  ├─ GET    /api/resources/topic     (Get resources)         │
│  ├─ POST   /api/progress            (Track progress)        │
│  ├─ GET    /api/dashboard           (Get dashboard)         │
│  ├─ POST   /api/chat/ask            (Ask AI)                │
│  ├─ GET    /api/chat/history        (Get chat history)      │
│  └─ POST   /api/chat/feedback       (Send feedback)         │
│                                                              │
│  Core Modules:                                               │
│  ├─ models.py         (Database schemas)                    │
│  ├─ schemas.py        (Request/Response schemas)            │
│  ├─ main.py           (FastAPI routes)                      │
│  └─ llm.py            (Claude API integration)              │
│                                                              │
└─────────┬──────────────────────┬─────────────────────┬─────┘
          │                      │                     │
          │                      │                     │
┌─────────┴──┐         ┌─────────┴──┐       ┌─────────┴──┐
│  DATABASE  │         │  CLAUDE    │       │   CACHE    │
│            │         │    API     │       │            │
│ - Profiles │         │            │       │            │
│ - Tests    │         │ LLM Engine │       │ (Optional) │
│ - Plans    │         │            │       │            │
│ - Progress │         │  Analysis  │       │            │
│ - Resources│         │  Planning  │       │            │
│ - Chat     │         │  Q&A       │       │            │
└────────────┘         └────────────┘       └────────────┘
```

---

## 📊 Data Flow Examples

### Flow 1: Viewing Recommendations
```
User selects weak topic
         ↓
GET /api/resources/topic/{topic} API call
         ↓
FastAPI queries FreeResource table
  WHERE topic LIKE '{selected_topic}'
  AND exam_type = student.exam_type
  ORDER BY difficulty_level
         ↓
Returns list of resources
         ↓
Frontend renders resources grouped by type
         ↓
User clicks "Visit" or "Track Progress"
```

### Flow 2: Dashboard Update
```
User opens Dashboard
         ↓
GET /api/dashboard/{student_id} API call
         ↓
FastAPI queries:
  1. StudentProfile (basic info)
  2. MockTest (latest for weak_topics)
  3. ProgressTracking (all progress records)
  4. ChatHistory (last 5 conversations)
         ↓
Calculates metrics:
  - total_topics
  - topics_completed
  - overall_progress_percentage
  - total_study_hours
  - average_performance_score
         ↓
Returns complete dashboard JSON
         ↓
Frontend displays cards + charts + lists
```

### Flow 3: AI Chatbot Interaction
```
User submits question
         ↓
POST /api/chat/ask with {student_id, question, topic}
         ↓
FastAPI validates student exists
         ↓
Calls llm.answer_question() with context
         ↓
Claude processes with system prompt
         ↓
Returns detailed answer
         ↓
FastAPI stores in ChatHistory table
         ↓
Returns response to frontend
         ↓
Frontend displays answer
         ↓
User can provide helpful/unhelpful feedback
         ↓
PUT feedback updates ChatHistory.helpful field
```

---

## 🔄 Database Schema Relationships

```
StudentProfile (1) ──────────────┬──────────── (n) MockTest
                                 │
                                 ├──────────── (n) RevisionPlan
                                 │
                                 ├──────────── (n) ProgressTracking
                                 │
                                 └──────────── (n) ChatHistory

ProgressTracking (n) ────────────┐
                                 │
                        (optional)└──────────── (1) FreeResource
```

---

## 📈 Performance Metrics Calculation

```
┌─ Dashboard Stats Calculation ──────────────────────────┐
│                                                         │
│  total_topics = COUNT(DISTINCT topic)                  │
│                 FROM ProgressTracking                  │
│                 WHERE student_id = ?                   │
│                                                         │
│  topics_completed = COUNT(*)                           │
│                   FROM ProgressTracking                │
│                   WHERE student_id = ?                 │
│                   AND status IN ['completed','mastered']│
│                                                         │
│  overall_progress_percentage = (topics_completed /     │
│                                 total_topics) × 100     │
│                                                         │
│  average_performance_score = AVG(performance_score)    │
│                             FROM ProgressTracking       │
│                             WHERE performance_score    │
│                             IS NOT NULL               │
│                                                         │
│  total_study_hours = SUM(time_spent_minutes) / 60      │
│                      FROM ProgressTracking             │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🎨 UI Component Hierarchy

```
App
├─ Home Page
│  └─ Hero Section
│     ├─ Text Content
│     ├─ Hero Image
│     └─ Feature Cards (4)
│        ├─ Analyze Card
│        ├─ Plan Card
│        ├─ Dashboard Card
│        └─ Chat Card
│
├─ Analyze Page
│  ├─ Test Result Input
│  └─ Weak Topics Display
│
├─ Plan Page
│  ├─ Study Hours Input
│  └─ Timetable Display
│
├─ Recommendations Page ⭐
│  ├─ Topic Selector
│  ├─ Resource List
│  │  ├─ Video Resources
│  │  ├─ Article Resources
│  │  ├─ PDF Resources
│  │  ├─ Problem Sets
│  │  └─ Practice Tests
│  └─ Action Buttons
│
├─ Dashboard Page ⭐
│  ├─ Metrics Cards
│  ├─ Topic Progress Cards
│  │  ├─ Status Indicator
│  │  ├─ Progress Bar
│  │  └─ Resources List
│  ├─ Chat Activity Section
│  └─ Action Buttons
│
└─ Chat Page ⭐
   ├─ Question Input
   ├─ Topic Input
   ├─ Get Answer Button
   └─ Chat History
      ├─ Question Display
      ├─ Answer Display
      └─ Feedback Buttons
```

---

## 🚀 Deployment Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    PRODUCTION SETUP                     │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────┐                                        │
│  │    Users     │                                        │
│  └──────┬───────┘                                        │
│         │                                                │
│  ┌──────▼──────────────────────────┐                    │
│  │  Browser / Streamlit Client     │                    │
│  └──────┬──────────────────────────┘                    │
│         │ HTTPS                                          │
│  ┌──────▼──────────────────────────┐                    │
│  │  Nginx / Load Balancer          │                    │
│  └──────┬──────────────────────────┘                    │
│         │                                                │
│  ┌──────┴──────────────┬───────────────┐                │
│  │                     │               │                │
│  ▼                     ▼               ▼                │
│ ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│ │  Frontend   │  │  Frontend   │  │  Frontend   │      │
│ │ Container 1 │  │ Container 2 │  │ Container 3 │      │
│ │(Streamlit)  │  │(Streamlit)  │  │(Streamlit)  │      │
│ └──────┬──────┘  └──────┬──────┘  └──────┬──────┘      │
│        │                 │                │             │
│        └─────────────────┼────────────────┘             │
│                          │                              │
│        API Gateway / Reverse Proxy                      │
│                          │                              │
│        ┌─────────────────┼─────────────────┐            │
│        │                 │                 │            │
│        ▼                 ▼                 ▼            │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │
│   │  Backend    │  │  Backend    │  │  Backend    │   │
│   │ Container 1 │  │ Container 2 │  │ Container 3 │   │
│   │  (FastAPI)  │  │  (FastAPI)  │  │  (FastAPI)  │   │
│   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘   │
│          │                 │                │          │
│          └─────────────────┼────────────────┘          │
│                            │                           │
│        ┌───────────────────┴───────────────────┐       │
│        │                                       │       │
│        ▼                                       ▼       │
│   ┌─────────────────────┐          ┌──────────────┐   │
│   │   PostgreSQL /      │          │  Redis Cache │   │
│   │   Database Server   │          │              │   │
│   └─────────────────────┘          └──────────────┘   │
│        │                                               │
│        └────────────── Backup ────────────────┐       │
│                                              │       │
│                                    ┌─────────▼────┐   │
│                                    │ S3 / Storage │   │
│                                    └──────────────┘   │
│                                                        │
│   External:                                            │
│   ├─ Claude API (Anthropic)                            │
│   └─ Email Service (SendGrid)                          │
│                                                        │
└────────────────────────────────────────────────────────┘
```

---

## Summary

✅ **Recommendation Page** - Free resources matched to weak topics  
✅ **Dashboard Page** - Student progress tracking and analytics  
✅ **AI Chatbot** - Interactive Q&A with detailed explanations  

All features integrate seamlessly with existing analysis and planning functionality!
