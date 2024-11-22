# Deployment Preparation

## Part 1: Production Docker Setup

1. Production Dockerfile for Backend:
   backend/Dockerfile:

   ```dockerfile
   # Build stage
   FROM python:3.9-slim as builder

   WORKDIR /app

   # Install build dependencies
   RUN apt-get update && apt-get install -y --no-install-recommends \\
       build-essential \\
       libpq-dev \\
       && rm -rf /var/lib/apt/lists/*

   # Install Poetry
   RUN pip install --no-cache-dir poetry

   # Copy dependency files
   COPY pyproject.toml poetry.lock ./

   # Configure Poetry and install dependencies
   RUN poetry config virtualenvs.create false \\
       && poetry install --no-dev --no-interaction

   # Final stage
   FROM python:3.9-slim

   WORKDIR /app

   # Install runtime dependencies
   RUN apt-get update && apt-get install -y --no-install-recommends \\
       libpq5 \\
       && rm -rf /var/lib/apt/lists/*

   # Copy dependencies from builder
   COPY --from=builder /usr/local/lib/python3.9/site-packages /usr/local/lib/python3.9/site-packages
   COPY --from=builder /usr/local/bin /usr/local/bin

   # Copy application code
   COPY . .

   # Create non-root user
   RUN useradd -m appuser && chown -R appuser:appuser /app
   USER appuser

   # Set environment variables
   ENV PYTHONPATH=/app \\
       PYTHONUNBUFFERED=1

   # Health check
   HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \\
       CMD curl -f http://localhost:8000/health || exit 1

   EXPOSE 8000

   CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
   ```

2. Production Dockerfile for Frontend:
   frontend/Dockerfile:

   ```dockerfile
   # Build stage
   FROM node:18-alpine as builder

   WORKDIR /app

   # Install dependencies
   COPY package*.json ./
   RUN npm ci

   # Copy source code
   COPY . .

   # Build application
   RUN npm run build

   # Final stage with Nginx
   FROM nginx:alpine

   # Copy built assets
   COPY --from=builder /app/dist /usr/share/nginx/html

   # Copy Nginx configuration
   COPY nginx.conf /etc/nginx/conf.d/default.conf

   # Add health check
   HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \\
       CMD wget -q --spider http://localhost:80/health || exit 1

   EXPOSE 80
   ```

3. Production Nginx Configuration:
   frontend/nginx.conf:

   ```nginx
   server {
       listen 80;
       server_name _;
       root /usr/share/nginx/html;
       index index.html;

       # Security headers
       add_header X-Frame-Options "DENY";
       add_header X-Content-Type-Options "nosniff";
       add_header X-XSS-Protection "1; mode=block";
       add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline';";

       # Gzip compression
       gzip on;
       gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

       # Cache control
       location /static/ {
           expires 1y;
           add_header Cache-Control "public, no-transform";
       }

       # Health check endpoint
       location /health {
           access_log off;
           return 200 'healthy';
           add_header Content-Type text/plain;
       }

       # API proxy
       location /api/ {
           proxy_pass http://backend:8000;
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection 'upgrade';
           proxy_set_header Host $host;
           proxy_cache_bypass $http_upgrade;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

           # Rate limiting
           limit_req zone=api burst=10 nodelay;
       }

       # SPA routing
       location / {
           try_files $uri $uri/ /index.html;
           expires -1;
           add_header Cache-Control "no-store, no-cache, must-revalidate";
       }
   }

   # Rate limiting configuration
   limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
   ```

