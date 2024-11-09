# Database Setup Guide

## Table of Contents

- [Database Setup Guide](#database-setup-guide)
  - [Initial Configuration](#initial-configuration)
    - [PostgreSQL and PostGIS Verification](#postgresql-and-postgis-verification)
    - [Schema Setup](#schema-setup)
  - [Database Optimization](#database-optimization)
    - [Index Creation](#index-creation)
    - [Trigger Functions](#trigger-functions)
    - [Area Calculation Functions](#area-calculation-functions)
  - [Application Integration](#application-integration)
    - [Database Connection](#database-connection)
    - [SQLAlchemy Models](#sqlalchemy-models)
  - [Migration Management](#migration-management)
    - [Alembic Setup](#alembic-setup)
    - [Migration Configuration](#migration-configuration)
  - [Utilities and Maintenance](#utilities-and-maintenance)
    - [Database Utilities](#database-utilities)
    - [Maintenance Scripts](#maintenance-scripts)
    - [Optimization Procedures](#optimization-procedures)
  - [Production Configuration](#production-configuration)
    - [Maintenance Schedule](#maintenance-schedule)
    - [Connection Pool Settings](#connection-pool-settings)
  - [Setup Verification](#setup-verification)
    - [Initial Verification](#initial-verification)
    - [Maintenance Guidelines](#maintenance-guidelines)

## Initial Configuration

### PostgreSQL and PostGIS Verification

```bash
# Connect to PostgreSQL container
docker exec -it gpx-tracker-postgres psql -U postgres

# Verify PostGIS extension
SELECT PostGIS_version();

# Create database if not exists
CREATE DATABASE gpx_tracker;
\c gpx_tracker

# Enable PostGIS extension
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
```

### Schema Setup

```sql
-- Create custom types
CREATE TYPE user_role AS ENUM ('user', 'admin');
CREATE TYPE track_status AS ENUM ('processing', 'completed', 'error');

-- Users table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email VARCHAR(255) UNIQUE NOT NULL,
    hashed_password VARCHAR(255) NOT NULL,
    full_name VARCHAR(255),
    role user_role NOT NULL DEFAULT 'user',
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- GPX tracks table
CREATE TABLE gpx_tracks (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    filename VARCHAR(255) NOT NULL,
    original_filename VARCHAR(255) NOT NULL,
    file_size INTEGER NOT NULL,
    status track_status NOT NULL DEFAULT 'processing',
    start_time TIMESTAMP WITH TIME ZONE,
    end_time TIMESTAMP WITH TIME ZONE,
    total_distance DECIMAL(10, 2),  -- in meters
    total_elevation_gain DECIMAL(10, 2),  -- in meters
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT gpx_tracks_user_id_fkey FOREIGN KEY (user_id)
        REFERENCES users(id) ON DELETE CASCADE
);

-- Track points table
CREATE TABLE track_points (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    track_id UUID NOT NULL REFERENCES gpx_tracks(id) ON DELETE CASCADE,
    location GEOGRAPHY(POINT, 4326) NOT NULL,
    elevation DECIMAL(8, 2),  -- in meters
    time TIMESTAMP WITH TIME ZONE,
    heart_rate INTEGER,  -- optional heart rate data
    speed DECIMAL(5, 2),  -- in m/s
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT track_points_track_id_fkey FOREIGN KEY (track_id)
        REFERENCES gpx_tracks(id) ON DELETE CASCADE
);

-- Covered areas table
CREATE TABLE covered_areas (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    geom GEOGRAPHY(MULTIPOLYGON, 4326) NOT NULL,
    area_km2 DECIMAL(10, 2) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT covered_areas_user_id_fkey FOREIGN KEY (user_id)
        REFERENCES users(id) ON DELETE CASCADE
);
```

## Database Optimization

### Index Creation

```sql
-- Users table indexes
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_created_at ON users(created_at);

-- GPX tracks indexes
CREATE INDEX idx_gpx_tracks_user_id ON gpx_tracks(user_id);
CREATE INDEX idx_gpx_tracks_status ON gpx_tracks(status);
CREATE INDEX idx_gpx_tracks_created_at ON gpx_tracks(created_at);

-- Track points spatial index
CREATE INDEX idx_track_points_location ON track_points USING GIST(location);
CREATE INDEX idx_track_points_track_id ON track_points(track_id);
CREATE INDEX idx_track_points_time ON track_points(time);

-- Covered areas spatial index
CREATE INDEX idx_covered_areas_geom ON covered_areas USING GIST(geom);
CREATE INDEX idx_covered_areas_user_id ON covered_areas(user_id);
```

### Trigger Functions

```sql
-- Function to update updated_at timestamp
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

-- Add triggers to tables
CREATE TRIGGER update_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_gpx_tracks_updated_at
    BEFORE UPDATE ON gpx_tracks
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_covered_areas_updated_at
    BEFORE UPDATE ON covered_areas
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

### Area Calculation Functions

```sql
-- Function to calculate track buffer area
CREATE OR REPLACE FUNCTION calculate_track_buffer_area(
    track_id UUID,
    buffer_meters DECIMAL DEFAULT 50
)
RETURNS GEOGRAPHY AS $$
DECLARE
    track_buffer GEOGRAPHY;
BEGIN
    SELECT ST_Union(ST_Buffer(location::geography, buffer_meters))
    INTO track_buffer
    FROM track_points
    WHERE track_points.track_id = $1;

    RETURN track_buffer;
END;
$$ LANGUAGE plpgsql;

-- Function to update user's covered area
CREATE OR REPLACE FUNCTION update_user_covered_area(
    user_id_param UUID
)
RETURNS VOID AS $$
DECLARE
    total_area GEOGRAPHY;
BEGIN
    WITH track_areas AS (
        SELECT DISTINCT calculate_track_buffer_area(id) as area
        FROM gpx_tracks
        WHERE user_id = user_id_param
        AND status = 'completed'
    )
    SELECT ST_Union(area)
    INTO total_area
    FROM track_areas;

    INSERT INTO covered_areas (user_id, geom, area_km2)
    VALUES (
        user_id_param,
        total_area::geography,
        ST_Area(total_area) / 1000000  -- Convert to km2
    )
    ON CONFLICT (user_id) DO UPDATE
    SET geom = EXCLUDED.geom,
        area_km2 = EXCLUDED.area_km2,
        updated_at = CURRENT_TIMESTAMP;
END;
$$ LANGUAGE plpgsql;
```

## Application Integration

### Database Connection

Create `backend/app/db/session.py`:

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.core.config import settings

engine = create_engine(
    settings.SQLALCHEMY_DATABASE_URI,
    pool_pre_ping=True,
    pool_size=5,
    max_overflow=10
)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

Create `backend/app/db/base.py`:

```python
from sqlalchemy.ext.declarative import declarative_base
from geoalchemy2 import functions as geofunc

Base = declarative_base()
```

### SQLAlchemy Models

Create `backend/app/models/models.py`:

```python
from sqlalchemy import Column, String, Boolean, DateTime, ForeignKey, Enum, \
                    Integer, Numeric, func
from sqlalchemy.dialects.postgresql import UUID
from geoalchemy2 import Geography
from app.db.base import Base
import enum
import uuid

class UserRole(str, enum.Enum):
    USER = "user"
    ADMIN = "admin"

class TrackStatus(str, enum.Enum):
    PROCESSING = "processing"
    COMPLETED = "completed"
    ERROR = "error"

class User(Base):
    __tablename__ = "users"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    email = Column(String, unique=True, index=True, nullable=False)
    hashed_password = Column(String, nullable=False)
    full_name = Column(String)
    role = Column(Enum(UserRole), nullable=False, default=UserRole.USER)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), server_default=func.now())

class GPXTrack(Base):
    __tablename__ = "gpx_tracks"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    user_id = Column(UUID(as_uuid=True), ForeignKey("users.id"), nullable=False)
    filename = Column(String, nullable=False)
    original_filename = Column(String, nullable=False)
    file_size = Column(Integer, nullable=False)
    status = Column(Enum(TrackStatus), nullable=False, default=TrackStatus.PROCESSING)
    start_time = Column(DateTime(timezone=True))
    end_time = Column(DateTime(timezone=True))
    total_distance = Column(Numeric(10, 2))  # meters
    total_elevation_gain = Column(Numeric(10, 2))  # meters
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), server_default=func.now())

class TrackPoint(Base):
    __tablename__ = "track_points"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    track_id = Column(UUID(as_uuid=True), ForeignKey("gpx_tracks.id"), nullable=False)
    location = Column(Geography("POINT", srid=4326), nullable=False)
    elevation = Column(Numeric(8, 2))  # meters
    time = Column(DateTime(timezone=True))
    heart_rate = Column(Integer)
    speed = Column(Numeric(5, 2))  # m/s
    created_at = Column(DateTime(timezone=True), server_default=func.now())

class CoveredArea(Base):
    __tablename__ = "covered_areas"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    user_id = Column(UUID(as_uuid=True), ForeignKey("users.id"), nullable=False)
    geom = Column(Geography("MULTIPOLYGON", srid=4326), nullable=False)
    area_km2 = Column(Numeric(10, 2), nullable=False)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), server_default=func.now())
```

## Migration Management

### Alembic Setup

```bash
cd backend
poetry run alembic init alembic
```

### Migration Configuration

Update `alembic/env.py`:

```python
from logging.config import fileConfig
from sqlalchemy import engine_from_config
from sqlalchemy import pool
from alembic import context
from app.core.config import settings
from app.db.base import Base
from app.models import models  # Import all models here

config = context.config

if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = Base.metadata

def get_url():
    return settings.SQLALCHEMY_DATABASE_URI

def run_migrations_offline() -> None:
    url = get_url()
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )

    with context.begin_transaction():
        context.run_migrations()

def run_migrations_online() -> None:
    configuration = config.get_section(config.config_ini_section)
    configuration["sqlalchemy.url"] = get_url()
    connectable = engine_from_config(
        configuration,
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(
            connection=connection, target_metadata=target_metadata
        )

        with context.begin_transaction():
            context.run_migrations()

if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

## Utilities and Maintenance

### Database Utilities

Create `backend/app/db/utils.py`:

```python
from sqlalchemy.orm import Session
from geoalchemy2.functions import ST_Area, ST_Buffer, ST_Union
from sqlalchemy import func
from typing import List, Optional
from datetime import datetime
from app.models.models import GPXTrack, TrackPoint, CoveredArea, User

def calculate_user_covered_area(db: Session, user_id: str) -> float:
    """Calculate total covered area for a user in square kilometers."""
    buffer_query = db.query(
        func.ST_Union(
            func.ST_Buffer(TrackPoint.location, 50)  # 50m buffer
        ).label('union_geom')
    ).join(
        GPXTrack, TrackPoint.track_id == GPXTrack.id
    ).filter(
        GPXTrack.user_id == user_id
    ).scalar()

    if buffer_query:
        area_km2 = db.scalar(
            func.ST_Area(buffer_query) / 1000000  # Convert to km²
        )
        return float(area_km2)
    return 0.0

def update_covered_areas(db: Session) -> None:
    """Update covered areas for all users."""
    users = db.query(User).all()
    for user in users:
        area_km2 = calculate_user_covered_area(db, str(user.id))

        covered_area = db.query(CoveredArea).filter(
            CoveredArea.user_id == user.id
        ).first()

        if covered_area:
            covered_area.area_km2 = area_km2
            covered_area.updated_at = datetime.utcnow()
        else:
            new_area = CoveredArea(
                user_id=user.id,
                area_km2=area_km2
            )
            db.add(new_area)

        db.commit()

def cleanup_old_track_points(db: Session, days: int = 90) -> int:
    """Clean up track points older than specified days while preserving track statistics."""
    cutoff_date = datetime.utcnow() - timedelta(days=days)
    deleted = db.query(TrackPoint).filter(
        TrackPoint.created_at < cutoff_date
    ).delete()
    db.commit()
    return deleted
```

### Maintenance Scripts

Create `backend/scripts/db_maintenance.py`:

```python
#!/usr/bin/env python
import click
from pathlib import Path
from datetime import datetime
import subprocess
from app.core.config import settings

@click.group()
def cli():
    """GPX Tracker Database Maintenance Tools"""
    pass

@cli.command()
@click.option('--output-dir', default='./backups', help='Backup output directory')
def backup(output_dir):
    """Create a database backup."""
    Path(output_dir).mkdir(parents=True, exist_ok=True)
    timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
    filename = f"gpx_tracker_backup_{timestamp}.sql"
    output_path = Path(output_dir) / filename

    cmd = [
        'pg_dump',
        '-h', settings.POSTGRES_SERVER,
        '-U', settings.POSTGRES_USER,
        '-d', settings.POSTGRES_DB,
        '-F', 'c',  # Custom format
        '-f', str(output_path)
    ]

    subprocess.run(cmd, check=True, env={'PGPASSWORD': settings.POSTGRES_PASSWORD})
    click.echo(f"Backup created successfully: {output_path}")

@cli.command()
@click.argument('backup-file')
def restore(backup_file):
    """Restore database from backup."""
    if not Path(backup_file).exists():
        click.echo(f"Backup file not found: {backup_file}", err=True)
        return

    cmd = [
        'pg_restore',
        '-h', settings.POSTGRES_SERVER,
        '-U', settings.POSTGRES_USER,
        '-d', settings.POSTGRES_DB,
        '-c',  # Clean (drop) database objects before recreating
        str(backup_file)
    ]

    subprocess.run(cmd, check=True, env={'PGPASSWORD': settings.POSTGRES_PASSWORD})
    click.echo("Database restored successfully")

if __name__ == '__main__':
    cli()
```

### Optimization Procedures

```sql
-- Analyze tables to update statistics
ANALYZE users;
ANALYZE gpx_tracks;
ANALYZE track_points;
ANALYZE covered_areas;

-- Vacuum tables to reclaim space and update statistics
VACUUM ANALYZE users;
VACUUM ANALYZE gpx_tracks;
VACUUM ANALYZE track_points;
VACUUM ANALYZE covered_areas;

-- Reindex tables to optimize index performance
REINDEX TABLE users;
REINDEX TABLE gpx_tracks;
REINDEX TABLE track_points;
REINDEX TABLE covered_areas;
```

## Production Configuration

### Maintenance Schedule

Create a crontab file for automated maintenance:

```bash
# /etc/cron.d/gpx-tracker-maintenance

# Daily backup at 2 AM
0 2 * * * postgres /app/scripts/db_maintenance.py backup

# Weekly vacuum and cleanup on Sunday at 3 AM
0 3 * * 0 postgres vacuumdb -z -d gpx_tracker

# Monthly reindex on the 1st at 4 AM
0 4 1 * * postgres /app/scripts/db_maintenance.py reindex
```

### Connection Pool Settings

Update `backend/app/db/session.py`:

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.core.config import settings

engine = create_engine(
    settings.SQLALCHEMY_DATABASE_URI,
    pool_size=10,  # Maximum number of permanent connections
    max_overflow=20,  # Maximum number of additional connections
    pool_timeout=30,  # Seconds to wait before giving up on getting a connection
    pool_recycle=1800,  # Recycle connections after 30 minutes
    echo=settings.SQL_ECHO  # Set to True for development/debugging
)

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
```

## Setup Verification

### Initial Verification

```bash
# Start PostgreSQL container
docker-compose up -d postgres

# Run migrations
cd backend
poetry run alembic upgrade head

# Connect to database
docker exec -it gpx-tracker-postgres psql -U postgres -d gpx_tracker

# Check tables
\dt

# Check PostGIS extension
SELECT PostGIS_version();
```

### Maintenance Guidelines

1. Always use spatial indexes for geographical queries
2. Regularly monitor and update statistics
3. Implement proper backup procedures
4. Use connection pooling in production
5. Follow the maintenance schedule
6. Monitor query performance regularly
7. Keep PostGIS and PostgreSQL versions up to date

**Note**: The next step would be implementing the API endpoints that interact with this database structure.
