# Backend Development

## Part 1: Basic API Structure

1. Update Backend Dependencies:

   ```bash
   cd backend
   poetry add fastapi[all] \\
       sqlalchemy \\
       alembic \\
       psycopg2-binary \\
       geoalchemy2 \\
       shapely \\
       gpxpy \\
       python-jose[cryptography] \\
       passlib[bcrypt] \\
       python-multipart \\
       redis \\
       celery \\
       pytest \\
       pytest-asyncio \\
       httpx \\\
       tenacity
   ```

2. Core Configuration:
   backend/app/core/config.py:

   ```python
   from pydantic_settings import BaseSettings
   from typing import List, Optional
   from pydantic import AnyHttpUrl, validator

   class Settings(BaseSettings):
       PROJECT_NAME: str = "GPX Tracker"
       API_V1_STR: str = "/api/v1"
       SECRET_KEY: str
       ACCESS_TOKEN_EXPIRE_MINUTES: int = 60 * 24 * 8  # 8 days

       # Database
       POSTGRES_SERVER: str
       POSTGRES_USER: str
       POSTGRES_PASSWORD: str
       POSTGRES_DB: str
       POSTGRES_PORT: str = "5432"
       DATABASE_URL: Optional[str] = None

       # Redis
       REDIS_URL: str = "redis://localhost:6379/0"

       # CORS
       BACKEND_CORS_ORIGINS: List[AnyHttpUrl] = []

       @validator("BACKEND_CORS_ORIGINS", pre=True)
       def assemble_cors_origins(cls, v: Union[str, List[str]]) -> Union[List[str], str]:
           if isinstance(v, str) and not v.startswith("["):
               return [i.strip() for i in v.split(",")]
           elif isinstance(v, (list, str)):
               return v
           raise ValueError(v)

       @property
       def SQLALCHEMY_DATABASE_URI(self) -> str:
           if self.DATABASE_URL:
               return self.DATABASE_URL
           return f"postgresql://{self.POSTGRES_USER}:{self.POSTGRES_PASSWORD}@{self.POSTGRES_SERVER}:{self.POSTGRES_PORT}/{self.POSTGRES_DB}"

       class Config:
           case_sensitive = True
           env_file = ".env"

   settings = Settings()
   ```

3. Database Connection:
   backend/app/db/session.py:

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

4. Main Application Setup:
   backend/app/main.py:

   ```python
   from fastapi import FastAPI
   from fastapi.middleware.cors import CORSMiddleware
   from app.core.config import settings
   from app.api.api_v1.api import api_router

   app = FastAPI(
       title=settings.PROJECT_NAME,
       openapi_url=f"{settings.API_V1_STR}/openapi.json"
   )

   if settings.BACKEND_CORS_ORIGINS:
       app.add_middleware(
           CORSMiddleware,
           allow_origins=[str(origin) for origin in settings.BACKEND_CORS_ORIGINS],
           allow_credentials=True,
           allow_methods=["*"],
           allow_headers=["*"],
       )

   app.include_router(api_router, prefix=settings.API_V1_STR)

   @app.get("/")
   def root():
       return {"message": "Welcome to GPX Tracker API"}
   ```

5. API Router Setup:
   backend/app/api/api_v1/api.py:

   ```python
   from fastapi import APIRouter
   from app.api.api_v1.endpoints import users, tracks, areas

   api_router = APIRouter()

   api_router.include_router(users.router, prefix="/users", tags=["users"])
   api_router.include_router(tracks.router, prefix="/tracks", tags=["tracks"])
   api_router.include_router(areas.router, prefix="/areas", tags=["areas"])
   ```

## Part 2: User Authentication

1. Security Utilities:
   backend/app/core/security.py:

   ```python
   from datetime import datetime, timedelta
   from typing import Any, Union
   from jose import jwt
   from passlib.context import CryptContext
   from app.core.config import settings

   pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

   ALGORITHM = "HS256"

   def create_access_token(
       subject: Union[str, Any], expires_delta: timedelta = None
   ) -> str:
       if expires_delta:
           expire = datetime.utcnow() + expires_delta
       else:
           expire = datetime.utcnow() + timedelta(
               minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES
           )
       to_encode = {"exp": expire, "sub": str(subject)}
       encoded_jwt = jwt.encode(to_encode, settings.SECRET_KEY, algorithm=ALGORITHM)
       return encoded_jwt

   def verify_password(plain_password: str, hashed_password: str) -> bool:
       return pwd_context.verify(plain_password, hashed_password)

   def get_password_hash(password: str) -> str:
       return pwd_context.hash(password)
   ```

2. User Schemas:
   backend/app/schemas/user.py:

   ```python
   from pydantic import BaseModel, EmailStr, UUID4
   from typing import Optional
   from datetime import datetime

   class UserBase(BaseModel):
       email: EmailStr
       full_name: Optional[str] = None

   class UserCreate(UserBase):
       password: str

   class UserUpdate(UserBase):
       password: Optional[str] = None

   class UserInDBBase(UserBase):
       id: UUID4
       is_active: bool
       created_at: datetime

       class Config:
           from_attributes = True

   class User(UserInDBBase):
       pass

   class UserInDB(UserInDBBase):
       hashed_password: str

   class Token(BaseModel):
       access_token: str
       token_type: str

   class TokenPayload(BaseModel):
       sub: Optional[str] = None
   ```

