# Development Environment Setup

- [Visual Studio Code Setup](#visual-studio-code-setup)
  - [Installation](#installation)
  - [Extensions](#extensions)
  - [Configuration](#configuration)
- [Development Tools Installation](#development-tools-installation)
  - [Git](#git)
  - [Docker](#docker)
  - [PostgreSQL](#postgresql)
  - [Python](#python)
  - [Poetry](#poetry)
  - [Node.js](#nodejs)
- [Troubleshooting Guide](#troubleshooting-guide)
  - [Docker Issues](#docker-issues)
  - [PostgreSQL Issues](#postgresql-issues)
  - [Python/Poetry Issues](#pythonpoetry-issues)
  - [Node.js Issues](#nodejs-issues)

## Visual Studio Code Setup

### Installation

Download from: https://code.visualstudio.com/

### Extensions

Install recommended extensions:

- Python (Microsoft)
- Python Indent
- ESLint
- Prettier
- Docker
- GitLens
- REST Client
- PostgreSQL
- React/Redux/React-Native snippets

### Configuration

Add to settings.json:

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "python.formatting.provider": "black",
  "python.linting.enabled": true,
  "python.linting.flake8Enabled": true
}
```

## Development Tools Installation

### Git

- Install GitHub Desktop from: https://desktop.github.com/

- Windows: Download from https://git-scm.com/download/windows
- macOS: `brew install git`
- Linux: `sudo apt-get install git`

### Docker

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

### PostgreSQL

- Windows: Download from https://www.postgresql.org/download/windows/
- macOS: `brew install postgresql postgis`
- Linux:
  ```bash
  sudo apt-get update
  sudo apt-get install postgresql postgresql-contrib postgis
  ```

### Python

- Windows: Download from https://www.python.org/downloads/
- macOS: `brew install python@3.9`
- Linux: `sudo apt-get install python3.9`

### Poetry

Python dependency management installation:

- Unix/macOS/WSL:
  ```bash
  curl -sSL https://install.python-poetry.org | python3 -
  ```
- Windows PowerShell:
  ```powershell
  (Invoke-WebRequest -Uri https://install.python-poetry.org -UseBasicParsing).Content | py -
  ```

### Node.js

- Download LTS version from https://nodejs.org/
- macOS: `brew install node`
- Linux:
  ```bash
  curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
  sudo apt-get install -y nodejs
  ```

## Troubleshooting Guide

### Docker Issues

- Permission denied: `sudo usermod -aG docker $USER` and restart
- Port conflicts: Check if ports 5432, 6379, 8000, or 3000 are in use

### PostgreSQL Issues

- Connection refused: Ensure PostgreSQL service is running
- Password authentication failed: Check credentials in .env file

### Python/Poetry Issues

- Command not found: Add Poetry to PATH
- Dependencies conflicts: Delete poetry.lock and reinstall

### Node.js Issues

- EACCES error: Use nvm or fix npm permissions
- Module not found: Run `npm install` in frontend directory

**Note**: If you encounter any issues or need to modify the configuration, refer to the troubleshooting section or the official documentation of the respective tools.

---

You're now ready to begin development of the GPX tracking application. The next phase will involve setting up the database schema and implementing the basic backend API endpoints.
For detailed project initialization, structure setup, and configuration steps, refer to [Project Initialization](02_Project_Initialization.md).
