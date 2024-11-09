# Async Processing Implementation

## Part 1: Basic Setup

1. Update Dependencies:
   In backend/pyproject.toml:

   ```toml
   [tool.poetry.dependencies]
   celery = "^5.3.6"
   redis = "^5.0.1"
   flower = "^2.0.1"  # For monitoring Celery tasks
   ```

2. Celery Configuration:
   backend/app/core/celery_app.py:

   ```python
   from celery import Celery
   from app.core.config import settings

   celery_app = Celery(
       "gpx_tracker",
       broker=settings.REDIS_URL,
       backend=settings.REDIS_URL,
       include=[
           "app.worker.tasks.gpx_tasks",
           "app.worker.tasks.area_tasks"
       ]
   )

   celery_app.conf.update(
       task_serializer="json",
       accept_content=["json"],
       result_serializer="json",
       timezone="UTC",
       enable_utc=True,
       worker_prefetch_multiplier=1,
       task_track_started=True,
       task_routes={
           "app.worker.tasks.gpx_tasks.*": {"queue": "gpx"},
           "app.worker.tasks.area_tasks.*": {"queue": "area"},
       },
       task_default_queue="default",
       # Retry settings
       task_retry_delay_start=1,  # 1 second initial delay
       task_max_retries=3,
       # Result settings
       task_ignore_result=False,
       result_expires=60 * 60 * 24,  # 24 hours
       # Error handling
       task_acks_late=True,
       task_reject_on_worker_lost=True,
   )

   # Optional: Configure Celery beat for periodic tasks
   celery_app.conf.beat_schedule = {
       "update-user-areas": {
           "task": "app.worker.tasks.area_tasks.update_all_user_areas",
           "schedule": 60 * 60,  # Every hour
       },
   }
   ```

3. Task Base Class:
   backend/app/worker/tasks/base.py:

   ```python
   from celery import Task
   from sqlalchemy.orm import Session
   from app.db.session import SessionLocal

   class DatabaseTask(Task):
       _db: Session | None = None

       @property
       def db(self) -> Session:
           if self._db is None:
               self._db = SessionLocal()
           return self._db

       def after_return(self, *args, **kwargs):
           if self._db is not None:
               self._db.close()
               self._db = None

       def on_failure(self, exc, task_id, args, kwargs, einfo):
           if self._db is not None:
               self._db.rollback()
           super().on_failure(exc, task_id, args, kwargs, einfo)

       def on_success(self, retval, task_id, args, kwargs):
           if self._db is not None:
               self._db.commit()
           super().on_success(retval, task_id, args, kwargs)
   ```