4. Production Docker Compose:
   docker-compose.prod.yml:

   ```yaml
   version: "3.8"

   services:
     postgres:
       image: postgis/postgis:14-3.3
       volumes:
         - postgres_data:/var/lib/postgresql/data
       environment:
         POSTGRES_USER: ${POSTGRES_USER}
         POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
         POSTGRES_DB: ${POSTGRES_DB}
       healthcheck:
         test: ["CMD-SHELL", "pg_isready -U postgres"]
         interval: 10s
         timeout: 5s
         retries: 5
       deploy:
         resources:
           limits:
             cpus: "1"
             memory: 1G

     redis:
       image: redis:alpine
       volumes:
         - redis_data:/data
       command: redis-server --appendonly yes
       healthcheck:
         test: ["CMD", "redis-cli", "ping"]
         interval: 10s
         timeout: 5s
         retries: 5
       deploy:
         resources:
           limits:
             cpus: "0.5"
             memory: 500M

     backend:
       build:
         context: ./backend
         dockerfile: Dockerfile
       environment:
         DATABASE_URL: postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}
         REDIS_URL: redis://redis:6379/0
         SECRET_KEY: ${SECRET_KEY}
       depends_on:
         postgres:
           condition: service_healthy
         redis:
           condition: service_healthy
       deploy:
         replicas: 2
         resources:
           limits:
             cpus: "0.5"
             memory: 500M

     celery_worker:
       build:
         context: ./backend
         dockerfile: Dockerfile
       command: celery -A app.worker.celery worker --loglevel=info
       environment:
         DATABASE_URL: postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}
         REDIS_URL: redis://redis:6379/0
       depends_on:
         postgres:
           condition: service_healthy
         redis:
           condition: service_healthy
       deploy:
         replicas: 2
         resources:
           limits:
             cpus: "0.5"
             memory: 500M

     frontend:
       build:
         context: ./frontend
         dockerfile: Dockerfile
       ports:
         - "80:80"
       depends_on:
         - backend
       deploy:
         replicas: 2
         resources:
           limits:
             cpus: "0.25"
             memory: 250M

   volumes:
     postgres_data:
     redis_data:

   networks:
     default:
       driver: overlay
       attachable: true
   ```

## Part 2: CI/CD Pipeline

1. GitHub Actions Workflow:
   .github/workflows/main.yml:

   ```yaml
   name: CI/CD Pipeline

   on:
     push:
       branches: [main]
     pull_request:
       branches: [main]

   jobs:
     test:
       runs-on: ubuntu-latest
       services:
         postgres:
           image: postgis/postgis:14-3.3
           env:
             POSTGRES_USER: postgres
             POSTGRES_PASSWORD: postgres
             POSTGRES_DB: test_db
           ports:
             - 5432:5432
           options: >-
             --health-cmd pg_isready
             --health-interval 10s
             --health-timeout 5s
             --health-retries 5

         redis:
           image: redis:alpine
           ports:
             - 6379:6379
           options: >-
             --health-cmd "redis-cli ping"
             --health-interval 10s
             --health-timeout 5s
             --health-retries 5

       steps:
         - uses: actions/checkout@v3

         - name: Set up Python
           uses: actions/setup-python@v4
           with:
             python-version: "3.9"

         - name: Install Poetry
           run: curl -sSL https://install.python-poetry.org | python3 -

         - name: Install backend dependencies
           working-directory: ./backend
           run: poetry install

         - name: Run backend tests
           working-directory: ./backend
           env:
             DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db
             REDIS_URL: redis://localhost:6379/0
             SECRET_KEY: test-secret-key
           run: |
             poetry run pytest --cov=app --cov-report=xml

         - name: Set up Node.js
           uses: actions/setup-node@v3
           with:
             node-version: "18"

         - name: Install frontend dependencies
           working-directory: ./frontend
           run: npm ci

         - name: Run frontend tests
           working-directory: ./frontend
           run: npm test

         - name: Upload coverage reports
           uses: codecov/codecov-action@v3
           with:
             files: ./backend/coverage.xml,./frontend/coverage/coverage-final.json

     build:
       needs: test
       runs-on: ubuntu-latest
       if: github.ref == 'refs/heads/main'
       steps:
         - uses: actions/checkout@v3

         - name: Configure AWS credentials
           uses: aws-actions/configure-aws-credentials@v2
           with:
             aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
             aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
             aws-region: ${{ secrets.AWS_REGION }}

         - name: Login to Amazon ECR
           id: login-ecr
           uses: aws-actions/amazon-ecr-login@v1

         - name: Build and push backend image
           env:
             ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
             ECR_REPOSITORY: gpx-tracker-backend
             IMAGE_TAG: ${{ github.sha }}
           run: |
             docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG ./backend
             docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG

         - name: Build and push frontend image
           env:
             ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
             ECR_REPOSITORY: gpx-tracker-frontend
             IMAGE_TAG: ${{ github.sha }}
           run: |
             docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG ./frontend
             docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG

     deploy:
       needs: build
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v3

         - name: Configure AWS credentials
           uses: aws-actions/configure-aws-credentials@v2
           with:
             aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
             aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
             aws-region: ${{ secrets.AWS_REGION }}

         - name: Fill in the new image ID in the Amazon ECS task definition
           id: task-def
           uses: aws-actions/amazon-ecs-render-task-definition@v1
           with:
             task-definition: .aws/task-definition.json
             container-name: gpx-tracker
             image: ${{ steps.login-ecr.outputs.registry }}/gpx-tracker-backend:${{ github.sha }}

         - name: Deploy to Amazon ECS
           uses: aws-actions/amazon-ecs-deploy-task-definition@v1
           with:
             task-definition: ${{ steps.task-def.outputs.task-definition }}
             service: gpx-tracker-service
             cluster: gpx-tracker-cluster
             wait-for-service-stability: true
   ```

