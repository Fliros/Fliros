# Database Setup - Part 1: Initial Configuration

1. Verify PostgreSQL and PostGIS Installation in Docker:

   ```bash
   # Connect to PostgreSQL container
   docker exec -it gpx-tracker-postgres psql -U postgres

   # Verify PostGIS extension
   SELECT PostGIS_version();

   # Create database if not exists
   CREATE DATABASE gpx_tracker;
   \\c gpx_tracker

   # Enable PostGIS extension
   CREATE EXTENSION IF NOT EXISTS postgis;
   CREATE EXTENSION IF NOT EXISTS \"uuid-ossp\";
   ```

2. Set up Initial Schema:

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

# Database Setup - Part 2: Indexes and Functions

3. Create Indexes:

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

4. Create Updated At Trigger:

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

5. Create Area Calculation Functions:

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
       -- Create buffer around track points and union them
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
       -- Calculate total covered area for user
       WITH track_areas AS (
           SELECT DISTINCT calculate_track_buffer_area(id) as area
           FROM gpx_tracks
           WHERE user_id = user_id_param
           AND status = 'completed'
       )
       SELECT ST_Union(area)
       INTO total_area
       FROM track_areas;

       -- Update or insert into covered_areas
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

# Database Setup - Part 3: Application Integration

6. Database Connection Setup:
   A. backend/app/db/session.py:

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

   B. backend/app/db/base.py:

   ```python
   from sqlalchemy.ext.declarative import declarative_base
   from geoalchemy2 import functions as geofunc

   Base = declarative_base()
   ```

7. SQLAlchemy Models:
   backend/app/models/models.py:

   ```python
   from sqlalchemy import Column, String, Boolean, DateTime, ForeignKey, Enum, \\
                      Integer, Numeric, func
   from sqlalchemy.dialects.postgresql import UUID
   from geoalchemy2 import Geography
   from app.db.base import Base
   import enum
   import uuid

   class UserRole(str, enum.Enum):
       USER = \"user\"
       ADMIN = \"admin\"

   class TrackStatus(str, enum.Enum):
       PROCESSING = \"processing\"
       COMPLETED = \"completed\"
       ERROR = \"error\"

   class User(Base):
       __tablename__ = \"users\"

       id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
       email = Column(String, unique=True, index=True, nullable=False)
       hashed_password = Column(String, nullable=False)
       full_name = Column(String)
       role = Column(Enum(UserRole), nullable=False, default=UserRole.USER)
       is_active = Column(Boolean, default=True)
       created_at = Column(DateTime(timezone=True), server_default=func.now())
       updated_at = Column(DateTime(timezone=True), server_default=func.now())

   class GPXTrack(Base):
       __tablename__ = \"gpx_tracks\"

       id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
       user_id = Column(UUID(as_uuid=True), ForeignKey(\"users.id\"), nullable=False)
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
       __tablename__ = \"track_points\"

       id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
       track_id = Column(UUID(as_uuid=True), ForeignKey(\"gpx_tracks.id\"), nullable=False)
       location = Column(Geography(\"POINT\", srid=4326), nullable=False)
       elevation = Column(Numeric(8, 2))  # meters
       time = Column(DateTime(timezone=True))
       heart_rate = Column(Integer)
       speed = Column(Numeric(5, 2))  # m/s
       created_at = Column(DateTime(timezone=True), server_default=func.now())

   class CoveredArea(Base):
       __tablename__ = \"covered_areas\"

       id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
       user_id = Column(UUID(as_uuid=True), ForeignKey(\"users.id\"), nullable=False)
       geom = Column(Geography(\"MULTIPOLYGON\", srid=4326), nullable=False)
       area_km2 = Column(Numeric(10, 2), nullable=False)
       created_at = Column(DateTime(timezone=True), server_default=func.now())
       updated_at = Column(DateTime(timezone=True), server_default=func.now())
   ```

# Database Setup - Part 4: Migrations and Utilities