4. GPX Processing Tasks:
   backend/app/worker/tasks/gpx_tasks.py:

   ```python
   from typing import Any, Dict
   from pathlib import Path
   import gpxpy
   from celery import chain
   from app.worker.tasks.base import DatabaseTask
   from app.core.celery_app import celery_app
   from app.models.track import Track, TrackPoint
   from app.services.gpx_processor import GPXProcessor
   from app.services.storage import StorageService

   @celery_app.task(base=DatabaseTask, bind=True)
   def process_gpx_file(self, track_id: str, file_path: str) -> Dict[str, Any]:
       """Process a GPX file and store track points"""
       try:
           # Get track record
           track = self.db.query(Track).get(track_id)
           if not track:
               raise ValueError(f"Track {track_id} not found")

           # Read and parse GPX file
           with open(file_path, 'r') as gpx_file:
               processor = GPXProcessor(gpx_file.read())

           # Extract track points
           points = processor.get_track_points()

           # Store track points
           for point in points:
               track_point = TrackPoint(
                   track_id=track_id,
                   latitude=point.latitude,
                   longitude=point.longitude,
                   elevation=point.elevation,
                   time=point.time
               )
               self.db.add(track_point)

           # Update track statistics
           stats = processor.calculate_statistics()
           track.start_time = stats["start_time"]
           track.end_time = stats["end_time"]
           track.total_distance = stats["total_distance"]
           track.total_elevation_gain = stats["total_elevation_gain"]
           track.status = "completed"

           # Clean up temporary file
           Path(file_path).unlink(missing_ok=True)

           return {
               "track_id": track_id,
               "points_count": len(points),
               "statistics": stats
           }

       except Exception as e:
           track.status = "error"
           track.error_message = str(e)
           raise

   @celery_app.task(base=DatabaseTask)
   def calculate_track_area(track_id: str) -> Dict[str, Any]:
       """Calculate the area covered by a track"""
       track = self.db.query(Track).get(track_id)
       points = self.db.query(TrackPoint).filter(
           TrackPoint.track_id == track_id
       ).all()

       processor = GPXProcessor(None)
       area = processor.calculate_covered_area(points)

       return {
           "track_id": track_id,
           "area_km2": area["size"],
           "geojson": area["geojson"]
       }

   @celery_app.task
   def cleanup_temporary_files(file_paths: list[str]):
       """Clean up temporary files after processing"""
       for path in file_paths:
           Path(path).unlink(missing_ok=True)

   # Task chains
   def process_gpx_upload(track_id: str, file_path: str):
       """Chain GPX processing tasks"""
       return chain(
           process_gpx_file.s(track_id, file_path),
           calculate_track_area.s(track_id),
           cleanup_temporary_files.s([file_path])
       )()
   ```

## Part 2: Area Calculations and Monitoring

1. Area Calculation Tasks:
   backend/app/worker/tasks/area_tasks.py:

   ```python
   from typing import Any, Dict, List
   from celery import group
   from app.worker.tasks.base import DatabaseTask
   from app.core.celery_app import celery_app
   from app.models import User, Track, CoveredArea
   from app.services.area_calculator import AreaCalculator

   @celery_app.task(base=DatabaseTask)
   def calculate_user_area(user_id: str) -> Dict[str, Any]:
       """Calculate total area covered by user's tracks"""
       calculator = AreaCalculator(self.db)

       try:
           # Get all completed tracks for user
           tracks = self.db.query(Track).filter(
               Track.user_id == user_id,
               Track.status == "completed"
           ).all()

           # Calculate total covered area
           coverage = calculator.calculate_total_coverage(tracks)

           # Update or create covered area record
           covered_area = self.db.query(CoveredArea).filter(
               CoveredArea.user_id == user_id
           ).first()

           if covered_area:
               covered_area.area_km2 = coverage["area_km2"]
               covered_area.geom = coverage["geom"]
               covered_area.percentage = coverage["percentage"]
           else:
               covered_area = CoveredArea(
                   user_id=user_id,
                   area_km2=coverage["area_km2"],
                   geom=coverage["geom"],
                   percentage=coverage["percentage"]
               )
               self.db.add(covered_area)

           return {
               "user_id": user_id,
               "area_km2": coverage["area_km2"],
               "percentage": coverage["percentage"]
           }

       except Exception as e:
           self.logger.error(f"Error calculating area for user {user_id}: {e}")
           raise

   @celery_app.task
   def update_all_user_areas():
       """Update covered areas for all users (periodic task)"""
       users = self.db.query(User).all()
       job = group(calculate_user_area.s(user.id) for user in users)()
       return job.get()

   @celery_app.task(bind=True)
   def merge_overlapping_areas(self, area_ids: List[str]) -> Dict[str, Any]:
       """Merge overlapping covered areas"""
       calculator = AreaCalculator(self.db)
       return calculator.merge_areas(area_ids)
   ```