## Part 3: Cloud Infrastructure

1. Terraform AWS Infrastructure:
   infrastructure/terraform/main.tf:

   ```hcl
   provider "aws" {
     region = var.aws_region
   }

   # VPC and Networking
   module "vpc" {
     source = "terraform-aws-modules/vpc/aws"
     version = "~> 3.0"

     name = "gpx-tracker-vpc"
     cidr = "10.0.0.0/16"

     azs             = ["${var.aws_region}a", "${var.aws_region}b", "${var.aws_region}c"]
     private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
     public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

     enable_nat_gateway = true
     single_nat_gateway = false
     one_nat_gateway_per_az = true

     enable_vpn_gateway = false

     tags = {
       Environment = var.environment
       Project     = "gpx-tracker"
     }
   }

   # ECS Cluster
   resource "aws_ecs_cluster" "main" {
     name = "gpx-tracker-cluster-${var.environment}"

     setting {
       name  = "containerInsights"
       value = "enabled"
     }

     tags = {
       Environment = var.environment
       Project     = "gpx-tracker"
     }
   }

   # RDS Instance
   resource "aws_db_instance" "postgres" {
     identifier = "gpx-tracker-${var.environment}"
     engine     = "postgres"
     engine_version = "14"
     instance_class = var.db_instance_class

     allocated_storage = 20
     max_allocated_storage = 100

     username = var.db_username
     password = var.db_password

     vpc_security_group_ids = [aws_security_group.rds.id]
     db_subnet_group_name   = aws_db_subnet_group.main.name

     backup_retention_period = 7
     backup_window          = "03:00-04:00"
     maintenance_window     = "Mon:04:00-Mon:05:00"

     multi_az = var.environment == "production"

     skip_final_snapshot = var.environment != "production"

     tags = {
       Environment = var.environment
       Project     = "gpx-tracker"
     }
   }

   # Elasticache Redis
   resource "aws_elasticache_cluster" "redis" {
     cluster_id           = "gpx-tracker-${var.environment}"
     engine               = "redis"
     node_type            = var.redis_node_type
     num_cache_nodes      = 1
     parameter_group_name = "default.redis6.x"
     port                 = 6379

     subnet_group_name    = aws_elasticache_subnet_group.main.name
     security_group_ids   = [aws_security_group.redis.id]

     tags = {
       Environment = var.environment
       Project     = "gpx-tracker"
     }
   }

   # Application Load Balancer
   resource "aws_lb" "main" {
     name               = "gpx-tracker-${var.environment}"
     internal           = false
     load_balancer_type = "application"
     security_groups    = [aws_security_group.alb.id]
     subnets            = module.vpc.public_subnets

     enable_deletion_protection = var.environment == "production"

     tags = {
       Environment = var.environment
       Project     = "gpx-tracker"
     }
   }

   # ECS Service
   resource "aws_ecs_service" "backend" {
     name            = "gpx-tracker-backend"
     cluster         = aws_ecs_cluster.main.id
     task_definition = aws_ecs_task_definition.backend.arn
     desired_count   = var.backend_instance_count

     launch_type         = "FARGATE"
     platform_version    = "1.4.0"

     network_configuration {
       subnets          = module.vpc.private_subnets
       security_groups  = [aws_security_group.ecs_tasks.id]
       assign_public_ip = false
     }

     load_balancer {
       target_group_arn = aws_lb_target_group.backend.arn
       container_name   = "backend"
       container_port   = 8000
     }

     deployment_controller {
       type = "ECS"
     }

     deployment_circuit_breaker {
       enable   = true
       rollback = true
     }

     tags = {
       Environment = var.environment
       Project     = "gpx-tracker"
     }
   }

   # S3 Bucket for file storage
   resource "aws_s3_bucket" "storage" {
     bucket = "gpx-tracker-storage-${var.environment}"

     tags = {
       Environment = var.environment
       Project     = "gpx-tracker"
     }
   }

   resource "aws_s3_bucket_public_access_block" "storage" {
     bucket = aws_s3_bucket.storage.id

     block_public_acls       = true
     block_public_policy     = true
     ignore_public_acls      = true
     restrict_public_buckets = true
   }
   ```