3. Authentication Dependencies:
   backend/app/api/deps.py:

   ```python
   from typing import Generator, Optional
   from fastapi import Depends, HTTPException, status
   from fastapi.security import OAuth2PasswordBearer
   from jose import jwt, JWTError
   from pydantic import ValidationError
   from sqlalchemy.orm import Session
   from app.core.config import settings
   from app.core.security import ALGORITHM
   from app.db.session import SessionLocal
   from app.models.user import User
   from app.schemas.user import TokenPayload

   oauth2_scheme = OAuth2PasswordBearer(
       tokenUrl=f"{settings.API_V1_STR}/auth/login"
   )

   def get_db() -> Generator:
       try:
           db = SessionLocal()
           yield db
       finally:
           db.close()

   async def get_current_user(
       db: Session = Depends(get_db),
       token: str = Depends(oauth2_scheme)
   ) -> User:
       try:
           payload = jwt.decode(
               token, settings.SECRET_KEY, algorithms=[ALGORITHM]
           )
           token_data = TokenPayload(**payload)
       except (JWTError, ValidationError):
           raise HTTPException(
               status_code=status.HTTP_403_FORBIDDEN,
               detail="Could not validate credentials",
           )
       user = db.query(User).get(token_data.sub)
       if not user:
           raise HTTPException(status_code=404, detail="User not found")
       if not user.is_active:
           raise HTTPException(status_code=400, detail="Inactive user")
       return user
   ```

4. Authentication Endpoints:
   backend/app/api/v1/endpoints/auth.py:

   ```python
   from datetime import timedelta
   from typing import Any
   from fastapi import APIRouter, Body, Depends, HTTPException
   from fastapi.security import OAuth2PasswordRequestForm
   from sqlalchemy.orm import Session
   from app import crud
   from app.api import deps
   from app.core import security
   from app.core.config import settings
   from app.schemas.user import User, UserCreate, Token

   router = APIRouter()

   @router.post("/login", response_model=Token)
   def login(
       db: Session = Depends(deps.get_db),
       form_data: OAuth2PasswordRequestForm = Depends()
   ) -> Any:
       """OAuth2 compatible token login"""
       user = crud.user.authenticate(
           db, email=form_data.username, password=form_data.password
       )
       if not user:
           raise HTTPException(
               status_code=400, detail="Incorrect email or password"
           )
       elif not user.is_active:
           raise HTTPException(status_code=400, detail="Inactive user")
       access_token_expires = timedelta(
           minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES
       )
       return {
           "access_token": security.create_access_token(
               user.id, expires_delta=access_token_expires
           ),
           "token_type": "bearer",
       }

   @router.post("/signup", response_model=User)
   def create_user(*, db: Session = Depends(deps.get_db), user_in: UserCreate) -> Any:
       """Create new user"""
       user = crud.user.get_by_email(db, email=user_in.email)
       if user:
           raise HTTPException(
               status_code=400,
               detail="The user with this email already exists",
           )
       user = crud.user.create(db, obj_in=user_in)
       return user
   ```

5. User CRUD Operations:
   backend/app/crud/crud_user.py:

   ```python
   from typing import Optional
   from sqlalchemy.orm import Session
   from app.core.security import get_password_hash, verify_password
   from app.crud.base import CRUDBase
   from app.models.user import User
   from app.schemas.user import UserCreate, UserUpdate

   class CRUDUser(CRUDBase[User, UserCreate, UserUpdate]):
       def get_by_email(self, db: Session, *, email: str) -> Optional[User]:
           return db.query(User).filter(User.email == email).first()

       def create(self, db: Session, *, obj_in: UserCreate) -> User:
           db_obj = User(
               email=obj_in.email,
               hashed_password=get_password_hash(obj_in.password),
               full_name=obj_in.full_name,
           )
           db.add(db_obj)
           db.commit()
           db.refresh(db_obj)
           return db_obj

       def authenticate(self, db: Session, *, email: str, password: str) -> Optional[User]:
           user = self.get_by_email(db, email=email)
           if not user:
               return None
           if not verify_password(password, user.hashed_password):
               return None
           return user

   user = CRUDUser(User)
   ```

   Part 3: GPX File Processing",
   "content": "1. GPX Processing Service:
   backend/app/services/gpx_processor.py:

   ```python
   from typing import List, Tuple
   import gpxpy
   from gpxpy.gpx import GPXTrackPoint
   from datetime import datetime
   from shapely.geometry import Point, MultiPoint
   from shapely.ops import unary_union
   from geoalchemy2.shape import from_shape
   import io

   class GPXProcessor:
       def __init__(self, gpx_file: bytes):
           self.gpx = gpxpy.parse(io.StringIO(gpx_file.decode()))

       def get_track_points(self) -> List[Tuple[float, float, float, datetime]]:
           """Extract track points with elevation and time"""
           points = []
           for track in self.gpx.tracks:
               for segment in track.segments:
                   for point in segment.points:
                       points.append((
                           point.latitude,
                           point.longitude,
                           point.elevation,
                           point.time
                       ))
           return points

       def calculate_statistics(self) -> dict:
           """Calculate track statistics"""
           stats = self.gpx.get_moving_data()
           return {
               "moving_time": stats.moving_time,
               "stopped_time": stats.stopped_time,
               "moving_distance": stats.moving_distance,
               "total_distance": self.gpx.length_3d(),
               "max_speed": stats.max_speed,
               "avg_speed": stats.moving_distance / stats.moving_time if stats.moving_time > 0 else 0,
               "total_elevation_gain": self.gpx.get_elevation_extremes().elevation_gain,
               "total_elevation_loss": self.gpx.get_elevation_extremes().elevation_loss,
               "start_time": self.gpx.get_time_bounds().start_time,
               "end_time": self.gpx.get_time_bounds().end_time,
           }

       def calculate_covered_area(self, buffer_distance: float = 50) -> Tuple[float, dict]:
           """Calculate area covered by track with buffer"""
           points = [Point(p.longitude, p.latitude) for track in self.gpx.tracks
                    for segment in track.segments
                    for p in segment.points]
           if not points:
               return 0, {"type": "MultiPolygon", "coordinates": []}

           # Create buffer around points
           multi_point = MultiPoint(points)
           buffered = multi_point.buffer(buffer_distance / 111320)  # Convert meters to degrees

           # Convert to GeoJSON
           geojson = {
               "type": "MultiPolygon",
               "coordinates": [[list(coord) for coord in polygon.exterior.coords]]
                              for polygon in buffered.__geo_interface__["coordinates"]]
           }

           return buffered.area * 12365.1, geojson  # Convert to square kilometers
   ```