2. Task Status Monitoring:
   backend/app/services/task_monitor.py:

   ```python
   from typing import Any, Dict, Optional
   from celery.result import AsyncResult
   from app.core.celery_app import celery_app

   class TaskMonitor:
       @staticmethod
       def get_task_status(task_id: str) -> Dict[str, Any]:
           """Get the current status of a task"""
           result = AsyncResult(task_id, app=celery_app)

           return {
               "task_id": task_id,
               "status": result.state,
               "result": result.result if result.ready() else None,
               "error": str(result.result) if result.failed() else None,
               "progress": result.info.get("progress", 0) if result.state == "PROGRESS" else 0
           }

       @staticmethod
       def get_active_tasks() -> Dict[str, Any]:
           """Get all currently active tasks"""
           inspector = celery_app.control.inspect()

           return {
               "active": inspector.active() or {},
               "reserved": inspector.reserved() or {},
               "scheduled": inspector.scheduled() or {}
           }

       @staticmethod
       def revoke_task(task_id: str, terminate: bool = False) -> None:
           """Revoke a running or scheduled task"""
           celery_app.control.revoke(task_id, terminate=terminate)
   ```

3. Task Progress Updates:
   backend/app/worker/tasks/mixins.py:

   ```python
   from celery import Task
   from typing import Any, Dict

   class ProgressMixin:
       def update_progress(self: Task, current: int, total: int, **kwargs) -> None:
           """Update task progress"""
           if not self.request.id:
               return

           progress = round((current / total) * 100, 2)
           self.update_state(
               state="PROGRESS",
               meta={
                   "progress": progress,
                   "current": current,
                   "total": total,
                   **kwargs
               }
           )

   class LoggingMixin:
       def log_start(self: Task, *args, **kwargs) -> None:
           """Log task start"""
           self.logger.info(
               f"Starting task {self.name}[{self.request.id}]",
               extra={
                   "task_id": self.request.id,
                   "args": args,
                   "kwargs": kwargs
               }
           )

       def log_success(self: Task, result: Any) -> None:
           """Log task success"""
           self.logger.info(
               f"Task {self.name}[{self.request.id}] completed successfully",
               extra={
                   "task_id": self.request.id,
                   "result": result
               }
           )

       def log_failure(self: Task, exc: Exception, task_id: str, args: tuple, kwargs: Dict) -> None:
           """Log task failure"""
           self.logger.error(
               f"Task {self.name}[{task_id}] failed: {exc}",
               extra={
                   "task_id": task_id,
                   "args": args,
                   "kwargs": kwargs,
                   "exception": exc
               },
               exc_info=True
           )
   ```

## Part 3: API Integration and Worker Configuration

1. API Endpoints:
   backend/app/api/v1/endpoints/tasks.py:

   ```python
   from fastapi import APIRouter, HTTPException, Depends
   from sqlalchemy.orm import Session
   from typing import List
   from app.api import deps
   from app.services.task_monitor import TaskMonitor
   from app.schemas.task import TaskStatus, TaskList

   router = APIRouter()

   @router.get("/tasks/{task_id}", response_model=TaskStatus)
   def get_task_status(task_id: str):
       """Get status of a specific task"""
       return TaskMonitor.get_task_status(task_id)

   @router.get("/tasks", response_model=TaskList)
   def list_active_tasks():
       """Get list of all active tasks"""
       return TaskMonitor.get_active_tasks()

   @router.delete("/tasks/{task_id}")
   def cancel_task(task_id: str):
       """Cancel a running task"""
       try:
           TaskMonitor.revoke_task(task_id, terminate=True)
           return {"message": f"Task {task_id} has been cancelled"}
       except Exception as e:
           raise HTTPException(status_code=400, detail=str(e))

   @router.post("/tasks/retry/{task_id}")
   def retry_task(task_id: str):
       """Retry a failed task"""
       try:
           result = AsyncResult(task_id)
           result.retry()
           return {"message": f"Task {task_id} has been queued for retry"}
       except Exception as e:
           raise HTTPException(status_code=400, detail=str(e))
   ```

