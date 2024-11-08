# Project Initialization

# Part 1: GitHub Repository Setup

1. Create GitHub Repository:
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

2. Clone and Set Up Basic Structure:

   ```bash
   # If you didn't clone during creation
   git clone https://github.com/yourusername/gpx-tracker.git
   cd gpx-tracker

   # Create basic folder structure
   mkdir -p backend/app/{api,core,db,models,schemas,services,tests}
   mkdir -p frontend/src/{components,pages,services,hooks,utils,assets,styles}
   mkdir -p docker/{backend,frontend,postgres}
   ```

3. Create Initial Configuration Files:
   A. Root .gitignore:

   ```gitignore # Python   __pycache__/
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

# Part 2: Backend Setup

4. Initialize Backend Project:

   ```bash
   cd backend

   # Initialize Poetry project
   poetry init \\
     --name gpx-tracker-backend \\
     --description "Backend for GPX tracking application" \\
     --author "Your Name <your.email@example.com>" \\
     --python "^3.9" \\
     --dependency fastapi \\
     --dependency uvicorn \\
     --dependency sqlalchemy \\\
     --dependency alembic \\\
     --dependency psycopg2-binary \\\
     --dependency geoalchemy2 \\\
     --dependency python-multipart \\\
     --dependency python-jose[cryptography] \\\
     --dependency passlib[bcrypt] \\\
     --dependency gpxpy \\\
     --dependency celery \\\
     --dependency redis \\\
     --dependency shapely \\\
     --dev-dependency pytest \\\
     --dev-dependency pytest-cov \\\
     --dev-dependency black \\\
     --dev-dependency flake8 \\\
     --dev-dependency mypy \\\
     --no-interaction

   # Install dependencies
   poetry install
   ```

5. Create Backend Configuration Files:
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

# Part 3: Frontend Setup

6. Initialize Frontend Project:

   ```bash
   cd ../frontend

   # Create new Vite project with React and TypeScript
   npm create vite@latest . -- --template react-ts

   # Install dependencies
   npm install

   # Install additional dependencies
   npm install @tanstack/react-query \\
             axios \\
             react-router-dom \\
             leaflet \\
             @types/leaflet \\
             @tailwindcss/forms \\
             tailwindcss \\
             postcss \\\
             autoprefixer \\\
             clsx \\\
             @headlessui/react \\\
             zustand \\\
             date-fns

   # Install dev dependencies
   npm install -D @testing-library/react \\
                  @testing-library/jest-dom \\
                  @testing-library/user-event \\
                  @types/node \\
                  prettier \\
                  eslint-config-prettier \\
                  vitest
   ```

7. Configure Frontend:
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

# Part 4: Docker and Documentation Setup

8.  Create Docker Configuration:
    A. docker/backend/Dockerfile:

```dockerfile
FROM python:3.9-slim

WORKDIR /app

RUN pip install poetry

COPY pyproject.toml poetry.lock ./
RUN poetry config virtualenvs.create false \\
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