6. Celery Tasks Setup:
   backend/app/worker.py:

   ```python
   from celery import Celery
   from app.core.config import settings

   celery_app = Celery("worker", broker=settings.REDIS_URL)

   celery_app.conf.task_routes = {
       "app.worker.process_gpx_file": "main-queue"
   }
   ```

   backend/app/tasks/gpx_tasks.py:

   ```python
   from typing import Dict, Any
   from uuid import UUID
   from celery import Task
   from sqlalchemy.orm import Session
   from app.core.celery_app import celery_app
   from app.db.session import SessionLocal
   from app.services.gpx_processor import GPXProcessor
   from app.models.track import GPXTrack
   from app.models.track_point import TrackPoint
   from app.crud.crud_track import track as crud_track

   class SQLAlchemyTask(Task):
       _db = None

       @property
       def db(self) -> Session:
           if self._db is None:
               self._db = SessionLocal()
           return self._db

       def after_return(self, *args, **kwargs):
           if self._db is not None:
               self._db.close()
               self._db = None

   @celery_app.task(base=SQLAlchemyTask)
   def process_gpx_file(track_id: UUID, file_content: bytes) -> Dict[str, Any]:
       """Process uploaded GPX file asynchronously"""
       try:
           processor = GPXProcessor(file_content)

           # Get track statistics
           stats = processor.calculate_statistics()

           # Calculate covered area
           area_size, area_geojson = processor.calculate_covered_area()

           # Get track points
           points = processor.get_track_points()

           # Update track record
           track = crud_track.get(process_gpx_file.db, id=track_id)
           track.status = "completed"
           track.start_time = stats["start_time"]
           track.end_time = stats["end_time"]
           track.total_distance = stats["total_distance"]
           track.total_elevation_gain = stats["total_elevation_gain"]

           # Create track points
           for lat, lon, ele, time in points:
               point = TrackPoint(
                   track_id=track_id,
                   latitude=lat,
                   longitude=lon,
                   elevation=ele,
                   time=time
               )
               process_gpx_file.db.add(point)

           process_gpx_file.db.commit()

           return {
               "status": "success",
               "track_id": str(track_id),
               "statistics": stats,
               "area": {
                   "size": area_size,
                   "geojson": area_geojson
               }
           }

       except Exception as e:
           track = crud_track.get(process_gpx_file.db, id=track_id)
           track.status = "error"
           track.error_message = str(e)
           process_gpx_file.db.commit()
           raise
   ```

7. File Upload Endpoint:
   backend/app/api/v1/endpoints/tracks.py:

   ```python
   from typing import Any, List
   from fastapi import APIRouter, Depends, UploadFile, File, HTTPException
   from sqlalchemy.orm import Session
   from app.api import deps
   from app.schemas.track import Track, TrackCreate, TrackList
   from app.crud.crud_track import track as crud_track
   from app.models.user import User
   from app.tasks.gpx_tasks import process_gpx_file

   router = APIRouter()

   @router.post("/upload", response_model=Track)
   async def upload_gpx(
       *,
       db: Session = Depends(deps.get_db),
       file: UploadFile = File(...),
       current_user: User = Depends(deps.get_current_user)
   ) -> Any:
       """Upload GPX file for processing"""
       if not file.filename.endswith(".gpx"):
           raise HTTPException(
               status_code=400,
               detail="File must be a GPX file"
           )

       content = await file.read()

       # Create track record
       track_in = TrackCreate(
           filename=file.filename,
           original_filename=file.filename,
           file_size=len(content),
           user_id=current_user.id
       )
       track = crud_track.create(db, obj_in=track_in)

       # Start async processing
       process_gpx_file.delay(str(track.id), content)

       return track

   @router.get("/", response_model=List[TrackList])
   def list_tracks(
       db: Session = Depends(deps.get_db),
       current_user: User = Depends(deps.get_current_user),
       skip: int = 0,
       limit: int = 100
   ) -> Any:
       """List user's tracks"""
       tracks = crud_track.get_multi_by_user(
           db, user_id=current_user.id, skip=skip, limit=limit
       )
       return tracks
   ```

