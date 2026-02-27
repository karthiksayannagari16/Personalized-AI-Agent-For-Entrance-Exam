# Quick Setup Guide - Integration Instructions

## Option 1: Complete Replacement (Recommended for Fresh Start)

If you're starting fresh, replace all files with the enhanced versions:

### Step 1: Backend Files
```bash
# Replace these files in backend/ folder:
- models.py → Use models_enhanced.py
- schemas.py → Use schemas_enhanced.py  
- main.py → Use main_enhanced.py
- llm.py → Use llm_enhanced.py
```

### Step 2: Frontend Files
```bash
# Replace this file in frontend/ folder:
- app.py → Use app_enhanced.py
```

### Step 3: Run Migrations
```bash
# Start FastAPI server to auto-create tables
python -m uvicorn backend.main:app --reload
```

---

## Option 2: Incremental Integration (For Existing Code)

If you want to merge with existing code:

### Step 1: Add New Models
In `backend/models.py`, add these new classes:

```python
from sqlalchemy import Column, Integer, String, Float, Text, ForeignKey, DateTime, Boolean, JSON
from datetime import datetime

class ProgressTracking(Base):
    """Tracks student progress for dashboard and analytics"""
    __tablename__ = "progress_tracking"

    id = Column(Integer, primary_key=True, index=True)
    student_id = Column(Integer, ForeignKey("student_profiles.id"))
    topic = Column(String, index=True)
    status = Column(String, default="pending")  # pending, in_progress, completed, mastered
    study_date = Column(DateTime, default=datetime.utcnow)
    time_spent_minutes = Column(Integer, default=0)
    performance_score = Column(Float, nullable=True)
    notes = Column(Text, nullable=True)
    resource_id = Column(Integer, ForeignKey("free_resources.id"), nullable=True)

    student = relationship("StudentProfile", back_populates="progress_tracking")
    resource = relationship("FreeResource", back_populates="progress_items")

class FreeResource(Base):
    """Stores free learning resources for each topic"""
    __tablename__ = "free_resources"

    id = Column(Integer, primary_key=True, index=True)
    topic = Column(String, index=True)
    exam_type = Column(String)
    resource_type = Column(String)  # video, article, pdf, problem_set, practice_test
    title = Column(String)
    description = Column(Text)
    url = Column(String)
    duration_minutes = Column(Integer, nullable=True)
    difficulty_level = Column(String, default="beginner")
    source = Column(String)
    created_at = Column(DateTime, default=datetime.utcnow)

    progress_items = relationship("ProgressTracking", back_populates="resource")

class ChatHistory(Base):
    """Stores chat history for Q&A bot"""
    __tablename__ = "chat_history"

    id = Column(Integer, primary_key=True, index=True)
    student_id = Column(Integer, ForeignKey("student_profiles.id"))
    question = Column(Text)
    answer = Column(Text)
    topic = Column(String, nullable=True)
    helpful = Column(Boolean, nullable=True)
    created_at = Column(DateTime, default=datetime.utcnow)

    student = relationship("StudentProfile", back_populates="chat_history")
```

Also update StudentProfile relationship:
```python
class StudentProfile(Base):
    __tablename__ = "student_profiles"
    # ... existing fields ...
    
    # Add these relationships:
    progress_tracking = relationship("ProgressTracking", back_populates="student")
    chat_history = relationship("ChatHistory", back_populates="student")
```

### Step 2: Add New Schemas
In `backend/schemas.py`, add:

```python
# Free Resources
class FreeResourceCreate(BaseModel):
    topic: str
    exam_type: str
    resource_type: str
    title: str
    description: str
    url: str
    duration_minutes: Optional[int] = None
    difficulty_level: str = "beginner"
    source: str

class FreeResource(FreeResourceCreate):
    id: int
    class Config:
        from_attributes = True

# Progress Tracking
class ProgressTrackingCreate(BaseModel):
    student_id: int
    topic: str
    status: str = "pending"
    time_spent_minutes: int = 0
    performance_score: Optional[float] = None
    notes: Optional[str] = None

class ProgressTracking(BaseModel):
    id: int
    student_id: int
    topic: str
    status: str
    class Config:
        from_attributes = True

# Chat
class ChatMessageCreate(BaseModel):
    student_id: int
    question: str
    topic: Optional[str] = None

class ChatFeedback(BaseModel):
    chat_id: int
    helpful: bool
```