2. Worker Configuration:
   backend/worker.py:

   ```python
   import os
   from celery import Celery
   from celery.signals import worker_init, worker_process_init
   from app.core.config import settings
   from app.core.logging import setup_logging

   # Initialize Celery app
   app = Celery('gpx_tracker')
   app.config_from_object(settings, namespace='CELERY')

   # Set up logging for workers
   @worker_init.connect
   def init_worker(**kwargs):
       setup_logging()

   # Initialize database connections for each worker process
   @worker_process_init.connect
   def init_worker_process(**kwargs):
       # Ensure each worker process has its own database connection
       from app.db.session import engine
       engine.dispose()

   if __name__ == '__main__':
       app.start()
   ```

3. Celery Configuration File:
   backend/celeryconfig.py:

   ```python
   from app.core.config import settings

   # Broker settings
   broker_url = settings.REDIS_URL
   result_backend = settings.REDIS_URL

   # Task settings
   task_serializer = 'json'
   accept_content = ['json']
   result_serializer = 'json'
   enable_utc = True
   timezone = 'UTC'

   # Worker settings
   worker_prefetch_multiplier = 1
   worker_max_tasks_per_child = 1000
   worker_max_memory_per_child = 200000  # 200MB

   # Queue settings
   task_queues = {
       'default': {
           'exchange': 'default',
           'routing_key': 'default',
       },
       'gpx': {
           'exchange': 'gpx',
           'routing_key': 'gpx',
       },
       'area': {
           'exchange': 'area',
           'routing_key': 'area',
       },
   }

   task_routes = {
       'app.worker.tasks.gpx_tasks.*': {'queue': 'gpx'},
       'app.worker.tasks.area_tasks.*': {'queue': 'area'},
   }

   # Beat settings (periodic tasks)
   beat_schedule = {
       'update-user-areas': {
           'task': 'app.worker.tasks.area_tasks.update_all_user_areas',
           'schedule': 3600.0,  # Every hour
       },
       'cleanup-old-files': {
           'task': 'app.worker.tasks.gpx_tasks.cleanup_temporary_files',
           'schedule': 86400.0,  # Every day
       },
   }
   ```

4. Docker Compose Update:
   docker-compose.yml:

   ```yaml
   services:
     # ... existing services ...

     celery_worker:
       build:
         context: ./backend
         dockerfile: Dockerfile
       command: celery -A worker.app worker -l INFO -Q default,gpx,area
       volumes:
         - ./backend:/app
       depends_on:
         - redis
         - postgres
       environment:
         - REDIS_URL=redis://redis:6379/0
         - DATABASE_URL=postgresql://postgres:password@postgres:5432/gpx_tracker

     celery_beat:
       build:
         context: ./backend
         dockerfile: Dockerfile
       command: celery -A worker.app beat -l INFO
       volumes:
         - ./backend:/app
       depends_on:
         - redis
         - postgres
       environment:
         - REDIS_URL=redis://redis:6379/0
         - DATABASE_URL=postgresql://postgres:password@postgres:5432/gpx_tracker

     flower:
       build:
         context: ./backend
         dockerfile: Dockerfile
       command: celery -A worker.app flower --port=5555
       ports:
         - "5555:5555"
       depends_on:
         - redis
         - celery_worker
       environment:
         - REDIS_URL=redis://redis:6379/0
   ```

5. Supervisor Configuration (for production):
   backend/supervisor/celery.conf:

   ```ini
   [program:celery_worker]
   command=/app/venv/bin/celery -A worker.app worker -l INFO -Q default,gpx,area
   directory=/app
   user=celery
   numprocs=1
   stdout_logfile=/var/log/celery/worker.log
   stderr_logfile=/var/log/celery/worker.error.log
   autostart=true
   autorestart=true
   startsecs=10
   stopwaitsecs=600
   killasgroup=true
   priority=998

   [program:celery_beat]
   command=/app/venv/bin/celery -A worker.app beat -l INFO
   directory=/app
   user=celery
   numprocs=1
   stdout_logfile=/var/log/celery/beat.log
   stderr_logfile=/var/log/celery/beat.error.log
   autostart=true
   autorestart=true
   startsecs=10
   priority=999

   [group:celery]
   programs=celery_worker,celery_beat
   priority=999
   ```

