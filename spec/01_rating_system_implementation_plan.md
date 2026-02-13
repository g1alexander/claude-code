# Platziflix Rating System - Implementation Plan

**Document Version:** 1.0
**Created:** 2026-02-13
**Status:** Proposed
**Author:** Architect Agent

---

## Executive Summary

The rating system is architecturally complete on the backend but lacks frontend integration and a critical authentication layer. This plan proposes a **phased approach** prioritizing immediate value delivery while building toward a complete solution.

**Key Decision: MVP-First Strategy**

Given the authentication blocker, I recommend a **temporary user simulation approach** for Phase 1, allowing immediate rating functionality while we build proper auth in Phase 2. This delivers value fast and validates the UX before heavy backend investment.

**Timeline Overview:**
- Phase 1 (MVP): 3-4 days - Interactive ratings with simulated users
- Phase 2 (Auth): 5-7 days - Full authentication system
- Phase 3 (Polish): 2-3 days - Advanced features and optimization
- **Total: 10-14 days**

---

## Current State Analysis

### Backend: 100% Complete ✅

**Implemented:**
- Database table `course_ratings` with constraints (CHECK 1-5, UNIQUE, FK, soft delete)
- Model: `CourseRating` in `Backend/app/models/course_rating.py`
- Service: 8 methods in `CourseService` for full CRUD + stats
- 6 REST endpoints: POST, GET, PUT, DELETE for ratings
- Schemas: `RatingRequest`, `RatingResponse`, `RatingStatsResponse`
- 32 passing tests

**Endpoints Available:**
```
POST   /courses/{id}/ratings              → Create rating (201)
GET    /courses/{id}/ratings              → List all ratings (200)
GET    /courses/{id}/ratings/stats        → Stats + distribution (200)
GET    /courses/{id}/ratings/user/{uid}   → User's rating (200/204)
PUT    /courses/{id}/ratings/{uid}        → Update rating (200)
DELETE /courses/{id}/ratings/{uid}        → Soft delete (204)
```

### Frontend: 20% Complete ⚠️

**Implemented:**
- Types: `CourseRating`, `RatingRequest`, `RatingStats` in `src/types/rating.ts`
- API Client: `ratingsApi.ts` with 6 methods (currently unused)
- Component: `StarRating` (read-only display, 35 tests)
- Home page: Shows ratings on course cards
- Course detail: NO rating display or interaction

**Critical Gaps:**
1. No interactive rating submission UI
2. No rating management (edit/delete)
3. No rating display on course detail page
4. API client methods exist but are not integrated

### Authentication: 0% Complete 🔴

**Missing:**
- No User model in backend
- No JWT/session system
- `user_id` accepted without validation
- Frontend has no auth context
- No login/register flows

---

## Phase 1: MVP - Interactive Rating UI (No Auth Required)

**Goal:** Enable users to rate courses immediately with a simulated user system.

**Strategy:** Use localStorage to track a temporary `guest_user_id` (UUID). This allows testing the full rating flow while auth is built in parallel.

**Duration:** 3-4 days

### 1.1 Simulated User Context (2-3 hours)

**Create:** `Frontend/src/contexts/UserContext.tsx`

```typescript
'use client';

import { createContext, useContext, useEffect, useState } from 'react';

interface UserContextType {
  userId: string;
  isGuest: boolean;
  isAuthenticated: false; // Always false in Phase 1
}

export const UserContext = createContext<UserContextType | null>(null);

export function UserProvider({ children }) {
  const [userId, setUserId] = useState<string>('');

  useEffect(() => {
    // Get or create guest user ID
    let guestId = localStorage.getItem('platziflix_guest_id');
    if (!guestId) {
      guestId = `guest_${crypto.randomUUID()}`;
      localStorage.setItem('platziflix_guest_id', guestId);
    }
    setUserId(guestId);
  }, []);

  return (
    <UserContext.Provider value={{ userId, isGuest: true, isAuthenticated: false }}>
      {children}
    </UserContext.Provider>
  );
}

export const useUser = () => {
  const context = useContext(UserContext);
  if (!context) throw new Error('useUser must be used within UserProvider');
  return context;
};
```

**Modify:** `Frontend/src/app/layout.tsx`

```typescript
import { UserProvider } from '@/contexts/UserContext';

export default function RootLayout({ children }) {
  return (
    <html lang="es">
      <body>
        <UserProvider>
          {children}
        </UserProvider>
      </body>
    </html>
  );
}
```

**Testing:**
- Create `Frontend/src/contexts/__tests__/UserContext.test.tsx`
- Test: generates UUID, persists to localStorage, provides userId
- Mock localStorage with `vi.stubGlobal()`

---

### 1.2 Interactive StarRating Component (3-4 hours)

**Strategy:** Create a NEW component `InteractiveStarRating` to avoid breaking existing read-only `StarRating`.

**Create:** `Frontend/src/components/StarRating/InteractiveStarRating.tsx`