### Step 3: Add New API Endpoints
In `backend/main.py`, add these endpoints:

```python
# Resources
@app.post("/api/resources", response_model=schemas.FreeResource)
def add_free_resource(resource: schemas.FreeResourceCreate, db: Session = Depends(get_db)):
    db_resource = models.FreeResource(**resource.model_dump())
    db.add(db_resource)
    db.commit()
    db.refresh(db_resource)
    return db_resource

@app.get("/api/resources/topic/{topic}")
def get_resources_by_topic(topic: str, exam_type: str = None, db: Session = Depends(get_db)):
    query = db.query(models.FreeResource).filter(models.FreeResource.topic.ilike(f"%{topic}%"))
    if exam_type:
        query = query.filter(models.FreeResource.exam_type == exam_type)
    return query.order_by(models.FreeResource.difficulty_level).all()

# Progress
@app.post("/api/progress", response_model=schemas.ProgressTracking)
def track_progress(progress: schemas.ProgressTrackingCreate, db: Session = Depends(get_db)):
    db_progress = models.ProgressTracking(**progress.model_dump())
    db.add(db_progress)
    db.commit()
    db.refresh(db_progress)
    return db_progress

@app.get("/api/progress/student/{student_id}")
def get_student_progress(student_id: int, db: Session = Depends(get_db)):
    return db.query(models.ProgressTracking).filter(
        models.ProgressTracking.student_id == student_id
    ).all()

# Dashboard
@app.get("/api/dashboard/{student_id}")
def get_student_dashboard(student_id: int, db: Session = Depends(get_db)):
    student = db.query(models.StudentProfile).filter(
        models.StudentProfile.id == student_id
    ).first()
    if not student:
        raise HTTPException(status_code=404, detail="Student not found")
    
    progress = db.query(models.ProgressTracking).filter(
        models.ProgressTracking.student_id == student_id
    ).all()
    
    return {
        "profile": student,
        "progress_items": progress,
        "total_topics": len(set(p.topic for p in progress)),
        "completed_topics": len([p for p in progress if p.status in ["completed", "mastered"]])
    }

# Chat
@app.post("/api/chat/ask")
def ask_question(chat: schemas.ChatMessageCreate, db: Session = Depends(get_db)):
    student = db.query(models.StudentProfile).filter(
        models.StudentProfile.id == chat.student_id
    ).first()
    if not student:
        raise HTTPException(status_code=404, detail="Student not found")
    
    # Get AI answer
    from backend import llm
    profile_data = {
        "exam_type": student.exam_type,
        "target_marks_rank": student.target_marks_rank
    }
    answer = llm.answer_question(profile_data, chat.question, chat.topic)
    
    # Store
    db_chat = models.ChatHistory(
        student_id=chat.student_id,
        question=chat.question,
        answer=answer,
        topic=chat.topic
    )
    db.add(db_chat)
    db.commit()
    db.refresh(db_chat)
    
    return db_chat
```

### Step 4: Add New Frontend Pages
In `frontend/app.py`, add these functions and integrate into main():

