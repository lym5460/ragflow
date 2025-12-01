# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

RAGFlow is an open-source RAG (Retrieval-Augmented Generation) engine based on deep document understanding. It's a full-stack application with:
- Python backend (Flask-based API server)
- React/TypeScript frontend (built with UmiJS)
- Microservices architecture with Docker deployment
- Multiple data stores (MySQL, Elasticsearch/Infinity, Redis, MinIO)

## Architecture

### Backend (`/api/`)
- **Main Server**: `api/ragflow_server.py` - Flask application entry point
- **Apps**: Modular Flask blueprints in `api/apps/` for different functionalities:
  - `kb_app.py` - Knowledge base management
  - `dialog_app.py` - Chat/conversation handling
  - `document_app.py` - Document processing
  - `canvas_app.py` - Agent workflow canvas
  - `file_app.py` - File upload/management
- **Services**: Business logic in `api/db/services/`
- **Models**: Database models in `api/db/db_models.py`

### Core Processing (`/rag/`)
- **Document Processing**: `deepdoc/` - PDF parsing, OCR, layout analysis
- **LLM Integration**: `rag/llm/` - Model abstractions for chat, embedding, reranking
- **RAG Pipeline**: `rag/flow/` - Chunking, parsing, tokenization
- **Graph RAG**: `graphrag/` - Knowledge graph construction and querying

### Agent System (`/agent/`)
- **Components**: Modular workflow components (LLM, retrieval, categorize, etc.)
- **Templates**: Pre-built agent workflows in `agent/templates/`
- **Tools**: External API integrations (Tavily, Wikipedia, SQL execution, etc.)

### Frontend (`/web/`)
- React/TypeScript with UmiJS framework
- Ant Design + shadcn/ui components
- State management with Zustand
- Tailwind CSS for styling

## Common Development Commands

### Backend Development
```bash
# Install Python dependencies
uv sync --python 3.10 --all-extras
uv run download_deps.py
pre-commit install

# Start dependent services
docker compose -f docker/docker-compose-base.yml up -d

# Add service name resolution to /etc/hosts
echo "127.0.0.1 es01 infinity mysql minio redis sandbox-executor-manager" | sudo tee -a /etc/hosts

# Install jemalloc if not already installed
# Ubuntu: sudo apt-get install libjemalloc-dev
# CentOS: sudo yum install jemalloc
# macOS: brew install jemalloc

# (Optional) Set HuggingFace mirror if needed
export HF_ENDPOINT=https://hf-mirror.com

# Run backend (requires services to be running)
source .venv/bin/activate
export PYTHONPATH=$(pwd)
bash docker/launch_backend_service.sh

# Stop backend services
pkill -f "ragflow_server.py|task_executor.py"

# Run tests
uv run pytest                    # All tests
uv run pytest -m p1             # High priority tests only
uv run pytest -m "p1 or p2"     # High and medium priority
uv run pytest test/path/test_file.py  # Single test file

# Linting and formatting
ruff check                      # Check code
ruff check --fix                # Auto-fix issues
ruff format                     # Format code
pre-commit run --all-files      # Run all pre-commit hooks
```

### Frontend Development
```bash
cd web
npm install
npm run dev        # Development server
npm run build      # Production build
npm run lint       # ESLint
npm run test       # Jest tests
```

### Docker Development
```bash
# Ensure vm.max_map_count is set (required for Elasticsearch)
sudo sysctl -w vm.max_map_count=262144

# Full stack with Docker
cd docker
docker compose -f docker-compose.yml up -d

# Check server status
docker logs -f ragflow-server

# Stop all containers
docker compose -f docker-compose.yml down

# Rebuild images (run from repository root)
cd ..
docker build --platform linux/amd64 -f Dockerfile -t infiniflow/ragflow:nightly .
```

## Key Configuration Files

- `docker/.env` - Environment variables for Docker deployment
  - `DOC_ENGINE` - Vector database selection (elasticsearch/infinity/oceanbase/opensearch)
  - `DEVICE` - Computation device for DeepDoc (cpu/gpu)
  - `WS` - Number of task executor workers (default: 1)
- `docker/service_conf.yaml.template` - Backend service configuration
- `pyproject.toml` - Python dependencies and project configuration
- `web/package.json` - Frontend dependencies and scripts
- `.pre-commit-config.yaml` - Pre-commit hooks configuration

## Testing

- **Python**: pytest with markers (p1/p2/p3 priority levels)
  - Run all tests: `uv run pytest`
  - Run by priority: `uv run pytest -m p1` (high), `uv run pytest -m p2` (medium), `uv run pytest -m p3` (low)
  - Run specific file: `uv run pytest test/path/to/test_file.py`
  - Run with coverage: `uv run pytest --cov=api --cov=rag`
- **Frontend**: Jest with React Testing Library
  - Run tests: `cd web && npm run test`
  - The test command includes `--no-cache --coverage` by default
- **API Tests**: HTTP API and SDK tests in `test/` and `sdk/python/test/`

## Database Engines

RAGFlow supports multiple vector database engines for storing full text and vectors:
- **Elasticsearch** (default) - Set `DOC_ENGINE=elasticsearch` in `docker/.env`
- **Infinity** - Set `DOC_ENGINE=infinity` in `docker/.env`
- **OceanBase** - Set `DOC_ENGINE=oceanbase` in `docker/.env`
- **OpenSearch** - Set `DOC_ENGINE=opensearch` in `docker/.env`

To switch engines:
1. Stop containers: `docker compose -f docker/docker-compose.yml down -v`
2. Update `DOC_ENGINE` in `docker/.env`
3. Restart: `docker compose -f docker/docker-compose.yml up -d`

**Note**: Using `-v` flag will delete volumes and clear existing data.

## Development Environment Requirements

- Python 3.10-3.12
- Node.js >=18.20.4
- Docker >= 24.0.0 & Docker Compose >= v2.26.1
- uv package manager (install: `pipx install uv`)
- System libraries: jemalloc
- System settings: `vm.max_map_count >= 262144` (for Elasticsearch)
- 16GB+ RAM, 50GB+ disk space

## Code Style and Quality

- **Backend**: Uses Ruff for linting and formatting (line length: 200)
  - Pre-commit hooks automatically run Ruff with `--fix` flag
  - Configuration in `pyproject.toml`
- **Frontend**: Uses ESLint and Prettier
  - Prettier runs on staged files via lint-staged
  - Configuration in `web/package.json` and related config files