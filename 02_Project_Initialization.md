# Project Initialization Guide

### Table of Contents

- [Part 1: GitHub Repository Setup](#part-1-github-repository-setup)
  - [Create GitHub Repository](#create-github-repository)
  - [Clone and Set Up Basic Structure](#clone-and-set-up-basic-structure)
  - [Initial Configuration Files](#initial-configuration-files)
- [Part 2: Backend Setup](#part-2-backend-setup)
  - [Initialize Backend Project](#initialize-backend-project)
  - [Backend Configuration Files](#backend-configuration-files)
- [Part 3: Frontend Setup](#part-3-frontend-setup)
  - [Initialize Frontend Project](#initialize-frontend-project)
  - [Configure Frontend](#configure-frontend)
- [Part 4: Docker and Documentation Setup](#part-4-docker-and-documentation-setup)
  - [Docker Configuration](#docker-configuration)
  - [Project Documentation](#project-documentation)
- [Part 5: Development Scripts and Testing Setup](#part-5-development-scripts-and-testing-setup)
  - [Development Scripts](#development-scripts)
  - [Initial Tests Setup](#initial-tests-setup)
  - [Git Hooks Setup](#git-hooks-setup)
- [IDE Configuration](#ide-configuration)
  - [VS Code Workspace Settings](#vs-code-workspace-settings)
- [Final Verification and Next Steps](#final-verification-and-next-steps)
  - [Verification Checklist](#verification-checklist)
  - [Common Issues and Solutions](#common-issues-and-solutions)
  - [Next Steps](#next-steps)

## Part 1: GitHub Repository Setup

### Create GitHub Repository

A. Using GitHub Desktop:

- Open GitHub Desktop
- Click File -> New Repository
- Fill in the following:
  - Name: gpx-tracker
  - Description: A web application for tracking and visualizing GPX routes
  - Local Path: Choose your preferred location
  - Initialize with README: Yes
  - Git ignore: Choose Python
  - License: MIT
- Click 'Create Repository'

B. Or using GitHub CLI:

```bash
# Install GitHub CLI if not installed
# macOS: brew install gh
# Linux: sudo apt install gh

# Login to GitHub
gh auth login

# Create repository
gh repo create gpx-tracker \
  --public \
  --description "A web application for tracking and visualizing GPX routes" \
  --license mit \
  --gitignore Python \
  --clone
```

### Clone and Set Up Basic Structure

```bash
# If you didn't clone during creation
git clone https://github.com/yourusername/gpx-tracker.git
cd gpx-tracker

# Create basic folder structure
mkdir -p backend/app/{api,core,db,models,schemas,services,tests}
mkdir -p frontend/src/{components,pages,services,hooks,utils,assets,styles}
mkdir -p docker/{backend,frontend,postgres}
```

### Initial Configuration Files

A. Root .gitignore:

```gitignore
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
env/
build/
develop-eggs/
dist/
downloads/
eggs/
.eggs/
lib/
lib64/
parts/
sdist/
var/
*.egg-info/
.installed.cfg
*.egg

# Node
node_modules/
npm-debug.log
yarn-debug.log
yarn-error.log
.env
.env.local
.env.development.local
.env.test.local
.env.production.local

# IDEs
.idea/
.vscode/
*.swp
*.swo

# Docker
docker-compose.override.yml

# Project specific
.coverage
coverage/
.pytest_cache/
```

## Part 2: Backend Setup

### Initialize Backend Project

```bash
cd backend

# Initialize Poetry project
poetry init \
  --name gpx-tracker-backend \
  --description "Backend for GPX tracking application" \
  --author "Your Name <your.email@example.com>" \
  --python "^3.9" \
  --dependency fastapi \
  --dependency uvicorn \
  --dependency sqlalchemy \
  --dependency alembic \
  --dependency psycopg2-binary \
  --dependency geoalchemy2 \
  --dependency python-multipart \
  --dependency python-jose[cryptography] \
  --dependency passlib[bcrypt] \
  --dependency gpxpy \
  --dependency celery \
  --dependency redis \
  --dependency shapely \
  --dev-dependency pytest \
  --dev-dependency pytest-cov \
  --dev-dependency black \
  --dev-dependency flake8 \
  --dev-dependency mypy \
  --no-interaction

# Install dependencies
poetry install
```

### Backend Configuration Files

A. backend/app/core/config.py:

```python
from pydantic_settings import BaseSettings
from typing import List, Optional

class Settings(BaseSettings):
    PROJECT_NAME: str = "GPX Tracker"
    VERSION: str = "1.0.0"
    API_V1_STR: str = "/api/v1"

    POSTGRES_SERVER: str = "localhost"
    POSTGRES_USER: str = "postgres"
    POSTGRES_PASSWORD: str = "password"
    POSTGRES_DB: str = "gpx_tracker"
    POSTGRES_PORT: str = "5432"

    DATABASE_URL: Optional[str] = None

    @property
    def SQLALCHEMY_DATABASE_URI(self) -> str:
        if self.DATABASE_URL:
            return self.DATABASE_URL
        return f"postgresql://{self.POSTGRES_USER}:{self.POSTGRES_PASSWORD}@{self.POSTGRES_SERVER}:{self.POSTGRES_PORT}/{self.POSTGRES_DB}"

    REDIS_URL: str = "redis://localhost:6379/0"

    # Security
    SECRET_KEY: str = "your-secret-key-here"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 60 * 24 * 8  # 8 days

    # CORS
    BACKEND_CORS_ORIGINS: List[str] = ["http://localhost:3000"]

    class Config:
        case_sensitive = True
        env_file = ".env"

settings = Settings()
```

B. backend/app/main.py:

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.core.config import settings

app = FastAPI(
    title=settings.PROJECT_NAME,
    version=settings.VERSION,
    openapi_url=f"{settings.API_V1_STR}/openapi.json"
)

# Set up CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.BACKEND_CORS_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/")
def read_root():
    return {"message": "Welcome to GPX Tracker API"}
```

## Part 3: Frontend Setup

### Initialize Frontend Project

```bash
cd ../frontend

# Create new Vite project with React and TypeScript
npm create vite@latest . -- --template react-ts

# Install dependencies
npm install

# Install additional dependencies
npm install @tanstack/react-query \
           axios \
           react-router-dom \
           leaflet \
           @types/leaflet \
           @tailwindcss/forms \
           tailwindcss \
           postcss \
           autoprefixer \
           clsx \
           @headlessui/react \
           zustand \
           date-fns

# Install dev dependencies
npm install -D @testing-library/react \
               @testing-library/jest-dom \
               @testing-library/user-event \
               @types/node \
               prettier \
               eslint-config-prettier \
               vitest
```

### Configure Frontend

A. Create src/config.ts:

```typescript
const config = {
  apiUrl: import.meta.env.VITE_API_URL || "http://localhost:8000",
  apiVersion: "v1",
  mapboxToken: import.meta.env.VITE_MAPBOX_TOKEN,
  defaultCenter: { lat: 0, lng: 0 },
  defaultZoom: 2,
};

export default config;
```

B. Create src/services/api.ts:

```typescript
import axios from "axios";
import config from "../config";

const api = axios.create({
  baseURL: `${config.apiUrl}/api/${config.apiVersion}`,
  headers: {
    "Content-Type": "application/json",
  },
});

// Add request interceptor for auth token
api.interceptors.request.use((config) => {
  const token = localStorage.getItem("token");
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export default api;
```

C. Configure Tailwind CSS:

```bash
# Initialize Tailwind CSS
npx tailwindcss init -p
```

Update tailwind.config.js:

```javascript
/** @type {import('tailwindcss').Config} */
export default {
  content: ["./index.html", "./src/**/*.{js,ts,jsx,tsx}"],
  theme: {
    extend: {},
  },
  plugins: [require("@tailwindcss/forms")],
};
```

D. Update src/index.css:

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

## Part 4: Docker and Documentation Setup

### Docker Configuration

A. docker/backend/Dockerfile:

```dockerfile
FROM python:3.9-slim

WORKDIR /app

RUN pip install poetry

COPY pyproject.toml poetry.lock ./
RUN poetry config virtualenvs.create false \
    && poetry install --no-dev --no-interaction

COPY . .

CMD ["poetry", "run", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
```

B. docker/frontend/Dockerfile:

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

CMD ["npm", "run", "dev", "--", "--host", "0.0.0.0"]
```

C. Root docker-compose.yml:

```yaml
version: "3.8"

services:
  postgres:
    image: postgis/postgis
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: gpx_tracker
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:alpine
    ports:
      - "6379:6379"

  backend:
    build:
      context: ./backend
      dockerfile: ../docker/backend/Dockerfile
    volumes:
      - ./backend:/app
    ports:
      - "8000:8000"
    depends_on:
      - postgres
      - redis
    environment:
      - DATABASE_URL=postgresql://postgres:password@postgres:5432/gpx_tracker
      - REDIS_URL=redis://redis:6379/0

  frontend:
    build:
      context: ./frontend
      dockerfile: ../docker/frontend/Dockerfile
    volumes:
      - ./frontend:/app
    ports:
      - "3000:3000"
    depends_on:
      - backend

volumes:
  postgres_data:
```

## IDE Configuration

### VS Code Workspace Settings

Create `.vscode/settings.json` in your project root:

```json
{
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  },
  "python.analysis.typeCheckingMode": "basic",
  "python.formatting.provider": "black",
  "python.linting.enabled": true,
  "python.linting.flake8Enabled": true,
  "[python]": {
    "editor.formatOnSave": true,
    "editor.defaultFormatter": "ms-python.python"
  },
  "[javascript][typescript][javascriptreact][typescriptreact]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "files.exclude": {
    "**/__pycache__": true,
    "**/.pytest_cache": true,
    "**/*.pyc": true
  }
}
```

These settings configure:

- Automatic code formatting on save
- ESLint auto-fix on save
- Python type checking and linting
- Language-specific formatters
- File exclusions for cleaner workspace

## Part 5: Development Scripts and Testing Setup

### Development Scripts

A. Add scripts to backend/pyproject.toml:

```toml
[tool.poetry.scripts]
start = "uvicorn app.main:app --reload"
test = "pytest"
lint = "flake8"
format = "black ."
typecheck = "mypy ."

[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = "test_*.py"
addopts = "--verbosity=2 --cov=app"

[tool.black]
line-length = 88
target-version = ['py39']
include = '\.pyi?$'

[tool.mypy]
python_version = "3.9"
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
```

B. Update frontend/package.json scripts:

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "lint": "eslint . --ext ts,tsx --report-unused-disable-directives --max-warnings 0",
    "preview": "vite preview",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "format": "prettier --write 'src/**/*.{ts,tsx,css}'"
  }
}
```

### Initial Tests Setup

A. backend/tests/conftest.py:

```python
import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.main import app
from app.core.config import settings
from app.db.base import Base

@pytest.fixture(scope="session")
def test_db():
    # Use SQLite for tests
    SQLALCHEMY_TEST_DATABASE_URL = "sqlite:///./test.db"
    engine = create_engine(SQLALCHEMY_TEST_DATABASE_URL)
    TestingSessionLocal = sessionmaker(bind=engine)

    Base.metadata.create_all(bind=engine)
    yield engine
    Base.metadata.drop_all(bind=engine)

@pytest.fixture
def client():
    with TestClient(app) as test_client:
        yield test_client
```

B. frontend/src/tests/setup.ts:

```typescript
import "@testing-library/jest-dom";
import { expect, afterEach } from "vitest";
import { cleanup } from "@testing-library/react";
import matchers from "@testing-library/jest-dom/matchers";

// Extend Vitest's expect method with testing-library methods
expect.extend(matchers);

// Clean up after each test
afterEach(() => {
  cleanup();
});
```

### Git Hooks Setup

```bash
# Install Husky
cd frontend
npm install -D husky lint-staged
npx husky install
npm pkg set scripts.prepare="husky install"

# Add pre-commit hook
npx husky add .husky/pre-commit "npm run lint-staged"
```

Add to frontend/package.json:

```json
{
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix"],
    "*.{ts,tsx,css,md}": ["prettier --write"]
  }
}
```

## Final Verification and Next Steps

### Verification Checklist

A. Repository Structure:

```bash
# Verify project structure
tree -L 2 -I 'node_modules|dist|__pycache__|.git'
```

B. Test Development Environment:

```bash
# Start all services
docker-compose up -d

# Check service status
docker-compose ps

# Verify backend
curl http://localhost:8000/api/v1/

# Verify frontend
curl http://localhost:3000
```

C. Run Tests:

```bash
# Backend tests
cd backend
poetry run pytest

# Frontend tests
cd frontend
npm run test
```

### Common Issues and Solutions

1. Docker:

   - If services won't start: `docker-compose down -v && docker-compose up -d`
   - If ports are in use: Check running services with `docker ps` or `netstat`

2. Dependencies:

   - Backend: `poetry install --no-root`
   - Frontend: Delete node_modules and run `npm install`

3. Database:
   - Connection issues: Check PostgreSQL logs with `docker-compose logs postgres`
   - Reset database: `docker-compose down -v` (warning: destroys all data)

### Next Steps

1. [Database Setup](03_Database_Setup.md):

   - Set up migrations using Alembic
   - Create initial database models

2. Authentication:

   - Implement user authentication system
   - Set up JWT token handling

3. Core Features:

   - GPX file upload and processing
   - Database models for tracks and user data
   - Map visualization components

4. Development Tips:
   - Use VS Code's REST Client extension to test API endpoints
   - Set up browser development tools for React
   - Configure debugging in VS Code for both Python and TypeScript

Remember to:

1. Never commit sensitive information
2. Keep dependencies updated
3. Write tests for new features
4. Document API changes
5. Follow the established code style guidelines
