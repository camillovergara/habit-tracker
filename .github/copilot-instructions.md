# GitHub Copilot Instructions - Habit Tracker Project

## Project Overview

A **local-first habit tracking application** built with FastAPI (Python 3.9+) and React. 
No accounts, no cloud—all data stored locally in SQLite for simple, distraction-free habit tracking.

**Full Product Requirements**: See `.claude/PRD.md`

## Core Architecture

```
React + Vite (Port 5173) ◄──HTTP/JSON──► FastAPI (Port 8000) ──► SQLite (habits.db)
```

### Tech Stack
- **Backend**: Python 3.9+, FastAPI, SQLAlchemy, SQLite
- **Frontend**: React 18, TypeScript, TanStack Query, Tailwind CSS
- **Date Handling**: date-fns

## Project Philosophy (from PRD)

1. **Simplicity First** - Minimal features, maximum utility
2. **Local & Private** - No accounts, no cloud, all data local
3. **Daily Focus** - Optimize for daily habits, avoid complexity
4. **One-Click Tracking** - Mark habits complete instantly
5. **Visual Feedback** - Streaks and completion rates motivate consistency

## Reference Documentation

Always consult these best practices documents when working on:

### Database Work
- **File**: `.claude/reference/sqlite-best-practices.md`
- **Use for**: Schema design, queries, SQLAlchemy patterns, indexing, transactions

### API Development
- **File**: `.claude/reference/fastapi-best-practices.md`
- **Use for**: Route design, async patterns, Pydantic validation, error handling

### Frontend Development
- **File**: `.claude/reference/react-frontend-best-practices.md`
- **Use for**: Component patterns, state management, TanStack Query, Tailwind CSS

### Testing
- **File**: `.claude/reference/testing-and-logging.md`
- **Use for**: Pytest patterns, test coverage, logging strategies

### Deployment
- **File**: `.claude/reference/deployment-best-practices.md`
- **Use for**: Production setup, Docker, environment configuration

## Implementation Plans

Active development plans in `.agents/plans/`:

1. **`backend-foundation.md`** - Backend setup, database, API endpoints
2. **`frontend-foundation.md`** - React setup, routing, basic UI
3. **`calendar-view.md`** - Calendar component with completion visualization
4. **`edit-delete-skip-features.md`** - CRUD operations and skip functionality

**Always reference the relevant plan** when implementing features in these areas.

## Code Standards

### Python/FastAPI (Python 3.9 Compatible)

```python
# ALWAYS use this import for Python 3.9 compatibility
from __future__ import annotations

# Use Optional[T] instead of T | None in SQLAlchemy Mapped types
from typing import Optional
from sqlalchemy.orm import Mapped, mapped_column

class Habit(Base):
    description: Mapped[Optional[str]] = mapped_column(String(500))
```

**Rules**:
- ✅ Use `from __future__ import annotations` at top of file
- ✅ Use `Optional[T]` instead of `T | None` in `Mapped` types
- ✅ Follow FastAPI async/await patterns
- ✅ Use Pydantic for all request/response validation
- ✅ Enable foreign keys: `PRAGMA foreign_keys = ON`
- ✅ Use ISO date strings (YYYY-MM-DD) for dates

### TypeScript/React

```typescript
// Prefer TanStack Query for server state
export function useHabits() {
  return useQuery({
    queryKey: ['habits'],
    queryFn: () => api.get<Habit[]>('/habits'),
  });
}

// Keep components small (<150 lines)
export function HabitCard({ habit }: Props) {
  // Functional component with hooks
}
```

**Rules**:
- ✅ Use functional components with hooks (no class components)
- ✅ Use TanStack Query (@tanstack/react-query) for server state
- ✅ Keep components small and composable (max 150 lines)
- ✅ Use Tailwind CSS for styling (no CSS modules)
- ✅ Place API client functions in `frontend/src/lib/api.ts`
- ✅ Colocate feature code: `features/habits/` contains components, hooks, types

## Database Schema

### Tables