```typescript
'use client';

import { useState } from 'react';
import styles from './InteractiveStarRating.module.scss';

interface InteractiveStarRatingProps {
  currentRating?: number; // User's existing rating (0 if none)
  onRate: (rating: number) => Promise<void>;
  disabled?: boolean;
  courseId: number;
}

export default function InteractiveStarRating({
  currentRating = 0,
  onRate,
  disabled = false,
  courseId
}: InteractiveStarRatingProps) {
  const [hoverRating, setHoverRating] = useState(0);
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [tempRating, setTempRating] = useState(currentRating);

  const handleClick = async (rating: number) => {
    setIsSubmitting(true);
    try {
      await onRate(rating);
      setTempRating(rating);
    } catch (error) {
      // Error handled by parent
    } finally {
      setIsSubmitting(false);
    }
  };

  const displayRating = hoverRating || tempRating;

  return (
    <div className={styles.container}>
      <div className={styles.stars} role="radiogroup" aria-label="Rate this course">
        {[1, 2, 3, 4, 5].map((star) => (
          <button
            key={star}
            type="button"
            disabled={disabled || isSubmitting}
            onClick={() => handleClick(star)}
            onMouseEnter={() => setHoverRating(star)}
            onMouseLeave={() => setHoverRating(0)}
            className={styles.starButton}
            aria-label={`Rate ${star} stars`}
            aria-checked={tempRating === star}
          >
            <svg
              className={`${styles.star} ${star <= displayRating ? styles.filled : ''}`}
              viewBox="0 0 24 24"
            >
              <path d="M12 17.27L18.18 21l-1.64-7.03L22 9.24l-7.19-.61L12 2 9.19 8.63 2 9.24l5.46 4.73L5.82 21z" />
            </svg>
          </button>
        ))}
      </div>
      {isSubmitting && <span className={styles.loading}>Saving...</span>}
      {tempRating > 0 && (
        <span className={styles.feedback}>
          You rated this course {tempRating} star{tempRating !== 1 ? 's' : ''}
        </span>
      )}
    </div>
  );
}
```

**Create:** `Frontend/src/components/StarRating/InteractiveStarRating.module.scss`

```scss
.container {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.stars {
  display: flex;
  gap: 0.25rem;
}

.starButton {
  background: none;
  border: none;
  padding: 0;
  cursor: pointer;
  transition: transform 0.1s;

  &:hover:not(:disabled) {
    transform: scale(1.15);
  }

  &:disabled {
    cursor: not-allowed;
    opacity: 0.5;
  }
}

.star {
  width: 32px;
  height: 32px;
  fill: color(grey-light);
  transition: fill 0.2s;

  &.filled {
    fill: color(warning);
  }
}

.loading {
  font-size: 0.875rem;
  color: color(grey);
}

.feedback {
  font-size: 0.875rem;
  color: color(success);
}
```

**Testing:**
- Create `Frontend/src/components/StarRating/__tests__/InteractiveStarRating.test.tsx`
- Test: renders 5 clickable stars, hover effects, calls onRate, shows loading, shows feedback
- Test: respects disabled state, respects currentRating
- Mock onRate with `vi.fn()`

---

### 1.3 Rating Section Component (2-3 hours)

**Create:** `Frontend/src/components/CourseDetail/RatingSection.tsx`

```typescript
'use client';

import { useState, useEffect } from 'react';
import { useUser } from '@/contexts/UserContext';
import { ratingsApi } from '@/services/ratingsApi';
import InteractiveStarRating from '@/components/StarRating/InteractiveStarRating';
import StarRating from '@/components/StarRating/StarRating';
import type { RatingStats } from '@/types/rating';
import styles from './RatingSection.module.scss';

interface RatingSectionProps {
  courseId: number;
  initialStats: RatingStats; // From SSR
}

export default function RatingSection({ courseId, initialStats }: RatingSectionProps) {
  const { userId } = useUser();
  const [userRating, setUserRating] = useState<number | null>(null);
  const [stats, setStats] = useState<RatingStats>(initialStats);
  const [error, setError] = useState<string | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    async function fetchUserRating() {
      try {
        const response = await ratingsApi.getUserRating(courseId, userId);
        setUserRating(response?.rating ?? null);
      } catch (err) {
        console.error('Failed to fetch user rating:', err);
      } finally {
        setIsLoading(false);
      }
    }

    if (userId) {
      fetchUserRating();
    }
  }, [courseId, userId]);

  const handleRate = async (rating: number) => {
    setError(null);
    try {
      if (userRating === null) {
        await ratingsApi.createRating(courseId, { user_id: userId, rating });
      } else {
        await ratingsApi.updateRating(courseId, userId, { user_id: userId, rating });
      }

      setUserRating(rating);

      // Refresh stats
      const newStats = await ratingsApi.getRatingStats(courseId);
      setStats(newStats);
    } catch (err) {
      setError((err as Error).message || 'Failed to submit rating');
      throw err;
    }
  };

  if (isLoading) {
    return <div className={styles.loading}>Loading your rating...</div>;
  }

  return (
    <section className={styles.section}>
      <h2 className={styles.title}>Rate this course</h2>

      <div className={styles.yourRating}>
        <h3 className={styles.subtitle}>Your Rating</h3>
        <InteractiveStarRating
          currentRating={userRating ?? 0}
          onRate={handleRate}
          courseId={courseId}
        />
        {error && <p className={styles.error}>{error}</p>}
      </div>

      <div className={styles.overallRating}>
        <h3 className={styles.subtitle}>Overall Rating</h3>
        <div className={styles.stats}>
          <StarRating rating={stats.average_rating} size="large" />
          <span className={styles.count}>({stats.total_ratings} ratings)</span>
        </div>

        {/* Rating Distribution */}
        <div className={styles.distribution}>
          {[5, 4, 3, 2, 1].map((stars) => {
            const count = stats.rating_distribution?.[stars] || 0;
            const percentage = stats.total_ratings > 0
              ? (count / stats.total_ratings) * 100
              : 0;

            return (
              <div key={stars} className={styles.distributionRow}>
                <span className={styles.distributionLabel}>
                  {stars} star{stars !== 1 ? 's' : ''}
                </span>
                <div className={styles.distributionBar}>
                  <div
                    className={styles.distributionFill}
                    style={{ width: `${percentage}%` }}
                  />
                </div>
                <span className={styles.distributionCount}>{count}</span>
              </div>
            );
          })}
        </div>
      </div>
    </section>
  );
}
```

**Create:** `Frontend/src/components/CourseDetail/RatingSection.module.scss`