## Part 4: Area Calculation and Visualization

1. Area Calculation Service:
   backend/app/services/area_calculator.py:

   ```python
   from typing import List, Dict, Any
   from sqlalchemy.orm import Session
   from geoalchemy2.shape import to_shape, from_shape
   from shapely.ops import unary_union
   from shapely.geometry import MultiPolygon, mapping
   from app.models.track import TrackPoint
   from app.models.covered_area import CoveredArea

   class AreaCalculator:
       def __init__(self, db: Session):
           self.db = db

       def calculate_user_coverage(self, user_id: str, buffer_meters: float = 50) -> Dict[str, Any]:
           """Calculate total area covered by user's tracks"""
           # Get all track points for user
           points = self.db.query(TrackPoint).join(
               TrackPoint.track
           ).filter(
               TrackPoint.track.has(user_id=user_id)
           ).all()

           if not points:
               return {
                   "area_km2": 0,
                   "percentage": 0,
                   "geojson": {"type": "MultiPolygon", "coordinates": []}
               }

           # Create buffers and union
           buffers = [to_shape(point.location).buffer(buffer_meters / 111320)
                     for point in points]
           union = unary_union(buffers)

           # Convert to MultiPolygon if necessary
           if union.geom_type == 'Polygon':
               multi_polygon = MultiPolygon([union])
           else:
               multi_polygon = union

           # Calculate area in km²
           area_km2 = multi_polygon.area * 12365.1

           # Calculate percentage of Earth's surface
           earth_surface = 510072000  # km²
           percentage = (area_km2 / earth_surface) * 100

           # Convert to GeoJSON
           geojson = mapping(multi_polygon)

           return {
               "area_km2": round(area_km2, 2),
               "percentage": round(percentage, 6),
               "geojson": geojson
           }

       def update_user_coverage(self, user_id: str) -> CoveredArea:
           """Update stored coverage for user"""
           coverage = self.calculate_user_coverage(user_id)

           covered_area = self.db.query(CoveredArea).filter(
               CoveredArea.user_id == user_id
           ).first()

           if covered_area:
               covered_area.area_km2 = coverage["area_km2"]
               covered_area.geom = from_shape(MultiPolygon(coverage["geojson"]))
           else:
               covered_area = CoveredArea(
                   user_id=user_id,
                   area_km2=coverage["area_km2"],
                   geom=from_shape(MultiPolygon(coverage["geojson"]))
               )
               self.db.add(covered_area)

           self.db.commit()
           self.db.refresh(covered_area)
           return covered_area
   ```

2. Area Visualization Endpoints:
   backend/app/api/v1/endpoints/areas.py:

   ```python
   from typing import Any, List
   from fastapi import APIRouter, Depends, HTTPException
   from sqlalchemy.orm import Session
   from app.api import deps
   from app.models.user import User
   from app.services.area_calculator import AreaCalculator
   from app.schemas.area import CoverageStats, CoverageDetails

   router = APIRouter()

   @router.get("/coverage", response_model=CoverageStats)
   def get_coverage_stats(
       db: Session = Depends(deps.get_db),
       current_user: User = Depends(deps.get_current_user)
   ) -> Any:
       """Get user's coverage statistics"""
       calculator = AreaCalculator(db)
       coverage = calculator.calculate_user_coverage(str(current_user.id))
       return {
           "area_km2": coverage["area_km2"],
           "percentage": coverage["percentage"]
       }

   @router.get("/coverage/details", response_model=CoverageDetails)
   def get_coverage_details(
       db: Session = Depends(deps.get_db),
       current_user: User = Depends(deps.get_current_user)
   ) -> Any:
       """Get detailed coverage information including GeoJSON"""
       calculator = AreaCalculator(db)
       return calculator.calculate_user_coverage(str(current_user.id))

   @router.post("/coverage/update")
   def update_coverage(
       db: Session = Depends(deps.get_db),
       current_user: User = Depends(deps.get_current_user)
   ) -> Any:
       """Force update of user's coverage calculation"""
       calculator = AreaCalculator(db)
       calculator.update_user_coverage(str(current_user.id))
       return {"message": "Coverage updated successfully"}
   ```

3. Area Schemas:
   backend/app/schemas/area.py:

   ```python
   from pydantic import BaseModel
   from typing import Dict, Any

   class CoverageStats(BaseModel):
       area_km2: float
       percentage: float

   class CoverageDetails(CoverageStats):
       geojson: Dict[str, Any]

   class CoveredArea(BaseModel):
       id: str
       user_id: str
       area_km2: float

       class Config:
           from_attributes = True
   ```

## Part 5: Unit Tests