**habits**
- `id` (INTEGER PRIMARY KEY)
- `name` (TEXT NOT NULL)
- `description` (TEXT)
- `color` (TEXT) - Hex color for UI
- `icon` (TEXT) - Icon name
- `frequency` (TEXT) - 'daily', 'weekly'
- `target_days` (INTEGER) - Days per week for weekly habits
- `archived` (BOOLEAN DEFAULT 0)
- `created_at` (TEXT NOT NULL)

**completions**
- `id` (INTEGER PRIMARY KEY)
- `habit_id` (INTEGER FK → habits.id ON DELETE CASCADE)
- `completed_date` (TEXT NOT NULL) - ISO date YYYY-MM-DD
- `status` (TEXT NOT NULL) - 'completed' or 'skipped'
- `created_at` (TEXT NOT NULL)
- UNIQUE(habit_id, completed_date)

**Key Features**:
- Foreign keys with CASCADE delete
- Unique constraint on habit_id + completed_date prevents duplicates
- Status field allows "skip" (planned absence) vs "complete"

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/habits` | List all active habits with streak stats |
| POST | `/api/habits` | Create new habit |
| PATCH | `/api/habits/{id}` | Update habit (name, description, etc.) |
| DELETE | `/api/habits/{id}` | Delete habit (CASCADE deletes completions) |
| POST | `/api/habits/{id}/complete` | Mark habit complete/skip for date |
| DELETE | `/api/habits/{id}/completions/{date}` | Undo completion |
| GET | `/api/habits/{id}/completions` | Get completion history with date range |

**Response includes calculated fields**:
- `current_streak` - Consecutive days completed
- `longest_streak` - Best streak ever
- `completion_rate` - % of days completed

## Common Patterns

### 1. SQLAlchemy Model with Relationships

```python
from __future__ import annotations
from sqlalchemy import String, Boolean, ForeignKey
from sqlalchemy.orm import Mapped, mapped_column, relationship
from typing import Optional

class Habit(Base):
    __tablename__ = "habits"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    description: Mapped[Optional[str]] = mapped_column(String(500))
    archived: Mapped[bool] = mapped_column(Boolean, default=False)
    
    # Relationship with cascade delete
    completions: Mapped[list[Completion]] = relationship(
        back_populates="habit",
        cascade="all, delete-orphan"
    )
```

### 2. FastAPI Route with Dependency Injection

```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from app.database import get_db
from app.schemas import HabitCreate, HabitResponse

router = APIRouter(prefix="/api/habits", tags=["habits"])

@router.post("", response_model=HabitResponse, status_code=201)
async def create_habit(
    habit: HabitCreate,
    db: Session = Depends(get_db)
):
    db_habit = Habit(**habit.model_dump())
    db.add(db_habit)
    db.commit()
    db.refresh(db_habit)
    return db_habit
```

### 3. React Query Hook with Mutation

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { api } from '@/lib/api';

export function useCreateHabit() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: (habit: HabitCreate) => 
      api.post<Habit>('/habits', habit),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['habits'] });
    },
  });
}
```

### 4. Streak Calculation Logic

```python
def calculate_current_streak(completions: list[Completion]) -> int:
    """Calculate consecutive days from today backwards."""
    if not completions:
        return 0
    
    # Sort by date descending
    sorted_completions = sorted(
        completions, 
        key=lambda c: c.completed_date, 
        reverse=True
    )
    
    streak = 0
    expected_date = date.today()
    
    for completion in sorted_completions:
        comp_date = date.fromisoformat(completion.completed_date)
        
        if comp_date == expected_date:
            if completion.status == 'completed':
                streak += 1
            expected_date -= timedelta(days=1)
        elif comp_date < expected_date:
            break
    
    return streak
```

## Testing Requirements

### Backend Tests (pytest)
- ✅ Test all API endpoints (GET, POST, PATCH, DELETE)
- ✅ Test streak calculation with edge cases
- ✅ Test date handling and timezone edge cases
- ✅ Validate Pydantic schema constraints
- ✅ Test CASCADE delete behavior

### Frontend Tests (Vitest + React Testing Library)
- ✅ Test component rendering with mock data
- ✅ Test user interactions (click, form submit)
- ✅ Test TanStack Query hooks with MSW
- ✅ Test calendar date calculations