```scss
.section {
  padding: 2rem 0;
  border-top: 1px solid color(grey-light);
}

.title {
  font-size: 1.5rem;
  margin-bottom: 1.5rem;
}

.subtitle {
  font-size: 1.125rem;
  margin-bottom: 0.75rem;
}

.yourRating {
  margin-bottom: 2rem;
}

.error {
  color: color(danger);
  font-size: 0.875rem;
  margin-top: 0.5rem;
}

.stats {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin-bottom: 1rem;
}

.count {
  color: color(grey);
  font-size: 0.875rem;
}

.distribution {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  max-width: 400px;
}

.distributionRow {
  display: grid;
  grid-template-columns: 60px 1fr 40px;
  align-items: center;
  gap: 0.75rem;
  font-size: 0.875rem;
}

.distributionLabel {
  text-align: right;
  color: color(grey);
}

.distributionBar {
  height: 8px;
  background: color(grey-light);
  border-radius: 4px;
  overflow: hidden;
}

.distributionFill {
  height: 100%;
  background: color(warning);
  transition: width 0.3s;
}

.distributionCount {
  text-align: right;
  color: color(grey);
}

.loading {
  padding: 2rem;
  text-align: center;
  color: color(grey);
}
```

**Testing:**
- Create `Frontend/src/components/CourseDetail/__tests__/RatingSection.test.tsx`
- Test: renders interactive rating, fetches user rating, submits rating, updates stats
- Test: handles API errors, shows distribution chart
- Mock ratingsApi with `vi.mock()`

---

### 1.4 Integrate into Course Detail Page (1-2 hours)

**Modify:** `Frontend/src/app/course/[slug]/page.tsx`

```typescript
import RatingSection from '@/components/CourseDetail/RatingSection';
import { ratingsApi } from '@/services/ratingsApi';

export default async function CoursePage({ params }: { params: { slug: string } }) {
  const { slug } = await params;

  // Existing course fetch
  const response = await fetch(`http://localhost:8000/courses/${slug}`, {
    cache: 'no-store'
  });

  if (!response.ok) {
    if (response.status === 404) notFound();
    throw new Error('Failed to fetch course');
  }

  const course = await response.json();

  // NEW: Fetch rating stats
  let ratingStats = {
    average_rating: 0,
    total_ratings: 0,
    rating_distribution: { 1: 0, 2: 0, 3: 0, 4: 0, 5: 0 }
  };

  try {
    const statsResponse = await fetch(
      `http://localhost:8000/courses/${course.id}/ratings/stats`,
      { cache: 'no-store' }
    );
    if (statsResponse.ok) {
      ratingStats = await statsResponse.json();
    }
  } catch (error) {
    console.error('Failed to fetch rating stats:', error);
  }

  return (
    <div className={styles.container}>
      <CourseDetail course={course} />

      {/* NEW: Add rating section */}
      <RatingSection courseId={course.id} initialStats={ratingStats} />
    </div>
  );
}
```

---

### 1.5 Backend Adjustment - Change user_id Type (2 hours)

**Modify:** `Backend/app/models/course_rating.py`

```python
class CourseRating(BaseModel):
    __tablename__ = "course_ratings"

    course_id = Column(Integer, ForeignKey("courses.id"), nullable=False, index=True)
    user_id = Column(String, nullable=False, index=True)  # Changed from Integer to String
    rating = Column(
        Integer,
        CheckConstraint("rating >= 1 AND rating <= 5", name="ck_course_ratings_rating_range"),
        nullable=False
    )

    __table_args__ = (
        UniqueConstraint("course_id", "user_id", "deleted_at", name="uq_course_ratings_user_course_deleted"),
        {"extend_existing": True}
    )
```

**Modify:** `Backend/app/schemas/rating.py`

```python
class RatingRequest(BaseModel):
    user_id: str  # Changed from int
    rating: int = Field(ge=1, le=5)

class RatingResponse(BaseModel):
    id: int
    course_id: int
    user_id: str  # Changed from int
    rating: int
    created_at: datetime
    updated_at: datetime
```

**Create Migration:**

```bash
cd Backend
docker compose exec api uv run alembic -c app/alembic.ini revision --autogenerate -m "change_user_id_to_string_for_guest_support"
```

Edit the generated migration:

```python
def upgrade():
    op.alter_column('course_ratings', 'user_id',
                    existing_type=sa.Integer(),
                    type_=sa.String(),
                    existing_nullable=False)

def downgrade():
    op.alter_column('course_ratings', 'user_id',
                    existing_type=sa.String(),
                    type_=sa.Integer(),
                    existing_nullable=False)
```

**Run Migration:**

```bash
docker compose exec api uv run alembic -c app/alembic.ini upgrade head
```

**Update Tests:**

```python
# In Backend/app/tests/test_rating_endpoints.py
# Change all test fixtures to use string user_id
def test_add_rating(client, mock_course_service):
    response = client.post("/courses/1/ratings", json={
        "user_id": "test_user_123",  # Changed from integer
        "rating": 5
    })
    assert response.status_code == 201