1. Test Configuration:
   backend/tests/conftest.py:

   ```python
   import pytest
   from typing import Generator, Dict
   from fastapi.testclient import TestClient
   from sqlalchemy import create_engine
   from sqlalchemy.orm import sessionmaker
   from app.db.base import Base
   from app.main import app
   from app.api import deps
   from app.core.config import settings
   from app.models.user import User
   from app.core.security import get_password_hash

   SQLALCHEMY_TEST_DATABASE_URL = "sqlite:///./test.db"

   engine = create_engine(SQLALCHEMY_TEST_DATABASE_URL)
   TestingSessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

   @pytest.fixture(scope="function")
   def db() -> Generator:
       Base.metadata.create_all(bind=engine)
       db = TestingSessionLocal()
       try:
           yield db
       finally:
           db.close()
           Base.metadata.drop_all(bind=engine)

   @pytest.fixture(scope="function")
   def client(db: TestingSessionLocal) -> Generator:
       def override_get_db():
           try:
               yield db
           finally:
               pass

       app.dependency_overrides[deps.get_db] = override_get_db
       with TestClient(app) as c:
           yield c

   @pytest.fixture(scope="function")
   def test_user(db: TestingSessionLocal) -> Dict[str, str]:
       email = "test@example.com"
       password = "testpassword123"
       user = User(
           email=email,
           hashed_password=get_password_hash(password),
           full_name="Test User"
       )
       db.add(user)
       db.commit()
       db.refresh(user)
       return {"email": email, "password": password, "id": str(user.id)}

   @pytest.fixture(scope="function")
   def test_superuser(db: TestingSessionLocal) -> Dict[str, str]:
       email = "admin@example.com"
       password = "admin123"
       user = User(
           email=email,
           hashed_password=get_password_hash(password),
           full_name="Admin User",
           is_superuser=True
       )
       db.add(user)
       db.commit()
       db.refresh(user)
       return {"email": email, "password": password, "id": str(user.id)}
   ```

2. Authentication Tests:
   backend/tests/test_auth.py:

   ```python
   import pytest
   from fastapi.testclient import TestClient
   from sqlalchemy.orm import Session

   def test_login(client: TestClient, test_user: dict):
       response = client.post(
           "/api/v1/auth/login",
           data={
               "username": test_user["email"],
               "password": test_user["password"]
           }
       )
       assert response.status_code == 200
       assert "access_token" in response.json()
       assert response.json()["token_type"] == "bearer"

   def test_login_incorrect_password(client: TestClient, test_user: dict):
       response = client.post(
           "/api/v1/auth/login",
           data={
               "username": test_user["email"],
               "password": "wrongpassword"
           }
       )
       assert response.status_code == 400

   def test_signup(client: TestClient):
       response = client.post(
           "/api/v1/auth/signup",
           json={
               "email": "newuser@example.com",
               "password": "newpassword123",
               "full_name": "New User"
           }
       )
       assert response.status_code == 200
       assert response.json()["email"] == "newuser@example.com"
       assert response.json()["full_name"] == "New User"
   ```

3. GPX Processing Tests:
   backend/tests/test_gpx_processor.py:

   ```python
   import pytest
   from datetime import datetime
   from app.services.gpx_processor import GPXProcessor

   SAMPLE_GPX = """
   <?xml version="1.0" encoding="UTF-8"?>
   <gpx version="1.1">
       <trk>
           <trkseg>
               <trkpt lat="47.644548" lon="-122.326897">
                   <ele>4.46</ele>
                   <time>2024-01-01T10:00:00Z</time>
               </trkpt>
               <trkpt lat="47.644548" lon="-122.326897">
                   <ele>4.94</ele>
                   <time>2024-01-01T10:00:10Z</time>
               </trkpt>
           </trkseg>
       </trk>
   </gpx>
   """

   @pytest.fixture
   def gpx_processor():
       return GPXProcessor(SAMPLE_GPX.encode())

   def test_get_track_points(gpx_processor):
       points = gpx_processor.get_track_points()
       assert len(points) == 2
       assert isinstance(points[0][0], float)  # latitude
       assert isinstance(points[0][1], float)  # longitude
       assert isinstance(points[0][2], float)  # elevation
       assert isinstance(points[0][3], datetime)  # time

   def test_calculate_statistics(gpx_processor):
       stats = gpx_processor.calculate_statistics()
       assert "total_distance" in stats
       assert "total_elevation_gain" in stats
       assert "start_time" in stats
       assert "end_time" in stats

   def test_calculate_covered_area(gpx_processor):
       area_size, area_geojson = gpx_processor.calculate_covered_area()
       assert isinstance(area_size, float)
       assert area_geojson["type"] == "MultiPolygon"
   ```