##Part 4: Monitoring and Error Handling

1. Task Monitoring Service:
   backend/app/services/monitoring.py:

   ```python
   from datetime import datetime, timedelta
   from typing import Dict, List
   from sqlalchemy.orm import Session
   from app.models import Track
   from app.core.celery_app import celery_app

   class TaskMonitoringService:
       def __init__(self, db: Session):
           self.db = db

       def get_task_metrics(self) -> Dict:
           """Get metrics about task processing"""
           inspector = celery_app.control.inspect()
           active = inspector.active() or {}
           reserved = inspector.reserved() or {}
           revoked = inspector.revoked() or {}

           # Count tasks by status
           processing_tracks = self.db.query(Track).filter(
               Track.status == "processing"
           ).count()

           error_tracks = self.db.query(Track).filter(
               Track.status == "error"
           ).count()

           # Calculate average processing time
           completed_tracks = self.db.query(Track).filter(
               Track.status == "completed",
               Track.processed_at.isnot(None),
               Track.created_at >= datetime.utcnow() - timedelta(days=7)
           ).all()

           processing_times = [
               (t.processed_at - t.created_at).total_seconds()
               for t in completed_tracks
               if t.processed_at
           ]

           avg_processing_time = (
               sum(processing_times) / len(processing_times)
               if processing_times else 0
           )

           return {
               "active_tasks": sum(len(tasks) for tasks in active.values()),
               "reserved_tasks": sum(len(tasks) for tasks in reserved.values()),
               "revoked_tasks": sum(len(tasks) for tasks in revoked.values()),
               "processing_tracks": processing_tracks,
               "error_tracks": error_tracks,
               "avg_processing_time": avg_processing_time,
               "total_completed_7d": len(completed_tracks)
           }

       def get_failed_tasks(self) -> List[Dict]:
           """Get information about failed tasks"""
           failed_tracks = self.db.query(Track).filter(
               Track.status == "error"
           ).order_by(Track.created_at.desc()).limit(50).all()

           return [
               {
                   "id": track.id,
                   "filename": track.filename,
                   "error_message": track.error_message,
                   "created_at": track.created_at,
                   "user_id": track.user_id
               }
               for track in failed_tracks
           ]

       def cleanup_stuck_tasks(self) -> int:
           """Reset tasks that have been stuck in processing state"""
           stuck_time = datetime.utcnow() - timedelta(hours=1)
           stuck_tracks = self.db.query(Track).filter(
               Track.status == "processing",
               Track.created_at < stuck_time
           ).all()

           for track in stuck_tracks:
               track.status = "error"
               track.error_message = "Task timed out"

           self.db.commit()
           return len(stuck_tracks)
   ```

2. Error Handling Utilities:
   backend/app/worker/utils/error_handling.py:

   ```python
   import traceback
   from functools import wraps
   from typing import Any, Callable
   from celery import Task
   from app.core.logging import logger

   def handle_task_failure(task: Task, exc: Exception, task_id: str, args: tuple, kwargs: dict) -> None:
       """Handle task failure with proper logging and cleanup"""
       logger.error(
           f"Task {task.name}[{task_id}] failed",
           extra={
               "task_id": task_id,
               "args": args,
               "kwargs": kwargs,
               "exception": str(exc),
               "traceback": traceback.format_exc()
           }
       )

       # Cleanup any resources if needed
       try:
           task.cleanup_after_failure(task_id, args, kwargs)
       except Exception as cleanup_exc:
           logger.error(
               f"Error during failure cleanup for task {task_id}",
               exc_info=cleanup_exc
           )

   def retry_on_error(
       max_retries: int = 3,
       countdown: int = 60,
       exponential_backoff: bool = True
   ) -> Callable:
       """Decorator to handle task retries with exponential backoff"""
       def decorator(func: Callable) -> Callable:
           @wraps(func)
           def wrapper(self: Task, *args: Any, **kwargs: Any) -> Any:
               try:
                   return func(self, *args, **kwargs)
               except Exception as exc:
                   # Calculate retry delay
                   retry_delay = countdown
                   if exponential_backoff:
                       retry_delay = countdown * (2 ** self.request.retries)

                   # Log the error
                   logger.warning(
                       f"Task {self.name} failed, attempt {self.request.retries + 1}",
                       exc_info=exc
                   )

                   # Retry if we haven't exceeded max_retries
                   if self.request.retries < max_retries:
                       raise self.retry(
                           exc=exc,
                           countdown=retry_delay,
                           max_retries=max_retries
                       )
                   else:
                       # If we've exceeded max_retries, handle the final failure
                       handle_task_failure(
                           self,
                           exc,
                           self.request.id,
                           args,
                           kwargs
                       )
                       raise
           return wrapper
       return decorator
   ```