```

---

### Phase 1 Deliverables

**Working Features:**
- ✅ Users can rate courses with 1-5 stars
- ✅ Ratings persist to database
- ✅ Rating stats update in real-time
- ✅ Distribution chart shows rating breakdown
- ✅ Guest user system with localStorage

**Files Created (9 files):**
1. `Frontend/src/contexts/UserContext.tsx`
2. `Frontend/src/contexts/__tests__/UserContext.test.tsx`
3. `Frontend/src/components/StarRating/InteractiveStarRating.tsx`
4. `Frontend/src/components/StarRating/InteractiveStarRating.module.scss`
5. `Frontend/src/components/StarRating/__tests__/InteractiveStarRating.test.tsx`
6. `Frontend/src/components/CourseDetail/RatingSection.tsx`
7. `Frontend/src/components/CourseDetail/RatingSection.module.scss`
8. `Frontend/src/components/CourseDetail/__tests__/RatingSection.test.tsx`
9. `Backend/app/alembic/versions/XXXXX_change_user_id_to_string.py`

**Files Modified (5 files):**
1. `Frontend/src/app/layout.tsx`
2. `Frontend/src/app/course/[slug]/page.tsx`
3. `Backend/app/models/course_rating.py`
4. `Backend/app/schemas/rating.py`
5. `Backend/app/tests/test_rating_endpoints.py`

**Test Coverage:**
- 10+ new frontend tests
- Backend tests updated for string user_id
- All existing tests still passing

**Timeline:** 3-4 days

---

## Phase 2: Authentication System (Full Production)

**Goal:** Replace guest users with real authentication (JWT-based).

**Strategy:** Implement a lightweight JWT auth system following FastAPI best practices.

**Duration:** 5-7 days

### 2.1 Backend - User Model & Auth (1-2 days)

**Create:** `Backend/app/models/user.py`

```python
from sqlalchemy import Column, String, Boolean
from sqlalchemy.orm import relationship
from app.models.base import BaseModel

class User(BaseModel):
    __tablename__ = "users"

    email = Column(String, unique=True, nullable=False, index=True)
    username = Column(String, unique=True, nullable=False, index=True)
    hashed_password = Column(String, nullable=False)
    full_name = Column(String, nullable=True)
    is_active = Column(Boolean, default=True, nullable=False)
    is_superuser = Column(Boolean, default=False, nullable=False)

    # Relationships
    ratings = relationship("CourseRating", back_populates="user")
```

**Create:** `Backend/app/schemas/user.py`

```python
from pydantic import BaseModel, EmailStr, Field
from datetime import datetime

class UserCreate(BaseModel):
    email: EmailStr
    username: str = Field(min_length=3, max_length=50)
    password: str = Field(min_length=8)
    full_name: str | None = None

class UserLogin(BaseModel):
    email: EmailStr
    password: str

class UserResponse(BaseModel):
    id: int
    email: str
    username: str
    full_name: str | None
    is_active: bool
    created_at: datetime

    class Config:
        from_attributes = True

class TokenResponse(BaseModel):
    access_token: str
    token_type: str = "bearer"
    user: UserResponse
```

**Create:** `Backend/app/core/security.py`

```python
from datetime import datetime, timedelta
from typing import Any
from jose import jwt
from passlib.context import CryptContext
from app.core.config import settings

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

ALGORITHM = "HS256"

def create_access_token(subject: str | Any, expires_delta: timedelta = None) -> str:
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES)

    to_encode = {"exp": expire, "sub": str(subject)}
    encoded_jwt = jwt.encode(to_encode, settings.SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password: str) -> str:
    return pwd_context.hash(password)
```

**Modify:** `Backend/app/core/config.py`

```python
class Settings(BaseSettings):
    # Existing settings...
    SECRET_KEY: str = "your-secret-key-change-in-production"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 60 * 24 * 7  # 7 days
```

**Create:** `Backend/app/services/auth_service.py`

```python
from sqlalchemy.orm import Session
from app.models.user import User
from app.schemas.user import UserCreate, UserLogin
from app.core.security import verify_password, get_password_hash, create_access_token
from fastapi import HTTPException, status

class AuthService:
    def __init__(self, db: Session):
        self.db = db

    def register(self, user_data: UserCreate) -> User:
        existing = self.db.query(User).filter(
            (User.email == user_data.email) | (User.username == user_data.username)
        ).first()
        if existing:
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail="Email or username already registered"
            )

        user = User(
            email=user_data.email,
            username=user_data.username,
            hashed_password=get_password_hash(user_data.password),
            full_name=user_data.full_name
        )
        self.db.add(user)
        self.db.commit()
        self.db.refresh(user)
        return user

    def login(self, credentials: UserLogin) -> tuple[User, str]:
        user = self.db.query(User).filter(User.email == credentials.email).first()
        if not user or not verify_password(credentials.password, user.hashed_password):
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Incorrect email or password"
            )

        if not user.is_active:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Account is inactive"
            )

        access_token = create_access_token(subject=user.id)
        return user, access_token

    def get_current_user(self, token: str) -> User:
        from jose import JWTError, jwt
        from app.core.security import ALGORITHM
        from app.core.config import settings

        try:
            payload = jwt.decode(token, settings.SECRET_KEY, algorithms=[ALGORITHM])
            user_id: int = int(payload.get("sub"))
            if user_id is None:
                raise HTTPException(
                    status_code=status.HTTP_401_UNAUTHORIZED,
                    detail="Invalid token"
                )
        except JWTError:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Invalid token"
            )

        user = self.db.query(User).filter(User.id == user_id).first()
        if user is None:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail="User not found"
            )

        return user
```

**Create:** `Backend/app/dependencies.py`

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from sqlalchemy.orm import Session
from app.db.base import get_db
from app.services.auth_service import AuthService
from app.models.user import User

security = HTTPBearer()

def get_auth_service(db: Session = Depends(get_db)) -> AuthService:
    return AuthService(db)

def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    auth_service: AuthService = Depends(get_auth_service)
) -> User:
    return auth_service.get_current_user(credentials.credentials)
```

**Create:** `Backend/app/routers/auth.py`