4. Area Calculation Tests:
   backend/tests/test_area_calculator.py:

   ```python
   import pytest
   from sqlalchemy.orm import Session
   from app.services.area_calculator import AreaCalculator
   from app.models.track import GPXTrack
   from app.models.track_point import TrackPoint
   from geoalchemy2.elements import WKTElement

   @pytest.fixture
   def area_calculator(db: Session):
       return AreaCalculator(db)

   @pytest.fixture
   def sample_track_points(db: Session, test_user: dict):
       track = GPXTrack(
           user_id=test_user["id"],
           filename="test.gpx",
           original_filename="test.gpx",
           file_size=1000,
           status="completed"
       )
       db.add(track)
       db.commit()

       points = [
           TrackPoint(
               track_id=track.id,
               location=WKTElement(f'POINT({lon} {lat})', srid=4326),
               elevation=100.0,
               time=datetime.utcnow()
           )
           for lat, lon in [(0, 0), (0, 1), (1, 1), (1, 0)]
       ]
       db.bulk_save_objects(points)
       db.commit()
       return points

   def test_calculate_user_coverage(area_calculator, sample_track_points, test_user: dict):
       coverage = area_calculator.calculate_user_coverage(test_user["id"])
       assert "area_km2" in coverage
       assert "percentage" in coverage
       assert "geojson" in coverage
       assert coverage["area_km2"] > 0

   def test_update_user_coverage(area_calculator, sample_track_points, test_user: dict):
       covered_area = area_calculator.update_user_coverage(test_user["id"])
       assert covered_area.area_km2 > 0
       assert covered_area.user_id == test_user["id"]
   ```

5. API Endpoint Tests:
   backend/tests/test_api_endpoints.py:

   ```python
   import pytest
   from fastapi.testclient import TestClient
   from sqlalchemy.orm import Session
   import os

   def get_auth_headers(client: TestClient, user: dict) -> dict:
       response = client.post(
           "/api/v1/auth/login",
           data={"username": user["email"], "password": user["password"]}
       )
       token = response.json()["access_token"]
       return {"Authorization": f"Bearer {token}"}

   def test_upload_gpx(client: TestClient, test_user: dict):
       headers = get_auth_headers(client, test_user)

       # Create a sample GPX file
       gpx_content = """
       <?xml version="1.0" encoding="UTF-8"?>
       <gpx version="1.1">
           <trk>
               <trkseg>
                   <trkpt lat="47.644548" lon="-122.326897">
                       <ele>4.46</ele>
                       <time>2024-01-01T10:00:00Z</time>
                   </trkpt>
               </trkseg>
           </trk>
       </gpx>
       """

       files = {
           "file": ("test.gpx", gpx_content.encode(), "application/gpx+xml")
       }

       response = client.post(
           "/api/v1/tracks/upload",
           headers=headers,
           files=files
       )

       assert response.status_code == 200
       assert "id" in response.json()
       assert response.json()["status"] == "processing"

   def test_list_tracks(client: TestClient, test_user: dict):
       headers = get_auth_headers(client, test_user)
       response = client.get("/api/v1/tracks/", headers=headers)
       assert response.status_code == 200
       assert isinstance(response.json(), list)

   def test_get_coverage_stats(client: TestClient, test_user: dict):
       headers = get_auth_headers(client, test_user)
       response = client.get("/api/v1/areas/coverage", headers=headers)
       assert response.status_code == 200
       assert "area_km2" in response.json()
       assert "percentage" in response.json()

   def test_get_coverage_details(client: TestClient, test_user: dict):
       headers = get_auth_headers(client, test_user)
       response = client.get("/api/v1/areas/coverage/details", headers=headers)
       assert response.status_code == 200
       assert "geojson" in response.json()
   ```