3. Monitoring API Endpoints:
   backend/app/api/v1/endpoints/monitoring.py:

   ```python
   from fastapi import APIRouter, Depends
   from sqlalchemy.orm import Session
   from app.api import deps
   from app.services.monitoring import TaskMonitoringService

   router = APIRouter()

   @router.get("/metrics")
   def get_task_metrics(db: Session = Depends(deps.get_db)):
       """Get task processing metrics"""
       monitoring = TaskMonitoringService(db)
       return monitoring.get_task_metrics()

   @router.get("/failed-tasks")
   def get_failed_tasks(db: Session = Depends(deps.get_db)):
       """Get information about failed tasks"""
       monitoring = TaskMonitoringService(db)
       return monitoring.get_failed_tasks()

   @router.post("/cleanup-stuck")
   def cleanup_stuck_tasks(db: Session = Depends(deps.get_db)):
       """Clean up tasks stuck in processing state"""
       monitoring = TaskMonitoringService(db)
       cleaned_count = monitoring.cleanup_stuck_tasks()
       return {"cleaned_tasks": cleaned_count}
   ```

## Async Processing Implementation Complete

The asynchronous processing system is now complete with the following components:

4. Core Features:

   - Celery task queue with Redis broker
   - GPX file processing tasks
   - Area calculation tasks
   - Task monitoring and metrics

5. Error Handling:

   - Automatic retries with exponential backoff
   - Comprehensive error logging
   - Task failure cleanup
   - Stuck task detection

6. Monitoring:

   - Task status tracking
   - Performance metrics
   - Failed task reporting
   - Flower dashboard integration

7. Production Setup:
   - Docker Compose configuration
   - Supervisor process management
   - Multiple worker queues
   - Resource management

To run the system:

1. Start the services:

```bash
# Start all services with Docker Compose
docker-compose up -d

# Or start individual components:
docker-compose up -d redis postgres
docker-compose up -d celery_worker celery_beat flower
```

2. Monitor tasks:

```bash
# View Flower dashboard
open http://localhost:5555

# Check task metrics
curl http://localhost:8000/api/v1/monitoring/metrics
```

3. Example task usage:

```python
from app.worker.tasks.gpx_tasks import process_gpx_upload

# Process GPX file
task = process_gpx_upload.delay(track_id, file_path)

# Check task status
status = task.status
result = task.get() # Wait for result

# Cancel task if needed
task.revoke(terminate=True)
```

Key Configuration Files:

- celeryconfig.py: Celery settings
- supervisor/celery.conf: Production process management
- docker-compose.yml: Container orchestration

Monitoring Tools:

- Flower: http://localhost:5555
- API endpoints: /api/v1/monitoring/\*
- Log files: /var/log/celery/\*

Maintenance Tasks:

1. Monitor queue sizes and processing times
2. Check for stuck tasks regularly
3. Review error logs and failed tasks
4. Scale workers based on load

Next steps could include:

1. Adding more sophisticated monitoring
2. Implementing task prioritization
3. Adding real-time progress updates
4. Optimizing worker configurations
5. Implementing task rate limiting
6. Adding more detailed metrics and alerts
