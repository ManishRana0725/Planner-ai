# Planner-ai

User Input
   ↓
Frontend Form
   ↓
POST /plan-event
   ↓
Validation Layer
   ↓
Identify Missing Data
   ↓
AI Estimation Agent for missing data 
   ↓
Planner Agent 
  ↓
LLM API
 ↓
LLM decides required agents
 ↓
Call Agent APIs
 ↓
Combine Results
   ↓
Budget Optimizer (if budget is higher than expected ) ---- again goes to planner agent and replan it 
   ↓
Final Event Plan
   ↓
Save to MongoDB
   ↓
Return to User