8. Set up Alembic Migrations:
   A. Initialize Alembic:

   ```bash
   cd backend
   poetry run alembic init alembic
   ```

   B. Update alembic/env.py:

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
           dialect_opts={\"paramstyle\": \"named\"},
       )

       with context.begin_transaction():
           context.run_migrations()

   def run_migrations_online() -> None:
       configuration = config.get_section(config.config_ini_section)
       configuration[\"sqlalchemy.url\"] = get_url()
       connectable = engine_from_config(
           configuration,
           prefix=\"sqlalchemy.\",
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

   C. Create alembic.ini configuration:

   ```ini
   [alembic]
   script_location = alembic
   sqlalchemy.url = driver://user:pass@localhost/dbname

   [loggers]
   keys = root,sqlalchemy,alembic

   [handlers]
   keys = console

   [formatters]
   keys = generic

   [logger_root]
   level = WARN
   handlers = console
   qualname =

   [logger_sqlalchemy]
   level = WARN
   handlers =
   qualname = sqlalchemy.engine

   [logger_alembic]
   level = INFO
   handlers =
   qualname = alembic

   [handler_console]
   class = StreamHandler
   args = (sys.stderr,)
   level = NOTSET
   formatter = generic

   [formatter_generic]
   format = %(levelname)-5.5s [%(name)s] %(message)s
   datefmt = %H:%M:%S
   ```

   D. Create initial migration:

   ```bash
   poetry run alembic revision --autogenerate -m \"initial_migration\"
   poetry run alembic upgrade head
   ```

# vDatabase Setup - Part 5: Utilities and Maintenance

9. Database Utility Functions:
   backend/app/db/utils.py:

   ```python
   from sqlalchemy.orm import Session
   from geoalchemy2.functions import ST_Area, ST_Buffer, ST_Union
   from sqlalchemy import func
   from typing import List, Optional
   from datetime import datetime
   from app.models.models import GPXTrack, TrackPoint, CoveredArea, User

   def calculate_user_covered_area(db: Session, user_id: str) -> float:
       \"\"\"
       Calculate total covered area for a user in square kilometers.
       \"\"\"
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
       \"\"\"
       Update covered areas for all users.
       \"\"\"
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
       \"\"\"
       Clean up track points older than specified days while preserving track statistics.
       \"\"\"
       cutoff_date = datetime.utcnow() - timedelta(days=days)
       deleted = db.query(TrackPoint).filter(
           TrackPoint.created_at < cutoff_date
       ).delete()
       db.commit()
       return deleted
   ```

10. Database Maintenance Scripts:
    backend/scripts/db_maintenance.py:

    ```python
    #!/usr/bin/env python
    import click
    from pathlib import Path
    from datetime import datetime
    import subprocess
    from app.core.config import settings

    @click.group()
    def cli():
        \"\"\"GPX Tracker Database Maintenance Tools\"\"\"
        pass

    @cli.command()
    @click.option('--output-dir', default='./backups', help='Backup output directory')
    def backup(output_dir):
        \"\"\"Create a database backup.\"\"\"
        Path(output_dir).mkdir(parents=True, exist_ok=True)
        timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
        filename = f\"gpx_tracker_backup_{timestamp}.sql\"
        output_path = Path(output_dir) / filename

        cmd = [
            'pg_dump',
            '-h', settings.POSTGRES_SERVER,
            '-U', settings.POSTGRES_USER,
            '-d', settings.POSTGRES_DB,
            '-F', 'c',  # Custom format
            '-f', str(output_path)
        ]

        try:
            subprocess.run(cmd, check=True, env={'PGPASSWORD': settings.POSTGRES_PASSWORD})
            click.echo(f\"Backup created successfully: {output_path}\")
        except subprocess.CalledProcessError as e:
            click.echo(f\"Backup failed: {e}\", err=True)

    @cli.command()
    @click.argument('backup-file')
    def restore(backup_file):
        \"\"\"Restore database from backup.\"\"\"
        if not Path(backup_file).exists():
            click.echo(f\"Backup file not found: {backup_file}\", err=True)
            return

        cmd = [
            'pg_restore',
            '-h', settings.POSTGRES_SERVER,
            '-U', settings.POSTGRES_USER,
            '-d', settings.POSTGRES_DB,
            '-c',  # Clean (drop) database objects before recreating
            str(backup_file)
        ]

        try:
            subprocess.run(cmd, check=True, env={'PGPASSWORD': settings.POSTGRES_PASSWORD})
            click.echo(\"Database restored successfully\")
        except subprocess.CalledProcessError as e:
            click.echo(f\"Restore failed: {e}\", err=True)

    if __name__ == '__main__':
        cli()
    ```

# Database Setup - Part 6: Optimization and Maintenance Procedures

11. Database Optimization Recommendations:

A. Regular Maintenance Tasks:

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

B. Performance Monitoring Queries:

```sql
-- Check table sizes
SELECT
    relname as table_name,
    pg_size_pretty(pg_total_relation_size(relid)) as total_size,
    pg_size_pretty(pg_relation_size(relid)) as data_size,
    pg_size_pretty(pg_total_relation_size(relid) - pg_relation_size(relid))
        as external_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;

-- Check index usage
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan as number_of_scans,
    idx_tup_read as tuples_read,
    idx_tup_fetch as tuples_fetched
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;
```

12. Maintenance Schedule:

A. Daily Tasks:

- Backup database
- Monitor disk space
- Check for long-running queries

B. Weekly Tasks:

- Run VACUUM ANALYZE on all tables
- Update user covered areas calculations
- Clean up old track points

C. Monthly Tasks:

- Reindex tables
- Review and optimize slow queries
- Check and update PostGIS statistics

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

13. Connection Pool Configuration:

Update backend/app/db/session.py with optimal pool settings:

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

# Database Setup Complete

The database setup is now complete with the following components:

1. Schema Design:

   - Users and authentication
   - GPX tracks and points with spatial data
   - Covered areas with PostGIS support
   - Efficient indexes and constraints

2. Integration Tools:

   - SQLAlchemy models
   - Alembic migrations
   - Database utility functions
   - Maintenance scripts

3. Performance Optimizations:
   - Spatial indexes for geographical queries
   - Connection pooling
   - Regular maintenance procedures
   - Monitoring queries

To get started:

1. Initialize the database:

   ```bash
   # Start PostgreSQL container
   docker-compose up -d postgres

   # Run migrations
   cd backend
   poetry run alembic upgrade head
   ```

2. Verify setup:

   ```bash
   # Connect to database
   docker exec -it gpx-tracker-postgres psql -U postgres -d gpx_tracker

   # Check tables
   \\dt

   # Check PostGIS extension
   SELECT PostGIS_version();
   ```

3. Set up maintenance schedule:
   - Configure cron jobs for regular maintenance
   - Implement backup procedures
   - Monitor database performance

Key Considerations:

1. Always use spatial indexes for geographical queries
2. Regularly monitor and update statistics
3. Implement proper backup procedures
4. Use connection pooling in production
5. Follow the maintenance schedule

The database is now ready for the GPX tracking application. The next step would be implementing the API endpoints that interact with this database structure.
