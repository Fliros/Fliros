# Development Environment Setup - Part 1: Basic Tools

1. Visual Studio Code Installation:

   - Download from: https://code.visualstudio.com/
   - Install recommended extensions:
     a. Python (Microsoft)
     b. Python Indent
     c. ESLint
     d. Prettier
     e. Docker
     f. GitLens
     g. REST Client
     h. PostgreSQL
     i. React/Redux/React-Native snippets

   VS Code Settings (settings.json):

   ```json
   {
     "editor.formatOnSave": true,
     "editor.defaultFormatter": "esbenp.prettier-vscode",
     "python.formatting.provider": "black",
     "python.linting.enabled": true,
     "python.linting.flake8Enabled": true
   }
   ```

2. Git and GitHub Desktop Installation:

   - Install Git:

     - Windows: Download from https://git-scm.com/download/windows
     - macOS: `brew install git`
     - Linux: `sudo apt-get install git`

   - Configure Git:

   ```bash
   git config --global user.name "Your Name"
   git config --global user.email "your.email@example.com"
   ```

   - Install GitHub Desktop from: https://desktop.github.com/

3. Docker Installation:

   - Windows/macOS: Install Docker Desktop from https://www.docker.com/products/docker-desktop
   - Linux:

   ```bash
   sudo apt-get update
   sudo apt-get install docker-ce docker-ce-cli containerd.io
   sudo systemctl start docker
   sudo systemctl enable docker
   sudo usermod -aG docker $USER
   ```

   Test Docker installation:

   ```bash
   docker --version
   docker run hello-world
   ```

# Development Environment Setup - Part 2: Database and Python

4. PostgreSQL Setup:

A. Local Installation:

- Windows: Download from https://www.postgresql.org/download/windows/
- macOS: `brew install postgresql postgis`
- Linux:

```bash
sudo apt-get update
sudo apt-get install postgresql postgresql-contrib postgis
```

B. Create PostgreSQL Docker Container:

```bash
# Pull PostgreSQL with PostGIS image
docker pull postgis/postgis

# Create a Docker volume for persistent data
docker volume create pgdata

# Run PostgreSQL container
docker run --name gpx-postgres \\
  -e POSTGRES_PASSWORD=yourpassword \\
  -e POSTGRES_DB=gpx_tracker \\
  -p 5432:5432 \\
  -v pgdata:/var/lib/postgresql/data \\
  -d postgis/postgis
```

5. Python Environment Setup:

A. Install Python 3.9 or later:

- Windows: Download from https://www.python.org/downloads/
- macOS: `brew install python@3.9`
- Linux: `sudo apt-get install python3.9`

B. Install Poetry (Python dependency management):

osx / linux / bashonwindows / Windows+MinGW install instructions

```bash
 curl -sSL https://install.python-poetry.org | python3 -
```

    windows powershell install instructions

```bash
 (Invoke-WebRequest -Uri https://install.python-poetry.org -UseBasicParsing).Content | py -
```

C. Create Python virtual environment:

```bash
# Create project directory
mkdir gpx-tracker
cd gpx-tracker

# Initialize Poetry project
poetry init --name gpx-tracker --description "GPX Track Analysis Application" --author "Your Name <your.email@example.com>" --python "^3.11"

# Add essential dependencies
poetry add fastapi uvicorn sqlalchemy psycopg2-binary geoalchemy2 gpxpy celery redis

# Add development dependencies
poetry add --dev pytest black flake8 mypy pytest-cov
```

# Development Environment Setup - Part 3: Frontend and Project Structure

6. Node.js and Frontend Setup:

A. Install Node.js:

- Download LTS version from https://nodejs.org/
- Or using package manager:
  - macOS: `brew install node`
  - Linux:
  ```bash
  curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
  sudo apt-get install -y nodejs
  ```

1. Create Main Project Directory:

   ```bash
   mkdir gpx-tracker
   cd gpx-tracker

   # Create the basic structure
   mkdir -p backend/app/{api,models,schemas,services,tests}
   mkdir -p frontend/src/{components,pages,services,hooks,utils}
   mkdir -p docker/{postgres,backend,frontend}
   ```

2. Initialize Frontend:

   ```bash
   # Move into frontend directory
   cd frontend

   # Initialize Vite project in the current directory
   npm create vite@latest . -- --template react-ts

   # Install dependencies
   npm install
   npm install leaflet @types/leaflet axios react-query tailwindcss postcss autoprefixer
   npm install --save-dev jest @testing-library/react @testing-library/jest-dom
   ```

3. Initialize Backend:

   ```bash
   # Move to backend directory
   cd ../backend

   # Initialize Poetry project
   poetry init
   ```

This approach ensures:

1. A clean monorepo structure
2. Proper isolation between frontend and backend
3. Correct Docker configuration paths
4. Easier dependency management",

5. Initialize Git Repository:

   ```bash
   # From the root directory (gpx-tracker)
   git init

   # Create root-level .gitignore
   cat > .gitignore << EOL
   # Dependencies
   node_modules/
   .pnp/
   .pnp.js

   # Python
   __pycache__/
   *.py[cod]
   .env
   .venv
   venv/

   # Production
   build/
   dist/

   # Testing
   coverage/
   .pytest_cache/

   # IDE
   .idea/
   .vscode/
   *.swp
   *.swo

   # Misc
   .DS_Store
   .env.local
   .env.development.local
   .env.test.local
   .env.production.local

   # Docker
   .docker/
   pgdata/
   EOL

   ```