```python
from fastapi import APIRouter, Depends
from app.schemas.user import UserCreate, UserLogin, UserResponse, TokenResponse
from app.services.auth_service import AuthService
from app.dependencies import get_auth_service, get_current_user
from app.models.user import User

router = APIRouter(prefix="/auth", tags=["Authentication"])

@router.post("/register", response_model=TokenResponse, status_code=201)
def register(
    user_data: UserCreate,
    auth_service: AuthService = Depends(get_auth_service)
):
    user = auth_service.register(user_data)
    access_token = auth_service.create_access_token(subject=user.id)
    return TokenResponse(
        access_token=access_token,
        user=UserResponse.from_orm(user)
    )

@router.post("/login", response_model=TokenResponse)
def login(
    credentials: UserLogin,
    auth_service: AuthService = Depends(get_auth_service)
):
    user, access_token = auth_service.login(credentials)
    return TokenResponse(
        access_token=access_token,
        user=UserResponse.from_orm(user)
    )

@router.get("/me", response_model=UserResponse)
def get_me(current_user: User = Depends(get_current_user)):
    return UserResponse.from_orm(current_user)
```

**Modify:** `Backend/app/main.py`

```python
from app.routers import auth

app.include_router(auth.router)
```

**Add Dependencies to pyproject.toml:**

```toml
python-jose = {extras = ["cryptography"], version = "^3.3.0"}
passlib = {extras = ["bcrypt"], version = "^1.7.4"}
```

**Create Migration:**

```bash
docker compose exec api uv run alembic -c app/alembic.ini revision --autogenerate -m "add_users_table_and_auth"
```

**Testing:**
- Create `Backend/app/tests/test_auth_service.py`
- Create `Backend/app/tests/test_auth_endpoints.py`
- Test: register, login, token validation, duplicate email, wrong password

---

### 2.2 Protect Rating Endpoints (3-4 hours)

**Modify:** `Backend/app/main.py` (rating endpoints)

```python
from app.dependencies import get_current_user
from app.models.user import User

@app.post("/courses/{course_id}/ratings", response_model=RatingResponse, status_code=201)
def create_rating(
    course_id: int,
    rating_data: RatingRequest,
    current_user: User = Depends(get_current_user),
    service: CourseService = Depends(get_course_service)
):
    # Use current_user.id instead of rating_data.user_id
    rating = service.add_course_rating(course_id, current_user.id, rating_data.rating)
    return rating
```

**Modify:** `Backend/app/schemas/rating.py`

```python
class RatingRequest(BaseModel):
    rating: int = Field(ge=1, le=5)  # Removed user_id
```

**Modify:** `Backend/app/models/course_rating.py`

```python
class CourseRating(BaseModel):
    # ... existing code ...
    user_id = Column(Integer, ForeignKey("users.id"), nullable=False, index=True)

    # Add relationship
    user = relationship("User", back_populates="ratings")
```

**Create Migration:**

```bash
docker compose exec api uv run alembic -c app/alembic.ini revision --autogenerate -m "change_user_id_back_to_int_with_fk"
```

**Testing:**
- Update all rating tests to include auth header
- Test 401 Unauthorized when no token
- Test 403 Forbidden when modifying another user's rating

---

### 2.3 Frontend - Auth Context (1 day)

**Replace:** `Frontend/src/contexts/UserContext.tsx`

```typescript
'use client';

import { createContext, useContext, useEffect, useState, ReactNode } from 'react';

interface User {
  id: number;
  email: string;
  username: string;
  full_name: string | null;
}

interface UserContextType {
  user: User | null;
  userId: number | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  login: (email: string, password: string) => Promise<void>;
  register: (email: string, username: string, password: string, fullName?: string) => Promise<void>;
  logout: () => void;
}

const UserContext = createContext<UserContextType | null>(null);

export function UserProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('platziflix_token');
    if (token) {
      fetchCurrentUser(token);
    } else {
      setIsLoading(false);
    }
  }, []);

  const fetchCurrentUser = async (token: string) => {
    try {
      const response = await fetch('http://localhost:8000/auth/me', {
        headers: { Authorization: `Bearer ${token}` }
      });
      if (response.ok) {
        const userData = await response.json();
        setUser(userData);
      } else {
        localStorage.removeItem('platziflix_token');
      }
    } catch (error) {
      console.error('Failed to fetch user:', error);
      localStorage.removeItem('platziflix_token');
    } finally {
      setIsLoading(false);
    }
  };

  const login = async (email: string, password: string) => {
    const response = await fetch('http://localhost:8000/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password })
    });

    if (!response.ok) {
      const error = await response.json();
      throw new Error(error.detail || 'Login failed');
    }

    const { access_token, user: userData } = await response.json();
    localStorage.setItem('platziflix_token', access_token);
    setUser(userData);
  };

  const register = async (
    email: string,
    username: string,
    password: string,
    fullName?: string
  ) => {
    const response = await fetch('http://localhost:8000/auth/register', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, username, password, full_name: fullName })
    });

    if (!response.ok) {
      const error = await response.json();
      throw new Error(error.detail || 'Registration failed');
    }

    const { access_token, user: userData } = await response.json();
    localStorage.setItem('platziflix_token', access_token);
    setUser(userData);
  };

  const logout = () => {
    localStorage.removeItem('platziflix_token');
    setUser(null);
  };

  return (
    <UserContext.Provider value={{
      user,
      userId: user?.id ?? null,
      isAuthenticated: user !== null,
      isLoading,
      login,
      register,
      logout
    }}>
      {children}
    </UserContext.Provider>
  );
}

export const useUser = () => {
  const context = useContext(UserContext);
  if (!context) throw new Error('useUser must be used within UserProvider');
  return context;
};
```

**Modify:** `Frontend/src/services/ratingsApi.ts`

```typescript
const getAuthHeader = (): HeadersInit => {
  const token = localStorage.getItem('platziflix_token');
  return token ? { Authorization: `Bearer ${token}` } : {};
};

// Update all methods to include auth header and remove user_id from body
export const ratingsApi = {
  async createRating(courseId: number, rating: number): Promise<CourseRating> {
    const response = await fetchWithTimeout(
      `${BASE_URL}/courses/${courseId}/ratings`,
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          ...getAuthHeader()
        },
        body: JSON.stringify({ rating })  // No user_id
      }
    );
    return handleApiResponse<CourseRating>(response);
  },

  // Update other methods similarly...
};
```