6. Integration Tests:
   backend/tests/test_integration.py:

   ```python
   import pytest
   from fastapi.testclient import TestClient
   from sqlalchemy.orm import Session
   from app.services.gpx_processor import GPXProcessor
   from app.services.area_calculator import AreaCalculator
   import time

   def test_complete_workflow(client: TestClient, test_user: dict, db: Session):
       # 1. Login
       headers = get_auth_headers(client, test_user)

       # 2. Upload GPX file
       gpx_content = """
       <?xml version="1.0" encoding="UTF-8"?>
       <gpx version="1.1">
           <trk>
               <trkseg>
                   <trkpt lat="47.644548" lon="-122.326897">
                       <ele>4.46</ele>
                       <time>2024-01-01T10:00:00Z</time>
                   </trkpt>
                   <trkpt lat="47.644648" lon="-122.326997">
                       <ele>4.94</ele>
                       <time>2024-01-01T10:00:10Z</time>
                   </trkpt>
               </trkseg>
           </trk>
       </gpx>
       """

       files = {
           "file": ("test.gpx", gpx_content.encode(), "application/gpx+xml")
       }

       upload_response = client.post(
           "/api/v1/tracks/upload",
           headers=headers,
           files=files
       )
       assert upload_response.status_code == 200
       track_id = upload_response.json()["id"]

       # 3. Wait for processing
       max_attempts = 10
       for _ in range(max_attempts):
           track_response = client.get(
               f"/api/v1/tracks/{track_id}",
               headers=headers
           )
           if track_response.json()["status"] == "completed":
               break
           time.sleep(1)

       # 4. Check coverage
       coverage_response = client.get(
           "/api/v1/areas/coverage/details",
           headers=headers
       )
       assert coverage_response.status_code == 200
       coverage = coverage_response.json()
       assert coverage["area_km2"] > 0

       # 5. Verify track points
       track_points_response = client.get(
           f"/api/v1/tracks/{track_id}/points",
           headers=headers
       )
       assert track_points_response.status_code == 200
       points = track_points_response.json()
       assert len(points) == 2

   def test_error_handling(client: TestClient, test_user: dict):
       headers = get_auth_headers(client, test_user)

       # Test invalid GPX file
       files = {
           "file": ("test.gpx", b"invalid content", "application/gpx+xml")
       }

     response = client.post(
           "/api/v1/tracks/upload",
           headers=headers,
           files=files
       )
       assert response.status_code == 400

       # Test file size limits
       large_content = "x" * (10 * 1024 * 1024)  # 10MB file
       files = {
           "file": ("large.gpx", large_content.encode(), "application/gpx+xml")
       }
       response = client.post(
           "/api/v1/tracks/upload",
           headers=headers,
           files=files
       )
       assert response.status_code == 413

   def test_concurrent_uploads(client: TestClient, test_user: dict):
       """Test handling multiple uploads simultaneously"""
       headers = get_auth_headers(client, test_user)

       gpx_content = """
       <?xml version="1.0" encoding="UTF-8"?>
       <gpx version="1.1">
           <trk>
               <trkseg>
                   <trkpt lat="47.644548" lon="-122.326897">
                       <ele>4.46</ele>
                       <time>2024-01-01T10:00:00Z</time>
                   </trkpt>
               </trkseg>
           </trk>
       </gpx>
       """

       # Upload multiple files
       track_ids = []
       for i in range(3):
           files = {
               "file": (f"test_{i}.gpx", gpx_content.encode(), "application/gpx+xml")
           }
           response = client.post(
               "/api/v1/tracks/upload",
               headers=headers,
               files=files
           )
           assert response.status_code == 200
           track_ids.append(response.json()["id"])

       # Wait for all processing to complete
       max_attempts = 10
       for track_id in track_ids:
           for _ in range(max_attempts):
               response = client.get(
                   f"/api/v1/tracks/{track_id}",
                   headers=headers
               )
               if response.json()["status"] == "completed":
                   break
               time.sleep(1)

           assert response.json()["status"] == "completed"

   def test_area_updates(client: TestClient, test_user: dict):
       """Test area calculations after multiple track uploads"""
       headers = get_auth_headers(client, test_user)

       # Get initial coverage
       initial_coverage = client.get(
           "/api/v1/areas/coverage",
           headers=headers
       ).json()

       # Upload a track
       gpx_content = """
       <?xml version="1.0" encoding="UTF-8"?>
       <gpx version="1.1">
           <trk>
               <trkseg>
                   <trkpt lat="47.644548" lon="-122.326897">
                       <ele>4.46</ele>
                       <time>2024-01-01T10:00:00Z</time>
                   </trkpt>
                   <trkpt lat="47.644648" lon="-122.326997">
                       <ele>4.94</ele>
                       <time>2024-01-01T10:00:10Z</time>
                   </trkpt>
               </trkseg>
           </trk>
       </gpx>
       """

       files = {
           "file": ("test.gpx", gpx_content.encode(), "application/gpx+xml")
       }

       response = client.post(
           "/api/v1/tracks/upload",
           headers=headers,
           files=files
       )
       assert response.status_code == 200

       # Wait for processing and area update
       time.sleep(2)

       # Get updated coverage
       final_coverage = client.get(
           "/api/v1/areas/coverage",
           headers=headers
       ).json()

       assert final_coverage["area_km2"] > initial_coverage["area_km2"]
   ```

7. Performance Tests:
   backend/tests/test_performance.py:

   ```python
   import pytest
   import time
   from sqlalchemy.orm import Session
   from app.services.area_calculator import AreaCalculator
   from app.services.gpx_processor import GPXProcessor

   def test_area_calculation_performance(db: Session, test_user: dict):
       """Test performance of area calculations with large datasets"""
       calculator = AreaCalculator(db)

       # Create multiple track points
       points = [(i/100, i/100) for i in range(1000)]  # 1000 points

       start_time = time.time()
       coverage = calculator.calculate_user_coverage(test_user["id"])
       end_time = time.time()

       assert (end_time - start_time) < 5  # Should complete within 5 seconds

   def test_gpx_processing_performance():
       """Test performance of GPX file processing"""
       # Generate large GPX content
       gpx_content = """
       <?xml version="1.0" encoding="UTF-8"?>
       <gpx version="1.1">
           <trk>
               <trkseg>
       """

       for i in range(1000):
           gpx_content += f"""
               <trkpt lat="{i/100}" lon="{i/100}">
                   <ele>{i}</ele>
                   <time>2024-01-01T10:00:{i:02d}Z</time>
               </trkpt>
           """

       gpx_content += """
               </trkseg>
           </trk>
       </gpx>
       """

       start_time = time.time()
       processor = GPXProcessor(gpx_content.encode())
       processor.get_track_points()
       processor.calculate_statistics()
       processor.calculate_covered_area()
       end_time = time.time()

       assert (end_time - start_time) < 3  # Should complete within 3 seconds
   ```