2. Variables Configuration:
   infrastructure/terraform/variables.tf:

   ```hcl
   variable "aws_region" {
     description = "AWS region"
     type        = string
     default     = "us-west-2"
   }

   variable "environment" {
     description = "Environment name"
     type        = string
     validation {
       condition     = contains(["development", "staging", "production"], var.environment)
       error_message = "Environment must be one of: development, staging, production."
     }
   }

   variable "db_instance_class" {
     description = "RDS instance class"
     type        = string
     default     = "db.t3.small"
   }

   variable "db_username" {
     description = "Database admin username"
     type        = string
     sensitive   = true
   }

   variable "db_password" {
     description = "Database admin password"
     type        = string
     sensitive   = true
   }

   variable "redis_node_type" {
     description = "Redis node type"
     type        = string
     default     = "cache.t3.micro"
   }

   variable "backend_instance_count" {
     description = "Number of backend instances"
     type        = number
     default     = 2
   }
   ```

3. Security Groups:
   infrastructure/terraform/security.tf:

   ```hcl
   resource "aws_security_group" "alb" {
     name        = "gpx-tracker-alb-${var.environment}"
     description = "ALB Security Group"
     vpc_id      = module.vpc.vpc_id

     ingress {
       description = "HTTP from anywhere"
       from_port   = 80
       to_port     = 80
       protocol    = "tcp"
       cidr_blocks = ["0.0.0.0/0"]
     }

     ingress {
       description = "HTTPS from anywhere"
       from_port   = 443
       to_port     = 443
       protocol    = "tcp"
       cidr_blocks = ["0.0.0.0/0"]
     }

     egress {
       from_port   = 0
       to_port     = 0
       protocol    = "-1"
       cidr_blocks = ["0.0.0.0/0"]
     }

     tags = {
       Environment = var.environment
       Project     = "gpx-tracker"
     }
   }
   ```

## Part 4: Monitoring and Logging

1. CloudWatch Logging Configuration:
   infrastructure/terraform/monitoring.tf:

   ```hcl
   # CloudWatch Log Groups
   resource "aws_cloudwatch_log_group" "backend" {
     name              = "/ecs/gpx-tracker-backend-${var.environment}"
     retention_in_days = 30

     tags = {
       Environment = var.environment
       Project     = "gpx-tracker"
     }
   }

   resource "aws_cloudwatch_log_group" "frontend" {
     name              = "/ecs/gpx-tracker-frontend-${var.environment}"
     retention_in_days = 30

     tags = {
       Environment = var.environment
       Project     = "gpx-tracker"
     }
   }

   # CloudWatch Alarms
   resource "aws_cloudwatch_metric_alarm" "backend_cpu" {
     alarm_name          = "gpx-tracker-backend-cpu-${var.environment}"
     comparison_operator = "GreaterThanThreshold"
     evaluation_periods  = "2"
     metric_name         = "CPUUtilization"
     namespace           = "AWS/ECS"
     period              = "300"
     statistic           = "Average"
     threshold           = "85"

     dimensions = {
       ClusterName = aws_ecs_cluster.main.name
       ServiceName = aws_ecs_service.backend.name
     }

     alarm_description = "This metric monitors ECS backend CPU utilization"
     alarm_actions     = [aws_sns_topic.alerts.arn]
   }

   resource "aws_cloudwatch_metric_alarm" "rds_cpu" {
     alarm_name          = "gpx-tracker-rds-cpu-${var.environment}"
     comparison_operator = "GreaterThanThreshold"
     evaluation_periods  = "2"
     metric_name         = "CPUUtilization"
     namespace           = "AWS/RDS"
     period              = "300"
     statistic           = "Average"
     threshold           = "85"

     dimensions = {
       DBInstanceIdentifier = aws_db_instance.postgres.id
     }

     alarm_description = "This metric monitors RDS CPU utilization"
     alarm_actions     = [aws_sns_topic.alerts.arn]
   }

   # SNS Topic for Alerts
   resource "aws_sns_topic" "alerts" {
     name = "gpx-tracker-alerts-${var.environment}"
   }
   ```