**Create:** `Frontend/src/components/Auth/LoginForm.tsx`

```typescript
'use client';

import { useState } from 'react';
import { useUser } from '@/contexts/UserContext';
import { useRouter } from 'next/navigation';
import styles from './LoginForm.module.scss';

export default function LoginForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  const { login } = useUser();
  const router = useRouter();

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setError('');
    setIsLoading(true);

    try {
      await login(email, password);
      router.push('/');
    } catch (err) {
      setError((err as Error).message);
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} className={styles.form}>
      <h2>Log In</h2>

      {error && <div className={styles.error}>{error}</div>}

      <div className={styles.field}>
        <label htmlFor="email">Email</label>
        <input
          id="email"
          type="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          required
        />
      </div>

      <div className={styles.field}>
        <label htmlFor="password">Password</label>
        <input
          id="password"
          type="password"
          value={password}
          onChange={(e) => setPassword(e.target.value)}
          required
        />
      </div>

      <button type="submit" disabled={isLoading}>
        {isLoading ? 'Logging in...' : 'Log In'}
      </button>
    </form>
  );
}
```

**Create:** `Frontend/src/components/Auth/RegisterForm.tsx` (similar structure)

**Create:** `Frontend/src/app/login/page.tsx`

```typescript
import LoginForm from '@/components/Auth/LoginForm';
import styles from './page.module.scss';

export default function LoginPage() {
  return (
    <div className={styles.container}>
      <LoginForm />
    </div>
  );
}
```

**Create:** `Frontend/src/app/register/page.tsx` (similar structure)

**Modify:** `Frontend/src/components/CourseDetail/RatingSection.tsx`

```typescript
const { userId, isAuthenticated } = useUser();

if (!isAuthenticated) {
  return (
    <section className={styles.section}>
      <h2>Rate this course</h2>
      <p>
        Please <Link href="/login">log in</Link> to rate this course.
      </p>
      {/* Still show overall rating stats */}
      <div className={styles.overallRating}>
        {/* ... stats display ... */}
      </div>
    </section>
  );
}
```

---

### Phase 2 Deliverables

**Working Features:**
- ✅ User registration with email/password
- ✅ JWT-based authentication
- ✅ Protected rating endpoints
- ✅ Login/logout functionality
- ✅ Token persistence and auto-login
- ✅ Ratings tied to user accounts

**Files Created (Backend - 9 files):**
1. `Backend/app/models/user.py`
2. `Backend/app/schemas/user.py`
3. `Backend/app/core/security.py`
4. `Backend/app/services/auth_service.py`
5. `Backend/app/dependencies.py`
6. `Backend/app/routers/auth.py`
7. `Backend/app/alembic/versions/XXXXX_add_users_table.py`
8. `Backend/app/tests/test_auth_service.py`
9. `Backend/app/tests/test_auth_endpoints.py`

**Files Created (Frontend - 6 files):**
1. `Frontend/src/components/Auth/LoginForm.tsx`
2. `Frontend/src/components/Auth/LoginForm.module.scss`
3. `Frontend/src/components/Auth/RegisterForm.tsx`
4. `Frontend/src/components/Auth/RegisterForm.module.scss`
5. `Frontend/src/app/login/page.tsx`
6. `Frontend/src/app/register/page.tsx`

**Files Modified:**
1. `Frontend/src/contexts/UserContext.tsx` (complete rewrite)
2. `Frontend/src/services/ratingsApi.ts` (add auth headers)
3. `Frontend/src/components/CourseDetail/RatingSection.tsx` (auth check)
4. `Backend/app/main.py` (protect endpoints)
5. `Backend/app/schemas/rating.py` (remove user_id from request)
6. `Backend/app/models/course_rating.py` (FK to users)

**Test Coverage:**
- 15+ new backend auth tests
- 10+ new frontend auth tests
- All existing tests updated

**Timeline:** 5-7 days

---

## Phase 3: Polish & Advanced Features

**Goal:** Enhance UX, optimize performance, add power user features.

**Duration:** 2-3 days

### 3.1 Optimistic UI Updates (4-6 hours)

**Modify:** `Frontend/src/components/CourseDetail/RatingSection.tsx`

Add optimistic updates for instant feedback:

```typescript
const handleRate = async (rating: number) => {
  const previousRating = userRating;
  const previousStats = stats;

  // Optimistic update
  setUserRating(rating);
  const optimisticStats = calculateOptimisticStats(rating, previousRating, previousStats);
  setStats(optimisticStats);

  try {
    if (previousRating === null) {
      await ratingsApi.createRating(courseId, rating);
    } else {
      await ratingsApi.updateRating(courseId, userId!, rating);
    }

    // Sync with server
    const realStats = await ratingsApi.getRatingStats(courseId);
    setStats(realStats);
  } catch (err) {
    // Rollback on error
    setUserRating(previousRating);
    setStats(previousStats);
    setError('Failed to submit rating');
  }
};

function calculateOptimisticStats(
  newRating: number,
  oldRating: number | null,
  currentStats: RatingStats
): RatingStats {
  const newDist = { ...currentStats.rating_distribution };

  if (oldRating !== null) {
    newDist[oldRating] = (newDist[oldRating] || 1) - 1;
  }

  newDist[newRating] = (newDist[newRating] || 0) + 1;

  const newTotal = oldRating === null
    ? currentStats.total_ratings + 1
    : currentStats.total_ratings;

  const sum = Object.entries(newDist).reduce(
    (acc, [stars, count]) => acc + parseInt(stars) * count,
    0
  );
  const newAverage = newTotal > 0 ? sum / newTotal : 0;

  return {
    average_rating: newAverage,
    total_ratings: newTotal,
    rating_distribution: newDist
  };
}
```