9. Create Project Documentation:
   A. Update README.md:

   ````markdown
   # GPX Tracker

   A web application for tracking and visualizing GPX routes.

   ## Prerequisites

   - Docker and Docker Compose
   - Node.js 18+
   - Python 3.9+
   - Poetry

   ## Development Setup

   1. Clone the repository:
      ```bash
      git clone https://github.com/yourusername/gpx-tracker.git
      cd gpx-tracker
      ```
   ````

   2. Set up environment variables:

      ```bash
      # Backend
      cp backend/.env.example backend/.env
      # Frontend
      cp frontend/.env.example frontend/.env
      ```

   3. Start the development environment:

      ```bash
      docker-compose up -d
      ```

   4. Access the applications:
      - Frontend: http://localhost:3000
      - Backend API: http://localhost:8000
      - API Documentation: http://localhost:8000/docs

   ## Development Commands

   Backend:

   ```bash
   cd backend
   poetry install  # Install dependencies
   poetry run pytest  # Run tests
   poetry run black .  # Format code
   poetry run flake8  # Lint code
   ```

   Frontend:

   ```bash
   cd frontend
   npm install  # Install dependencies
   npm run dev  # Start development server
   npm run test  # Run tests
   npm run build  # Build for production
   ```

   ## Project Structure

   ```
   gpx-tracker/
   ├── backend/           # FastAPI application
   │   ├── app/
   │   │   ├── api/      # API endpoints
   │   │   ├── core/     # Core functionality
   │   │   ├── db/       # Database
   │   │   ├── models/   # SQLAlchemy models
   │   │   ├── schemas/  # Pydantic schemas
   │   │   └── services/ # Business logic
   │   └── tests/        # Backend tests
   ├── frontend/         # React application
   │   ├── src/
   │   │   ├── components/
   │   │   ├── pages/
   │   │   ├── services/
   │   │   └── utils/
   │   └── tests/
   └── docker/           # Docker configuration
       ├── backend/      # Backend Dockerfile
       ├── frontend/     # Frontend Dockerfile
       └── postgres/     # PostgreSQL initialization scripts
   ```

   ## Contributing

   1. Create a new branch for your feature:

      ```bash
      git checkout -b feature/your-feature-name
      ```

   2. Make your changes and commit them:

      ```bash
      git add .
      git commit -m "Add your feature description"
      ```

   3. Push to your branch:

      ```bash
      git push origin feature/your-feature-name
      ```

   4. Create a Pull Request on GitHub

   ## Code Style

   - Backend: Follow PEP 8 guidelines
   - Frontend: Follow ESLint and Prettier configuration
   - Use meaningful commit messages
   - Write tests for new features

   ## License

   MIT License - see LICENSE file for details

   ````

   B. Create .env.example files:
   backend/.env.example:
   ```env
   # Database
   DATABASE_URL=postgresql://postgres:password@localhost:5432/gpx_tracker

   # Redis
   REDIS_URL=redis://localhost:6379/0

   # Security
   SECRET_KEY=your-secret-key-here
   ACCESS_TOKEN_EXPIRE_MINUTES=11520

   # CORS
   BACKEND_CORS_ORIGINS=["http://localhost:3000"]

   # Debug
   DEBUG=True
   ````

   frontend/.env.example:

   ```env
   VITE_API_URL=http://localhost:8000
   VITE_MAPBOX_TOKEN=your-mapbox-token-here
   ```

# Part 5: Development Scripts and Testing Setup

10. Set up Development Scripts:
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
    include = '\\.pyi?$'

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

11. Set up Initial Tests:
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

12. Add Git Hooks (using Husky):

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

# Final Verification and Next Steps

13. Verification Checklist:

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

Next Steps:

1. Database Setup:

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

Common Issues and Solutions:

1. Docker:

   - If services won't start: `docker-compose down -v && docker-compose up -d`
   - If ports are in use: Check running services with `docker ps` or `netstat`

2. Dependencies:

   - Backend: `poetry install --no-root`
   - Frontend: Delete node_modules and run `npm install`

3. Database:
   - Connection issues: Check PostgreSQL logs with `docker-compose logs postgres`
   - Reset database: `docker-compose down -v` (warning: destroys all data)

Remember to:

1. Never commit sensitive information
2. Keep dependencies updated
3. Write tests for new features
4. Document API changes
5. Follow the established code style guidelines

# Project Initialization Complete

You now have a fully initialized project with:

1. GitHub repository with proper structure and configuration
2. Backend (FastAPI) setup with:

   - Poetry for dependency management
   - Testing infrastructure
   - Code formatting and linting
   - Basic API structure

3. Frontend (React) setup with:

   - TypeScript configuration
   - Vite build tools
   - Testing framework
   - Code formatting and linting

4. Docker configuration for development
5. Comprehensive documentation
6. Development scripts and tools

You can now start development by:

1. Cloning the repository
2. Installing dependencies
3. Starting the development environment
4. Following the development workflow in the README

The next phase will involve setting up the database schema and implementing the core features of the GPX tracking application.