2. Application Monitoring Configuration:
   backend/app/core/monitoring.py:

   ```python
   import logging
   from typing import Any, Dict
   from datadog import initialize, statsd
   from prometheus_client import Counter, Histogram
   from app.core.config import settings

   # Initialize DataDog
   initialize(
       api_key=settings.DATADOG_API_KEY,
       app_key=settings.DATADOG_APP_KEY
   )

   # Prometheus metrics
   TRACK_UPLOAD_COUNTER = Counter(
       'gpx_track_uploads_total',
       'Total number of GPX track uploads'
   )

   PROCESSING_TIME = Histogram(
       'gpx_processing_duration_seconds',
       'Time spent processing GPX tracks'
   )

   class Monitoring:
       @staticmethod
       def track_upload(user_id: str, file_size: int) -> None:
           """Track GPX file upload"""
           TRACK_UPLOAD_COUNTER.inc()
           statsd.increment('gpx.uploads')
           statsd.histogram('gpx.file_size', file_size)

       @staticmethod
       def track_processing_time(duration: float) -> None:
           """Track GPX processing time"""
           PROCESSING_TIME.observe(duration)
           statsd.histogram('gpx.processing_time', duration)

       @staticmethod
       def track_error(error_type: str, details: Dict[str, Any]) -> None:
           """Track application errors"""
           statsd.increment(f'gpx.errors.{error_type}')
           logging.error(f"Error: {error_type}", extra=details)
   ```

3. Monitoring Middleware:
   backend/app/middleware/monitoring.py:

   ```python
   from fastapi import Request
   from time import time
   from app.core.monitoring import Monitoring

   async def monitoring_middleware(request: Request, call_next):
       start_time = time()

       try:
           response = await call_next(request)

           duration = time() - start_time
           Monitoring.track_request(
               path=request.url.path,
               method=request.method,
               status_code=response.status_code,
               duration=duration
           )

           return response
       except Exception as e:
           duration = time() - start_time
           Monitoring.track_error(
               'request_error',
               {
                   'path': request.url.path,
                   'method': request.method,
                   'error': str(e),
                   'duration': duration
               }
           )
           raise
   ```

4. Grafana Dashboard Configuration:
   infrastructure/grafana/dashboard.json:

   ```json
   {
     "annotations": {
       "list": [
         {
           "builtIn": 1,
           "datasource": "-- Grafana --",
           "enable": true,
           "hide": true,
           "iconColor": "rgba(0, 211, 255, 1)",
           "name": "Annotations & Alerts",
           "type": "dashboard"
         }
       ]
     },
     "description": "GPX Tracker Application Metrics",
     "editable": true,
     "gnetId": null,
     "graphTooltip": 0,
     "id": 1,
     "links": [],
     "panels": [
       {
         "aliasColors": {},
         "bars": false,
         "dashLength": 10,
         "dashes": false,
         "datasource": null,
         "fieldConfig": {
           "defaults": {
             "custom": {}
           },
           "overrides": []
         },
         "fill": 1,
         "fillGradient": 0,
         "gridPos": {
           "h": 8,
           "w": 12,
           "x": 0,
           "y": 0
         },
         "hiddenSeries": false,
         "id": 2,
         "legend": {
           "avg": false,
           "current": false,
           "max": false,
           "min": false,
           "show": true,
           "total": false,
           "values": false
         },
         "lines": true,
         "linewidth": 1,
         "nullPointMode": "null",
         "percentage": false,
         "pointradius": 2,
         "points": false,
         "renderer": "flot",
         "seriesOverrides": [],
         "spaceLength": 10,
         "stack": false,
         "steppedLine": false,
         "targets": [
           {
             "expr": "rate(gpx_track_uploads_total[5m])",
             "interval": "",
             "legendFormat": "Uploads per second",
             "refId": "A"
           }
         ],
         "thresholds": [],
         "timeFrom": null,
         "timeRegions": [],
         "timeShift": null,
         "title": "Track Upload Rate",
         "tooltip": {
           "shared": true,
           "sort": 0,
           "value_type": "individual"
         },
         "type": "graph",
         "xaxis": {
           "buckets": null,
           "mode": "time",
           "name": null,
           "show": true,
           "values": []
         },
         "yaxes": [
           {
             "format": "short",
             "label": null,
             "logBase": 1,
             "max": null,
             "min": null,
             "show": true
           },
           {
             "format": "short",
             "label": null,
             "logBase": 1,
             "max": null,
             "min": null,
             "show": true
           }
         ],
         "yaxis": {
           "align": false,
           "alignLevel": null
         }
       }
     ],
     "refresh": "5s",
     "schemaVersion": 25,
     "style": "dark",
     "tags": [],
     "templating": {
       "list": []
     },
     "time": {
       "from": "now-6h",
       "to": "now"
     },
     "timepicker": {},
     "timezone": "",
     "title": "GPX Tracker Metrics",
     "uid": "gpx-tracker-metrics",
     "version": 1
   }
   ```

