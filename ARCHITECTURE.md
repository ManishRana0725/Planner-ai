# 🍽️ Restaurant Booking AI Agent — Architecture

A two-phase intelligent system that **finds the perfect restaurant** and then **books a table through a voice agent**.

---

## High-Level Architecture

```mermaid
graph LR
    A([User]) --> B[Web Frontend]
    B --> C{Phase 1: Search}
    C --> D{Phase 2: Voice Book}
    D --> E[(MongoDB)]
    E --> F([Confirmation])
```

---

## Detailed System Flow

```mermaid
graph TD
    User([User]) -->|Preferences| Frontend[Web Frontend]

    subgraph "Phase 1 — Restaurant Search"
        Frontend -->|POST /api/search| API[FastAPI Backend]
        API --> Validate[Validate & Enrich Missing Data]
        Validate --> Agent[Single Search Agent - LangGraph]
        Agent -->|Tool Call| RestAPI[Restaurant Lookup Tool]
        Agent -->|Tool Call| WeatherTool[Weather Tool]
        Agent -->|Reasoning| Rank[Rank & Recommend]
        Rank --> Results[Top Restaurants Returned]
    end

    Results -->|User Picks One| Pick[Restaurant Selected]

    subgraph "Phase 2 — Voice Booking"
        Pick -->|Connect| LK[LiveKit WebRTC]
        LK <-->|Audio Stream| VA[Voice Agent Worker]
        VA --> STT[Whisper STT]
        STT --> LLM[Booking LLM + Function Calling]
        LLM --> TTS[ElevenLabs TTS]
        LLM -->|Tool Call| BookFn[Create Booking]
        BookFn --> DB[(MongoDB)]
    end
```

---

## Phase 1 — Restaurant Search

### Request Flow

```mermaid
sequenceDiagram
    participant U as User
    participant F as Frontend
    participant API as FastAPI
    participant A as Search Agent (LangGraph)
    participant T as Tools (APIs / Mock)
    participant LLM as GPT-4o

    U->>F: Enter preferences (cuisine, budget, location, date)
    F->>API: POST /api/search
    API->>A: Pass validated request
    A->>LLM: "Find best restaurants for these preferences"
    LLM-->>A: Decide which tools to call
    A->>T: search_restaurants(cuisine, location, budget)
    T-->>A: Restaurant list (API or mock fallback)
    A->>T: get_weather(date, location)
    T-->>A: Weather forecast (API or mock fallback)
    A->>LLM: Rank results using all data
    LLM-->>A: Scored & ranked recommendations
    A-->>API: Top restaurants
    API-->>F: JSON response
    F-->>U: Display results
```

### Single Agent with Tools

One LangGraph agent handles everything — no sub-agents needed:

| Tool | Purpose | Fallback |
|------|---------|----------|
| `search_restaurants` | Find restaurants by cuisine, location, budget, dietary needs | LLM-generated mock data |
| `get_weather` | Fetch weather forecast for the booking date | LLM-generated mock data |

> **Fallback Mechanism**: If an external API key is missing or a call fails, the tool automatically returns realistic LLM-generated mock data so the pipeline never breaks.

---

## Phase 2 — Voice Booking

### Conversation Flow

```mermaid
sequenceDiagram
    participant U as User (Voice)
    participant LK as LiveKit SFU
    participant VA as Voice Agent
    participant STT as Whisper
    participant LLM as GPT-4o
    participant TTS as ElevenLabs
    participant DB as MongoDB

    U->>LK: Connect to room (mic enabled)
    LK->>VA: Audio stream
    VA->>STT: Transcribe audio
    STT-->>VA: "I'd like a table for 4"

    VA->>LLM: Process intent + context
    LLM-->>VA: "Great! What date and time?"
    VA->>TTS: Generate speech
    TTS-->>LK: Audio response
    LK-->>U: Plays response

    Note over U,DB: Conversation continues collecting: guests, date, time, special requests, seating

    VA->>LLM: All details collected, confirm?
    LLM-->>VA: Confirmation summary
    VA->>TTS: Speak confirmation
    TTS-->>LK: Audio
    LK-->>U: "Your table for 4 at Bella Italia on March 28 at 7 PM is confirmed!"

    LLM->>DB: POST /api/bookings (save)
    DB-->>LLM: Booking ID
```

### Voice Stack

| Component | Technology |
|-----------|------------|
| WebRTC SFU | LiveKit Cloud |
| Speech-to-Text | OpenAI Whisper |
| LLM | GPT-4o (function calling) |
| Text-to-Speech | ElevenLabs |
| VAD | LiveKit Agents built-in |

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/search` | Phase 1 — Search & rank restaurants |
| `POST` | `/api/bookings` | Create a new booking |
| `GET` | `/api/bookings` | List all bookings |
| `GET` | `/api/bookings/{id}` | Get a specific booking |
| `DELETE` | `/api/bookings/{id}` | Cancel a booking |

---

## Database Schema (MongoDB)

### `bookings` Collection

```mermaid
erDiagram
    BOOKING {
        ObjectId _id
        String restaurant_id
        String restaurant_name
        String customer_name
        String phone
        Int guests
        ISODate booking_datetime
        String seating_preference
        String special_requests
        String weather_summary
        String status
        ISODate created_at
    }
```

```json
{
  "_id": "ObjectId",
  "restaurant_id": "String",
  "restaurant_name": "String",
  "customer_name": "String",
  "phone": "String",
  "guests": "Int",
  "booking_datetime": "ISODate",
  "seating_preference": "Indoor / Outdoor",
  "special_requests": "String",
  "weather_summary": "String",
  "status": "Confirmed / Cancelled",
  "created_at": "ISODate"
}
```

---

## Tech Stack

```mermaid
graph TB
    subgraph Frontend
        React[React - Vite]
        LKC[LiveKit Components]
    end
    subgraph Backend
        FastAPI[Python FastAPI]
        LG[LangGraph Agent]
        LKA[LiveKit Agent SDK]
    end
    subgraph External
        OAI[OpenAI GPT-4o + Whisper]
        EL[ElevenLabs TTS]
        YELP[Yelp / Google Places API]
        OWM[OpenWeatherMap API]
    end
    subgraph Storage
        Mongo[(MongoDB Atlas)]
    end

    React --> FastAPI
    LKC --> LKA
    FastAPI --> LG
    LG --> OAI
    LG --> YELP
    LG --> OWM
    LKA --> OAI
    LKA --> EL
    FastAPI --> Mongo
```

---

## Project Structure

```
Planner-ai/
├── main.py                  # FastAPI entry point
├── requirements.txt
├── .env.example
├── agents/
│   ├── search_agent.py      # LangGraph search agent
│   └── voice_agent.py       # LiveKit voice booking agent
├── tools/
│   ├── restaurant_tool.py   # Restaurant lookup (API + mock)
│   └── weather_tool.py      # Weather lookup (API + mock)
├── models/
│   └── schemas.py           # Pydantic models
├── db/
│   └── mongo.py             # MongoDB connection & CRUD
├── frontend/                # React app
│   ├── src/
│   └── package.json
└── ARCHITECTURE.md          # This file
```