---

### 3.2 Rating Management UI (3-4 hours)

**Create:** `Frontend/src/components/CourseDetail/UserRatingCard.tsx`

```typescript
interface UserRatingCardProps {
  rating: number;
  courseId: number;
  onUpdate: (rating: number) => Promise<void>;
  onDelete: () => Promise<void>;
}

export default function UserRatingCard({
  rating,
  courseId,
  onUpdate,
  onDelete
}: UserRatingCardProps) {
  const [isEditing, setIsEditing] = useState(false);
  const [isDeleting, setIsDeleting] = useState(false);

  const handleDelete = async () => {
    if (!confirm('Remove your rating?')) return;

    setIsDeleting(true);
    try {
      await onDelete();
    } finally {
      setIsDeleting(false);
    }
  };

  return (
    <div className={styles.card}>
      {isEditing ? (
        <InteractiveStarRating
          currentRating={rating}
          onRate={async (newRating) => {
            await onUpdate(newRating);
            setIsEditing(false);
          }}
          courseId={courseId}
        />
      ) : (
        <>
          <StarRating rating={rating} size="medium" />
          <div className={styles.actions}>
            <button onClick={() => setIsEditing(true)}>Edit</button>
            <button onClick={handleDelete} disabled={isDeleting}>
              {isDeleting ? 'Deleting...' : 'Delete'}
            </button>
          </div>
        </>
      )}
    </div>
  );
}
```

---

### 3.3 Performance Optimizations (2-3 hours)

**Backend - Add Database Index:**

```python
# In Backend/app/models/course_rating.py
from sqlalchemy import Index

__table_args__ = (
    UniqueConstraint(...),
    Index('ix_course_ratings_course_deleted', 'course_id', 'deleted_at'),
    {"extend_existing": True}
)
```

**Frontend - Add Loading Skeletons:**

```typescript
if (isLoading) {
  return (
    <div className={styles.skeleton}>
      <div className={styles.skeletonTitle} />
      <div className={styles.skeletonStars} />
      <div className={styles.skeletonBars} />
    </div>
  );
}
```

---

### Phase 3 Deliverables

**Working Features:**
- ✅ Optimistic UI updates
- ✅ Edit/delete rating UI
- ✅ Loading skeletons
- ✅ Performance optimizations

**Files Created:**
1. `Frontend/src/components/CourseDetail/UserRatingCard.tsx`
2. `Frontend/src/components/CourseDetail/UserRatingCard.module.scss`

**Files Modified:**
1. `Frontend/src/components/CourseDetail/RatingSection.tsx`
2. `Backend/app/models/course_rating.py`

**Test Coverage:**
- 5+ new tests for optimistic updates
- 3+ tests for edit/delete UI

**Timeline:** 2-3 days

---

## Risk Analysis & Mitigation

### Critical Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| **Data loss in migration** | High | Low | Backup DB, test on staging, rollback plan |
| **Race conditions** | Medium | Medium | DB constraints + frontend debounce |
| **Token expiration UX** | Medium | High | Auto-refresh, clear error messages |
| **Guest → User migration** | High | High | Document limitation, manual migration tool |
| **Rating spam** | High | Medium | Rate limiting + email verification |

### Technical Risks

**SQL Injection:**
- **Mitigation:** SQLAlchemy ORM prevents injection, Pydantic validates inputs

**XSS Attacks:**
- **Mitigation:** Next.js auto-escapes JSX, no dangerouslySetInnerHTML

**N+1 Query Problem:**
- **Mitigation:** Eager loading with joinedload, computed columns

---

## Testing Strategy

### Unit Tests (Target: 80% coverage)

**Backend:**
- User model, auth service, security functions
- All rating service methods
- JWT creation/validation

**Frontend:**
- Auth context (login/logout/register)
- Interactive rating component
- Rating section with API integration

### Integration Tests

**Backend:**
- Auth endpoints (register, login, /me)
- Protected rating endpoints (401/403 scenarios)

**Frontend:**
- API client with auth headers
- Token refresh logic

### E2E Tests (Playwright)

1. Register → Rate → Logout → Login → See saved rating
2. Rate → Edit → Delete
3. Network error handling
4. Token expiration flow
5. Mobile responsive
6. Accessibility (keyboard navigation)

---

## Timeline Summary

| Phase | Duration | Working Days | Parallel Work Possible |
|-------|----------|--------------|------------------------|
| **Phase 1: MVP** | 14-18 hours | 3-4 days | Backend migration can run parallel |
| **Phase 2: Auth** | 24-32 hours | 5-7 days | Frontend/Backend can be parallel |
| **Phase 3: Polish** | 15-20 hours | 2-3 days | Most tasks can be parallel |
| **Total** | **53-70 hours** | **10-14 days** | With 1-2 developers |

**Critical Path:**
```
Day 1-4:   Phase 1 (MVP working)
Day 5-11:  Phase 2 (Auth complete)
Day 12-14: Phase 3 (Polish)
```

---

## Success Metrics

### Phase 1 Success Criteria
- [ ] Users can rate courses (1-5 stars)
- [ ] Ratings persist to database
- [ ] Stats update in real-time
- [ ] Distribution chart visible
- [ ] All tests passing
- [ ] Page load < 2s

### Phase 2 Success Criteria
- [ ] Users can register/login
- [ ] Ratings tied to user accounts
- [ ] Protected endpoints return 401 without token
- [ ] Token persists across reloads
- [ ] Zero security vulnerabilities
- [ ] All tests passing