```python
def show_dashboard():
    st.button("← Back to Home", on_click=navigate_to, args=("home",))
    st.markdown("## 📊 Student Dashboard")
    
    if not require_profile():
        return
    
    with st.spinner("Loading dashboard..."):
        try:
            response = requests.get(f"{API_BASE_URL}/dashboard/{st.session_state.profile_id}")
            if response.status_code == 200:
                dashboard = response.json()
                
                col1, col2, col3, col4 = st.columns(4)
                with col1:
                    st.metric("Total Topics", dashboard.get("total_topics", 0))
                with col2:
                    st.metric("Completed", dashboard.get("completed_topics", 0))
                with col3:
                    st.metric("In Progress", "-")
                with col4:
                    st.metric("Progress %", "-")
        except Exception as e:
            st.error(f"Error: {e}")

def show_recommendations():
    st.button("← Back to Home", on_click=navigate_to, args=("home",))
    st.markdown("## 📚 Topic Resources")
    
    if not require_profile():
        return
    
    if not st.session_state.weak_topics:
        st.warning("Please analyze a mock test first.")
        return
    
    selected_topic = st.selectbox("Select a topic", st.session_state.weak_topics)
    
    if selected_topic:
        with st.spinner("Loading resources..."):
            try:
                response = requests.get(f"{API_BASE_URL}/resources/topic/{selected_topic}")
                if response.status_code == 200:
                    resources = response.json()
                    if resources:
                        st.markdown(f"### Resources for {selected_topic}")
                        for res in resources:
                            with st.expander(res.get("title", "Resource")):
                                st.write(res.get("description", ""))
                                st.write(f"**Source:** {res.get('source', 'Unknown')}")
                                st.write(f"[Visit Resource]({res.get('url', '#')})")
            except Exception as e:
                st.error(f"Error: {e}")

def show_chat():
    st.button("← Back to Home", on_click=navigate_to, args=("home",))
    st.markdown("## 💬 Ask AI Tutor")
    
    if not require_profile():
        return
    
    question = st.text_input("Ask a question:")
    topic = st.text_input("Topic (optional):")
    
    if st.button("Get Answer", type="primary"):
        if not question:
            st.warning("Please ask a question!")
        else:
            with st.spinner("AI thinking..."):
                try:
                    response = requests.post(f"{API_BASE_URL}/chat/ask", json={
                        "student_id": st.session_state.profile_id,
                        "question": question,
                        "topic": topic if topic else None
                    })
                    if response.status_code == 200:
                        data = response.json()
                        st.success("Answer:")
                        st.markdown(data.get("answer", ""))
                except Exception as e:
                    st.error(f"Error: {e}")

# Update main() to route to new pages:
def main():
    initialize_session()
    
    # Add sidebar navigation
    with st.sidebar:
        st.markdown("### Navigation")
        if st.button("Home", use_container_width=True):
            navigate_to("home")
            st.rerun()
        if st.session_state.profile_id:
            if st.button("Dashboard", use_container_width=True):
                navigate_to("dashboard")
                st.rerun()
            if st.button("Resources", use_container_width=True):
                navigate_to("recommendations")
                st.rerun()
            if st.button("Ask AI", use_container_width=True):
                navigate_to("chat")
                st.rerun()
    
    # Route pages
    if st.session_state.page == "home":
        show_home()
    elif st.session_state.page == "analyze":
        show_analyze()
    elif st.session_state.page == "plan":
        show_plan()
    elif st.session_state.page == "dashboard":
        show_dashboard()
    elif st.session_state.page == "recommendations":
        show_recommendations()
    elif st.session_state.page == "chat":
        show_chat()
```

### Step 5: Update LLM Functions
In `backend/llm.py`, add:

```python
def answer_question(profile_data: dict, question: str, topic: str = None) -> str:
    exam_type = profile_data.get("exam_type", "")
    
    system_prompt = f"""You are an expert tutor for {exam_type} exams.
    Provide clear, detailed answers to student questions."""
    
    from anthropic import Anthropic
    client = Anthropic()
    
    message = client.messages.create(
        model="claude-opus-4-20250805",
        max_tokens=1500,
        system=system_prompt,
        messages=[{"role": "user", "content": question}]
    )
    
    return message.content[0].text.strip()
```

---

## Verification Checklist

After integration:

- [ ] All 3 new models created in database
- [ ] All 6+ new API endpoints working
- [ ] Frontend loads without errors
- [ ] Can navigate to all 3 new pages
- [ ] API calls return correct data
- [ ] Chat bot generates responses
- [ ] Dashboard displays metrics
- [ ] Resources display with links

---

## Testing Commands

```bash
# Test resource endpoint
curl -X GET "http://127.0.0.1:8000/api/resources/topic/Integration"

# Test dashboard
curl -X GET "http://127.0.0.1:8000/api/dashboard/1"

# Test chat (POST)
curl -X POST "http://127.0.0.1:8000/api/chat/ask" \
  -H "Content-Type: application/json" \
  -d '{"student_id": 1, "question": "What is calculus?"}'

# Test progress
curl -X GET "http://127.0.0.1:8000/api/progress/student/1"
```

---

## File Summary

| File | Purpose | Location |
|------|---------|----------|
| models_enhanced.py | New database models | backend/ |
| schemas_enhanced.py | New data schemas | backend/ |
| main_enhanced.py | New API endpoints | backend/ |
| llm_enhanced.py | New LLM functions | backend/ |
| app_enhanced.py | New frontend pages | frontend/ |
| IMPLEMENTATION_GUIDE.md | Detailed documentation | root/ |

---

## Next Steps

1. Choose Option 1 or Option 2 above
2. Replace or merge files
3. Run database migrations
4. Start backend and frontend servers
5. Test all features
6. Deploy to production

Happy coding! 🚀