6. Create Root-Level Configuration:
   A. Create .env.example:

   ```env
   # PostgreSQL
   POSTGRES_USER=postgres
   POSTGRES_PASSWORD=yourpassword
   POSTGRES_DB=gpx_tracker

   # Backend
   DATABASE_URL=postgresql://postgres:yourpassword@postgres:5432/gpx_tracker
   REDIS_URL=redis://redis:6379/0

   # Frontend
   VITE_API_URL=http://localhost:8000
   ```

   B. Create README.md:

   ````markdown
   # GPX Tracker

   A web application for tracking and visualizing GPX routes.

   ## Project Structure

   - `backend/`: FastAPI application
   - `frontend/`: React application
   - `docker/`: Docker configuration files

   ## Development Setup

   1. Clone the repository
   2. Copy .env.example to .env and adjust values
   3. Start the development environment:
      ```bash
      docker-compose up -d
      ```
   ````

   ## Available Scripts

   Backend:

   ```bash
   cd backend
   poetry install  # Install dependencies
   poetry run pytest  # Run tests
   ```

   Frontend:

   ```bash
   cd frontend
   npm install  # Install dependencies
   npm run dev  # Start development server
   ```

   ```

   ```

7. Initial Commit:
   ```bash
   git add .
   git commit -m \"Initial project setup\"
   ```

This structure provides a solid foundation for the project with:

1. Clear separation of concerns
2. Proper version control setup
3. Environment variable management
4. Comprehensive documentation

You now have a clean, well-organized project structure ready for development. The next steps would be implementing the specific components within each directory.

---

7. Project Structure Setup:

```
gpx-tracker/
├── backend/
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py
│   │   ├── models/
│   │   ├── schemas/
│   │   ├── services/
│   │   └── api/
│   ├── tests/
│   ├── alembic/
│   ├── pyproject.toml
│   └── poetry.lock
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   ├── pages/
│   │   ├── services/
│   │   ├── hooks/
│   │   └── utils/
│   ├── package.json
│   └── tsconfig.json
├── docker/
│   ├── postgres/
│   ├── backend/
│   └── frontend/
└── docker-compose.yml
```

8. Docker Compose Configuration:

Create docker-compose.yml:

```yaml
version: "3.8"

services:
  postgres:
    image: postgis/postgis
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: yourpassword
      POSTGRES_DB: gpx_tracker
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

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
      DATABASE_URL: postgresql://postgres:yourpassword@postgres:5432/gpx_tracker
      REDIS_URL: redis://redis:6379/0

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
  pgdata:
```

# Development Environment Setup - Part 4: Verification and Configuration",

9. Environment Verification:  
   Run these commands to verify your setup:

```bash
   # Check versions
   python --version  # Should be 3.9 or later
   node --version    # Should be 14 or later
   docker --version
   poetry --version
   psql --version

   # Test PostgreSQL connection
   docker exec -it gpx-postgres psql -U postgres -d gpx_tracker -c "\\dt"

   # Start the development environment
   docker-compose up -d
```

10. Environment Variables Setup:

    Create .env files for both frontend and backend:

    backend/.env:

    ```env
    DATABASE_URL=postgresql://postgres:yourpassword@localhost:5432/gpx_tracker
    REDIS_URL=redis://localhost:6379/0
    SECRET_KEY=your-secret-key-here
    DEBUG=true
    ALLOWED_ORIGINS=http://localhost:3000
    ```

    frontend/.env:

    ```env
    VITE_API_URL=http://localhost:8000
    VITE_MAPBOX_TOKEN=your-mapbox-token-here
    ```

11. VS Code Workspace Settings:

    Create .vscode/settings.json:

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

12. Common Troubleshooting:

    - Docker Issues:

      - Permission denied: `sudo usermod -aG docker $USER` and restart
      - Port conflicts: Check if ports 5432, 6379, 8000, or 3000 are in use

    - PostgreSQL Issues:

      - Connection refused: Ensure PostgreSQL service is running
      - Password authentication failed: Check credentials in .env file

    - Python/Poetry Issues:

      - Command not found: Add Poetry to PATH
      - Dependencies conflicts: Delete poetry.lock and reinstall

    - Node.js Issues:
      - EACCES error: Use nvm or fix npm permissions
      - Module not found: Run `npm install` in frontend directory",
        "next_action": "final_answer",
        "confidence": 0.95

# Final Summary and Next Steps",

You now have a complete development environment setup for the GPX tracking application. Here's a quick checklist to verify everything is ready:

1. Essential Tools Installed:
   ✓ VS Code with extensions
   ✓ Git and GitHub Desktop
   ✓ Docker and Docker Compose
   ✓ PostgreSQL with PostGIS
   ✓ Python 3.9+ with Poetry
   ✓ Node.js with npm

2. Project Structure Created:
   ✓ Backend directory with FastAPI setup
   ✓ Frontend directory with React setup
   ✓ Docker configuration files
   ✓ Development environment configuration

3. Development Environment Ready:
   ✓ Database container running
   ✓ Redis container running
   ✓ Environment variables configured
   ✓ VS Code workspace settings optimized

Next Steps:

1. Clone your GitHub repository and set up version control
2. Start the development servers:

   ```bash
   # Start all services
   docker-compose up -d

   # Check logs
   docker-compose logs -f
   ```

3. Access your applications:
   - Backend API: http://localhost:8000
   - Frontend: http://localhost:3000
   - API Documentation: http://localhost:8000/docs

You're now ready to begin development of the GPX tracking application. The next phase will involve setting up the database schema and implementing the basic backend API endpoints.

If you encounter any issues or need to modify the configuration, refer to the troubleshooting section or the official documentation of the respective tools.