**Run tests**:
```bash
# Backend
cd backend && pytest

# Frontend
cd frontend && npm test
```

## Development Workflow

### Starting the Application

```bash
# Backend (Terminal 1)
cd backend
uv sync
uv run uvicorn app.main:app --reload --port 8000

# Frontend (Terminal 2)
cd frontend
npm install
npm run dev
```

**Access**:
- Frontend: http://localhost:5173
- Backend API: http://localhost:8000
- API Docs: http://localhost:8000/docs

### PIV Loop Commands (Claude Code)

See `.claude/commands/` for full command definitions:

**Planning**:
- `/core_piv_loop:prime` - Load project context
- `/core_piv_loop:plan-feature` - Create implementation plan
- `/core_piv_loop:execute` - Execute plan step-by-step

**Validation**:
- `/validation:validate` - Run tests, linting, build
- `/validation:code-review` - Review changed files
- `/validation:execution-report` - Generate feature report

**Bug Fixing**:
- `/github_bug_fix:rca` - Root cause analysis
- `/github_bug_fix:implement-fix` - Implement fix

## File Structure

```
habit-tracker/
├── .claude/                    # AI workflow documentation
│   ├── PRD.md                  # Product Requirements Document
│   ├── commands/               # Claude slash commands
│   └── reference/              # Best practices docs
│       ├── sqlite-best-practices.md
│       ├── fastapi-best-practices.md
│       ├── react-frontend-best-practices.md
│       ├── testing-and-logging.md
│       └── deployment-best-practices.md
├── .agents/                    # Implementation plans
│   └── plans/
│       ├── backend-foundation.md
│       ├── frontend-foundation.md
│       ├── calendar-view.md
│       └── edit-delete-skip-features.md
├── backend/
│   ├── app/
│   │   ├── main.py            # FastAPI app
│   │   ├── database.py        # SQLite connection
│   │   ├── models.py          # SQLAlchemy models
│   │   ├── schemas.py         # Pydantic schemas
│   │   └── routers/           # API routes
│   └── tests/                 # pytest tests
├── frontend/
│   ├── src/
│   │   ├── features/          # Feature-based organization
│   │   │   ├── habits/
│   │   │   └── calendar/
│   │   ├── components/ui/     # Shared components
│   │   ├── lib/               # Utilities (api.ts)
│   │   └── pages/             # Route pages
│   └── package.json
└── README.md
```

## Important Notes

1. **Python 3.9 Compatibility**: Always use `from __future__ import annotations` and `Optional[T]` in Mapped types
2. **Foreign Keys**: Must be explicitly enabled in SQLite with `PRAGMA foreign_keys = ON`
3. **Date Format**: Use ISO strings (YYYY-MM-DD) consistently across backend and frontend
4. **Local-First**: No authentication, no user management—single-user application
5. **Simplicity**: Resist feature creep—focus on core habit tracking experience

## When to Reference Documentation

| Working On | Read This First |
|------------|-----------------|
| Database schema | `.claude/reference/sqlite-best-practices.md` |
| API endpoints | `.claude/reference/fastapi-best-practices.md` |
| React components | `.claude/reference/react-frontend-best-practices.md` |
| Writing tests | `.claude/reference/testing-and-logging.md` |
| Deploying | `.claude/reference/deployment-best-practices.md` |
| New feature | Relevant plan in `.agents/plans/` |
| Product questions | `.claude/PRD.md` |

## Quick Reference Links

- **Project Goals**: `.claude/PRD.md` (sections 1-2)
- **Tech Stack Details**: `.claude/PRD.md` (section 5)
- **Database Schema**: `.claude/reference/sqlite-best-practices.md` + `backend/app/models.py`
- **API Endpoints**: `README.md` + `backend/app/routers/`
- **Component Structure**: `frontend/src/features/`

---

**Remember**: This is a **simple, local-first habit tracker**. Every feature should support the core goal: making it easy to track daily habits and build streaks. Keep it minimal, keep it focused.