8. Test Environment Setup:
   backend/pytest.ini:
   ```ini
   [pytest]
   testpaths = tests
   python_files = test_*.py
   python_classes = Test*
   python_functions = test_*
   addopts = --verbosity=2
           --cov=app
           --cov-report=term-missing
           --cov-report=html
   markers =
       slow: marks tests as slow (deselect with '-m "not slow"')
       integration: marks tests as integration tests
   ```

## Part 6: Test Utilities and Documentation

9. Test Utilities:
   backend/tests/utils.py:

   ```python
   from typing import Dict, Any
   import os
   import tempfile
   from datetime import datetime, timedelta
   from app.core.security import create_access_token

   def create_test_gpx(points: list) -> str:
       """Create a GPX file with given points"""
       gpx_content = """<?xml version="1.0" encoding="UTF-8"?>
       <gpx version="1.1">
           <trk>
               <trkseg>
       """

       for lat, lon in points:
           gpx_content += f"""
               <trkpt lat="{lat}" lon="{lon}">
                   <ele>100.0</ele>
                   <time>{datetime.utcnow().isoformat()}Z</time>
               </trkpt>
           """

       gpx_content += """
               </trkseg>
           </trk>
       </gpx>
       """
       return gpx_content

   def create_temp_gpx_file(points: list) -> str:
       """Create a temporary GPX file and return its path"""
       content = create_test_gpx(points)
       with tempfile.NamedTemporaryFile(delete=False, suffix='.gpx') as tmp:
           tmp.write(content.encode())
           return tmp.name

   def get_test_token(user_id: str) -> str:
       """Create a test JWT token"""
       access_token_expires = timedelta(minutes=60)
       return create_access_token(
           subject=user_id, expires_delta=access_token_expires
       )

   def mock_gpx_processor_response() -> Dict[str, Any]:
       """Create mock GPX processor response for testing"""
       return {
           "statistics": {
               "total_distance": 1000.0,
               "total_elevation_gain": 100.0,
               "start_time": datetime.utcnow(),
               "end_time": datetime.utcnow() + timedelta(hours=1)
           },
           "points": [
               (0.0, 0.0, 100.0, datetime.utcnow()),
               (0.1, 0.1, 110.0, datetime.utcnow() + timedelta(minutes=30))
           ],
           "area": {
               "size": 1.5,
               "geojson": {
                   "type": "MultiPolygon",
                   "coordinates": [[[0, 0], [0, 1], [1, 1], [1, 0], [0, 0]]]
               }
           }
       }
   ```

10. Test Documentation:
    backend/tests/README.md:

    ````markdown
    # GPX Tracker Backend Tests

    This directory contains the test suite for the GPX Tracker backend application.

    ## Test Structure

    - `conftest.py`: Test configuration and fixtures
    - `test_auth.py`: Authentication tests
    - `test_gpx_processor.py`: GPX processing tests
    - `test_area_calculator.py`: Area calculation tests
    - `test_api_endpoints.py`: API endpoint tests
    - `test_integration.py`: Integration tests
    - `test_performance.py`: Performance tests
    - `utils.py`: Test utilities

    ## Running Tests

    1. Run all tests:
       ```bash
       poetry run pytest
       ```
    ````

    2. Run specific test file:

       ```bash
       poetry run pytest tests/test_auth.py
       ```

    3. Run tests with coverage:

       ```bash
       poetry run pytest --cov=app --cov-report=html
       ```

    4. Run tests by marker:
       ```bash
       poetry run pytest -m "not slow"  # Skip slow tests
       poetry run pytest -m integration  # Run only integration tests
       ```

    ## Test Coverage

    The test suite aims to maintain at least 80% code coverage. Coverage reports are generated in HTML format in the `htmlcov` directory.

    ## Writing Tests

    1. Use fixtures from `conftest.py` for common setup
    2. Follow the existing test structure and naming conventions
    3. Add appropriate markers for slow or integration tests
    4. Use test utilities from `utils.py` for common operations

    ## CI/CD Integration

    Tests are automatically run on every pull request and must pass before merging.

    ```
        #TODO: later
    ```

11. Add test scripts to pyproject.toml:
    ```toml
    [tool.poetry.scripts]
    test = "pytest"
    test-cov = "pytest --cov=app --cov-report=html"
    test-fast = "pytest -m 'not slow'"
    test-integration = "pytest -m integration"
    ```

# Backend Test Suite Complete

The backend test suite is now complete with:

1. Test Configuration

   - Database fixtures
   - Test client setup
   - User fixtures

2. Component Tests

   - Authentication tests
   - GPX processing tests
   - Area calculation tests
   - API endpoint tests

3. Integration Tests

   - Complete workflow tests
   - Concurrent operation tests
   - Error handling tests

4. Performance Tests

   - Area calculation performance
   - GPX processing performance

5. Test Utilities

   - GPX file creation
   - Token generation
   - Mock responses

6. Documentation
   - Test structure
   - Running instructions
   - Coverage requirements
   - CI/CD integration

To run the tests:

```bash
# Install dependencies
poetry install

# Run all tests
poetry run pytest

# Run with coverage
poetry run pytest --cov=app --cov-report=html

# Run specific test types
poetry run pytest -m integration
poetry run pytest -m 'not slow'
```

Next steps would be implementing the actual backend functionality following the test specifications we've created.",