5. Application Performance Monitoring:
   backend/app/core/apm.py:

   ```python
   from elasticapm.contrib.starlette import ElasticAPM
   from fastapi import FastAPI
   from app.core.config import settings

   def setup_apm(app: FastAPI) -> None:
       """Configure Elastic APM for application monitoring"""
       app.add_middleware(
           ElasticAPM,
           server_url=settings.ELASTIC_APM_SERVER_URL,
           service_name="gpx-tracker",
           environment=settings.ENVIRONMENT,
           secret_token=settings.ELASTIC_APM_SECRET_TOKEN,
           transaction_sample_rate=1.0,
           capture_body="all",
           capture_headers=True
       )

   def track_task(task_name: str):
       """Decorator to track Celery tasks in APM"""
       def decorator(func):
           def wrapper(*args, **kwargs):
               from elasticapm import Client
               client = Client()

               with client.capture_transaction(task_name, "celery"):
                   try:
                       return func(*args, **kwargs)
                   except Exception as e:
                       client.capture_exception()
                       raise
           return wrapper
       return decorator
   ```

6. Logging Configuration:
   backend/app/core/logging.py:

   ```python
   import logging
   import json
   from datetime import datetime
   from pythonjsonlogger import jsonlogger
   from app.core.config import settings

   class CustomJsonFormatter(jsonlogger.JsonFormatter):
       def add_fields(self, log_record, record, message_dict):
           super(CustomJsonFormatter, self).add_fields(log_record, record, message_dict)
           log_record['timestamp'] = datetime.utcnow().isoformat()
           log_record['level'] = record.levelname
           log_record['environment'] = settings.ENVIRONMENT

           if hasattr(record, 'request_id'):
               log_record['request_id'] = record.request_id

   def setup_logging():
       """Configure application logging"""
       logger = logging.getLogger()
       handler = logging.StreamHandler()

       formatter = CustomJsonFormatter(
           '%(timestamp)s %(level)s %(name)s %(message)s'
       )
       handler.setFormatter(formatter)

       logger.addHandler(handler)
       logger.setLevel(settings.LOG_LEVEL)

       # Disable unnecessary logs
       logging.getLogger('boto3').setLevel(logging.WARNING)
       logging.getLogger('botocore').setLevel(logging.WARNING)
       logging.getLogger('urllib3').setLevel(logging.WARNING)

       return logger

   def log_request_middleware(request_id: str):
       """Add request ID to log context"""
       logger = logging.getLogger()
       old_factory = logger.getLogRecordFactory()

       def record_factory(*args, **kwargs):
           record = old_factory(*args, **kwargs)
           record.request_id = request_id
           return record

       logger.setLogRecordFactory(record_factory)
   ```

7. Monitoring Dashboard Script:
   scripts/create_dashboards.py:

   ```python
   import requests
   import json
   import os
   from typing import Dict, Any

   def create_grafana_dashboard(api_key: str, dashboard_json: Dict[str, Any]) -> None:
       """Create or update Grafana dashboard"""
       headers = {
           'Authorization': f'Bearer {api_key}',
           'Content-Type': 'application/json',
       }

       payload = {
           'dashboard': dashboard_json,
           'overwrite': True
       }

       response = requests.post(
           f'{os.environ["GRAFANA_URL"]}/api/dashboards/db',
           headers=headers,
           json=payload
       )

       response.raise_for_status()
       print(f"Dashboard created: {dashboard_json['title']}")

   if __name__ == '__main__':
       # Load dashboard configurations
       dashboards_dir = 'infrastructure/grafana/'
       for filename in os.listdir(dashboards_dir):
           if filename.endswith('.json'):
               with open(os.path.join(dashboards_dir, filename)) as f:
                   dashboard = json.load(f)
                   create_grafana_dashboard(
                       os.environ['GRAFANA_API_KEY'],
                       dashboard
                   )
   ```