### Phase 3 Success Criteria
- [ ] Optimistic updates < 100ms perceived
- [ ] Edit/delete working
- [ ] Lighthouse score > 90
- [ ] All E2E tests passing
- [ ] Positive user feedback

---

## Documentation Updates

### Required Documentation

1. **Update CLAUDE.md**
   - Add auth system overview
   - Document rating endpoints with auth
   - Update architecture diagrams

2. **Create Backend/docs/API.md**
   - Document all auth endpoints
   - Include curl examples with JWT
   - Rate limiting info

3. **Create Frontend/README_RATING.md**
   - How to use rating components
   - Auth context usage
   - Error handling patterns

4. **Update Database Schema Diagram**
   - Include users table
   - Show FK relationships
   - Document indexes

---

## Alternative Approaches

### Alternative 1: Auth-First Strategy

**Timeline:** Same 10-14 days, different order

**Pros:**
- No guest user migration
- Cleaner architecture from start

**Cons:**
- Slower to first working demo
- Can't validate UX until auth done

### Alternative 2: OAuth-Only

**Replace email/password with Google/GitHub OAuth**

**Pros:**
- No password storage
- Faster registration

**Cons:**
- Third-party dependency
- Complex setup

**Recommendation:** Add in Phase 4, not replacement

---

## Next Steps

### Immediate Actions
1. Review plan with stakeholders
2. Decide: MVP-First vs Auth-First
3. Set up feature branch: `feature/rating-system`
4. Create project tracking (GitHub Projects)

### Week 1: Phase 1
1. Implement guest user context
2. Build InteractiveStarRating
3. Integrate RatingSection
4. Test end-to-end

### Week 2: Phase 2
1. Build backend auth
2. Implement frontend auth
3. Protect endpoints
4. Integration testing

### Week 3: Phase 3 + Launch
1. Optimistic updates
2. Rating management
3. Performance optimization
4. Final testing
5. Deploy

---

## Questions for Stakeholders

Before implementation:

1. **MVP viability:** Accept guest users for Phase 1?
2. **Auth requirements:** Email/password only or OAuth too?
3. **Moderation:** Need admin tools for ratings?
4. **User profiles:** Future rating history pages?
5. **Rating comments:** Future text reviews?
6. **Anonymous viewing:** Can logged-out users see ratings?
7. **Migration plan:** How to handle guest → registered users?
8. **Timeline:** 10-14 days acceptable or need faster?

---

## Appendix: File Changes Summary

### Phase 1: Files to Create (9)
1. `Frontend/src/contexts/UserContext.tsx`
2. `Frontend/src/contexts/__tests__/UserContext.test.tsx`
3. `Frontend/src/components/StarRating/InteractiveStarRating.tsx`
4. `Frontend/src/components/StarRating/InteractiveStarRating.module.scss`
5. `Frontend/src/components/StarRating/__tests__/InteractiveStarRating.test.tsx`
6. `Frontend/src/components/CourseDetail/RatingSection.tsx`
7. `Frontend/src/components/CourseDetail/RatingSection.module.scss`
8. `Frontend/src/components/CourseDetail/__tests__/RatingSection.test.tsx`
9. `Backend/app/alembic/versions/XXXXX_change_user_id_to_string.py`

### Phase 1: Files to Modify (5)
1. `Frontend/src/app/layout.tsx`
2. `Frontend/src/app/course/[slug]/page.tsx`
3. `Backend/app/models/course_rating.py`
4. `Backend/app/schemas/rating.py`
5. `Backend/app/tests/test_rating_endpoints.py`

### Phase 2: Files to Create (15)
**Backend (9):**
1. `Backend/app/models/user.py`
2. `Backend/app/schemas/user.py`
3. `Backend/app/core/security.py`
4. `Backend/app/services/auth_service.py`
5. `Backend/app/dependencies.py`
6. `Backend/app/routers/auth.py`
7. `Backend/app/alembic/versions/XXXXX_add_users.py`
8. `Backend/app/tests/test_auth_service.py`
9. `Backend/app/tests/test_auth_endpoints.py`

**Frontend (6):**
1. `Frontend/src/components/Auth/LoginForm.tsx`
2. `Frontend/src/components/Auth/LoginForm.module.scss`
3. `Frontend/src/components/Auth/RegisterForm.tsx`
4. `Frontend/src/components/Auth/RegisterForm.module.scss`
5. `Frontend/src/app/login/page.tsx`
6. `Frontend/src/app/register/page.tsx`

### Phase 2: Files to Modify (6)
1. `Frontend/src/contexts/UserContext.tsx`
2. `Frontend/src/services/ratingsApi.ts`
3. `Frontend/src/components/CourseDetail/RatingSection.tsx`
4. `Backend/app/main.py`
5. `Backend/app/schemas/rating.py`
6. `Backend/app/models/course_rating.py`

### Phase 3: Files to Create (2)
1. `Frontend/src/components/CourseDetail/UserRatingCard.tsx`
2. `Frontend/src/components/CourseDetail/UserRatingCard.module.scss`

### Phase 3: Files to Modify (2)
1. `Frontend/src/components/CourseDetail/RatingSection.tsx`
2. `Backend/app/models/course_rating.py`

---

**Total Implementation:**
- **26 new files**
- **13 modified files**
- **~3200 lines of code**
- **~1000 lines of tests**
- **10-14 days**

---

## Conclusion

This implementation plan provides a pragmatic, phased approach to completing the Platziflix rating system. By starting with an MVP (Phase 1), we can validate the UX and deliver value quickly before investing in authentication infrastructure (Phase 2). Phase 3 ensures a production-ready experience.

**Recommended Starting Point:** Begin with Phase 1 to ship a working rating system in 3-4 days, then iterate based on user feedback.

**Ready to implement?** Start with Phase 1, Task 1.1 (UserContext).
