# Testing Strategy

## Part 1: Backend Unit Tests

1. Backend Test Configuration:
   backend/pyproject.toml:

   ```toml
   [tool.poetry.dependencies]
   pytest = "^7.4.0"
   pytest-asyncio = "^0.21.0"
   pytest-cov = "^4.1.0"
   httpx = "^0.24.1"
   faker = "^19.3.0"
   factory-boy = "^3.3.0"

   [tool.pytest.ini_options]
   testpaths = ["tests"]
   python_files = "test_*.py"
   asyncio_mode = "auto"
   addopts = [
       "--verbosity=2",
       "--cov=app",
       "--cov-report=term-missing",
       "--cov-report=html",
       "--no-cov-on-fail",
   ]
   ```

2. Test Fixtures:
   backend/tests/conftest.py:

   ```python
   import pytest
   from typing import Generator, AsyncGenerator
   from fastapi.testclient import TestClient
   from sqlalchemy.orm import Session
   from sqlalchemy import create_engine
   from sqlalchemy.pool import StaticPool
   from app.main import app
   from app.db.base import Base
   from app.db.session import get_db
   from app.core.config import settings
   from app.models.user import User
   from app.core.security import get_password_hash
   import asyncio
   from httpx import AsyncClient

   # Database fixtures
   SQLALCHEMY_DATABASE_URL = "sqlite:///./test.db"

   @pytest.fixture(scope="session")
   def engine():
       engine = create_engine(
           SQLALCHEMY_DATABASE_URL,
           connect_args={"check_same_thread": False},
           poolclass=StaticPool,
       )
       Base.metadata.create_all(bind=engine)
       yield engine
       Base.metadata.drop_all(bind=engine)

   @pytest.fixture(scope="function")
   def db(engine) -> Generator:
       connection = engine.connect()
       transaction = connection.begin()
       db = Session(bind=connection)
       yield db
       db.close()
       transaction.rollback()
       connection.close()

   @pytest.fixture(scope="function")
   def client(db) -> Generator:
       def override_get_db():
           try:
               yield db
           finally:
               pass

       app.dependency_overrides[get_db] = override_get_db
       with TestClient(app) as c:
           yield c

   @pytest.fixture(scope="function")
   async def async_client() -> AsyncGenerator:
       async with AsyncClient(app=app, base_url="http://test") as ac:
           yield ac

   # Test user fixtures
   @pytest.fixture(scope="function")
   def test_user(db: Session) -> User:
       user = User(
           email="test@example.com",
           hashed_password=get_password_hash("password123"),
           full_name="Test User"
       )
       db.add(user)
       db.commit()
       db.refresh(user)
       return user

   @pytest.fixture(scope="function")
   def test_superuser(db: Session) -> User:
       user = User(
           email="admin@example.com",
           hashed_password=get_password_hash("admin123"),
           full_name="Admin User",
           is_superuser=True
       )
       db.add(user)
       db.commit()
       db.refresh(user)
       return user

   # Auth token fixture
   @pytest.fixture(scope="function")
   def auth_headers(test_user: User) -> dict:
       from app.core.security import create_access_token
       access_token = create_access_token(subject=str(test_user.id))
       return {"Authorization": f"Bearer {access_token}"}
   ```

3. Model Tests:
   backend/tests/test_models.py:

   ```python
   import pytest
   from app.models import User, Track, TrackPoint
   from datetime import datetime
   from sqlalchemy.orm import Session
   from uuid import uuid4

   def test_user_model(db: Session, test_user: User):
       """Test User model relationships and methods"""
       assert test_user.email == "test@example.com"
       assert test_user.full_name == "Test User"
       assert not test_user.is_superuser

       # Test track relationship
       track = Track(
           user_id=test_user.id,
           filename="test.gpx",
           original_filename="test.gpx",
           file_size=1000,
           status="processing"
       )
       db.add(track)
       db.commit()

       assert len(test_user.tracks) == 1
       assert test_user.tracks[0].filename == "test.gpx"

   def test_track_model(db: Session, test_user: User):
       """Test Track model relationships and calculations"""
       track = Track(
           user_id=test_user.id,
           filename="test.gpx",
           original_filename="test.gpx",
           file_size=1000,
           status="completed",
           start_time=datetime.utcnow(),
           end_time=datetime.utcnow(),
           total_distance=1000.0,
           total_elevation_gain=100.0
       )
       db.add(track)
       db.commit()

       # Add track points
       points = [
           TrackPoint(
               track_id=track.id,
               latitude=0.0,
               longitude=0.0,
               elevation=100.0,
               time=datetime.utcnow()
           ),
           TrackPoint(
               track_id=track.id,
               latitude=0.1,
               longitude=0.1,
               elevation=150.0,
               time=datetime.utcnow()
           )
       ]
       db.bulk_save_objects(points)
       db.commit()

       assert track.points_count == 2
       assert track.average_elevation == 125.0
       assert track.elevation_gain == 50.0

   @pytest.mark.asyncio
   async def test_track_status_transitions(db: Session):
       """Test Track status transitions and validations"""
       track = Track(
           user_id=uuid4(),
           filename="test.gpx",
           original_filename="test.gpx",
           file_size=1000,
           status="processing"
       )
       db.add(track)
       db.commit()

       # Test valid transition
       track.status = "completed"
       db.commit()
       assert track.status == "completed"

       # Test invalid transition
       with pytest.raises(ValueError):
           track.status = "invalid_status"
   ```

## Part 2: Backend API and Service Tests

1. API Endpoint Tests:
   backend/tests/api/test_tracks.py:

   ```python
   import pytest
   from fastapi.testclient import TestClient
   from sqlalchemy.orm import Session
   from app.models import Track, User
   import os

   def test_upload_track(client: TestClient, test_user: User, auth_headers: dict):
       """Test track upload endpoint"""
       # Create test GPX file
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

       with open("test.gpx", "w") as f:
           f.write(gpx_content)

       try:
           with open("test.gpx", "rb") as f:
               response = client.post(
                   "/api/v1/tracks/upload",
                   files={"file": ("test.gpx", f, "application/gpx+xml")},
                   headers=auth_headers
               )

           assert response.status_code == 200
           data = response.json()
           assert data["status"] == "processing"
           assert "id" in data

           # Verify database record
           track = client.get(
               f"/api/v1/tracks/{data['id']}",
               headers=auth_headers
           )
           assert track.status_code == 200
           assert track.json()["filename"] == "test.gpx"

       finally:
           os.remove("test.gpx")

   def test_list_tracks(client: TestClient, test_user: User, auth_headers: dict, db: Session):
       """Test track listing endpoint"""
       # Create test tracks
       tracks = [
           Track(
               user_id=test_user.id,
               filename=f"test_{i}.gpx",
               original_filename=f"test_{i}.gpx",
               file_size=1000,
               status="completed"
           ) for i in range(3)
       ]
       db.bulk_save_objects(tracks)
       db.commit()

       response = client.get("/api/v1/tracks", headers=auth_headers)
       assert response.status_code == 200
       data = response.json()
       assert len(data) == 3

   @pytest.mark.asyncio
   async def test_track_processing(async_client, test_user: User, auth_headers: dict):
       """Test track processing workflow"""
       # Upload track
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

       with open("test.gpx", "w") as f:
           f.write(gpx_content)

       try:
           files = {"file": ("test.gpx", open("test.gpx", "rb"), "application/gpx+xml")}
           response = await async_client.post(
               "/api/v1/tracks/upload",
               files=files,
               headers=auth_headers
           )
           assert response.status_code == 200
           track_id = response.json()["id"]

           # Wait for processing
           max_attempts = 10
           for _ in range(max_attempts):
               response = await async_client.get(
                   f"/api/v1/tracks/{track_id}",
                   headers=auth_headers
               )
               if response.json()["status"] == "completed":
                   break
               await asyncio.sleep(1)

           assert response.json()["status"] == "completed"
           assert response.json()["total_distance"] > 0

       finally:
           os.remove("test.gpx")

   def test_track_validation(client: TestClient, auth_headers: dict):
       """Test track upload validation"""
       # Test invalid file type
       response = client.post(
           "/api/v1/tracks/upload",
           files={"file": ("test.txt", b"invalid content", "text/plain")},
           headers=auth_headers
       )
       assert response.status_code == 400

       # Test missing file
       response = client.post(
           "/api/v1/tracks/upload",
           headers=auth_headers
       )
       assert response.status_code == 422
   ```

2. Service Layer Tests:
   backend/tests/services/test_gpx_processor.py:

   ```python
   import pytest
   from app.services.gpx_processor import GPXProcessor
   from datetime import datetime

   @pytest.fixture
   def sample_gpx():
       return """
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

   def test_gpx_parsing(sample_gpx):
       """Test GPX file parsing"""
       processor = GPXProcessor(sample_gpx.encode())
       points = processor.get_track_points()

       assert len(points) == 2
       assert points[0]["latitude"] == 47.644548
       assert points[0]["longitude"] == -122.326897
       assert points[0]["elevation"] == 4.46

   def test_statistics_calculation(sample_gpx):
       """Test track statistics calculation"""
       processor = GPXProcessor(sample_gpx.encode())
       stats = processor.calculate_statistics()

       assert stats["total_distance"] > 0
       assert stats["total_elevation_gain"] == 0.48  # 4.94 - 4.46
       assert isinstance(stats["start_time"], datetime)
       assert isinstance(stats["end_time"], datetime)
       assert stats["moving_time"] == 10  # 10 seconds between points

   def test_area_calculation(sample_gpx):
       """Test covered area calculation"""
       processor = GPXProcessor(sample_gpx.encode())
       area_size, area_geojson = processor.calculate_covered_area()

       assert area_size > 0
       assert area_geojson["type"] == "MultiPolygon"
       assert len(area_geojson["coordinates"]) > 0

   def test_invalid_gpx():
       """Test handling of invalid GPX data"""
       with pytest.raises(ValueError):
           GPXProcessor(b"invalid data")
   ```

backend/tests/services/test_area_calculator.py:

```python

   import pytest
   from app.services.area_calculator import AreaCalculator
   from app.models import Track, TrackPoint
   from datetime import datetime
   from sqlalchemy.orm import Session

   @pytest.fixture
   def sample_track(db: Session, test_user):
       track = Track(
           user_id=test_user.id,
           filename="test.gpx",
           status="completed"
       )
       db.add(track)
       db.commit()

       points = [
           TrackPoint(
               track_id=track.id,
               latitude=0.0,
               longitude=0.0,
               elevation=100,
               time=datetime.utcnow()
           ),
           TrackPoint(
               track_id=track.id,
               latitude=0.001,
               longitude=0.001,
               elevation=110,
               time=datetime.utcnow()
           )
       ]
       db.bulk_save_objects(points)
       db.commit()
       return track

   def test_calculate_user_coverage(db: Session, test_user, sample_track):
       """Test user coverage calculation"""
       calculator = AreaCalculator(db)
       coverage = calculator.calculate_user_coverage(str(test_user.id))

       assert coverage["area_km2"] > 0
       assert coverage["percentage"] > 0
       assert "geojson" in coverage

   def test_merge_overlapping_areas(db: Session, test_user):
       """Test merging of overlapping areas"""
       calculator = AreaCalculator(db)

       # Create two overlapping tracks
       tracks = []
       for i in range(2):
           track = Track(
               user_id=test_user.id,
               filename=f"test_{i}.gpx",
               status="completed"
           )
           db.add(track)
           db.commit()

           points = [
               TrackPoint(
                   track_id=track.id,
                   latitude=0.0 + (i * 0.0001),
                   longitude=0.0 + (i * 0.0001),
                   elevation=100,
                   time=datetime.utcnow()
               )
           ]
           db.bulk_save_objects(points)
           db.commit()
           tracks.append(track)

       # Calculate merged area
       merged = calculator.merge_areas([str(t.id) for t in tracks])
       assert merged["area_km2"] > 0
       assert merged["area_km2"] < sum(t.area_km2 for t in tracks)  # Should be less due to overlap
```

3. Integration Tests:
   backend/tests/integration/test_track_workflow.py:

   ```python
   import pytest
   from httpx import AsyncClient
   import asyncio
   from app.models import Track
   from sqlalchemy.orm import Session

   @pytest.mark.asyncio
   async def test_complete_track_workflow(async_client: AsyncClient, db: Session, test_user, auth_headers):
       """Test complete track upload and processing workflow"""
       # 1. Upload GPX file
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

       with open("test.gpx", "w") as f:
           f.write(gpx_content)

       files = {"file": ("test.gpx", open("test.gpx", "rb"), "application/gpx+xml")}
       response = await async_client.post(
           "/api/v1/tracks/upload",
           files=files,
           headers=auth_headers
       )
       assert response.status_code == 200
       track_id = response.json()["id"]

       # 2. Wait for processing completion
       max_attempts = 10
       track_data = None
       for _ in range(max_attempts):
           response = await async_client.get(
               f"/api/v1/tracks/{track_id}",
               headers=auth_headers
           )
           track_data = response.json()
           if track_data["status"] == "completed":
               break
           await asyncio.sleep(1)

       assert track_data["status"] == "completed"

       # 3. Verify track points
       response = await async_client.get(
           f"/api/v1/tracks/{track_id}/points",
           headers=auth_headers
       )
       assert response.status_code == 200
       points = response.json()
       assert len(points) == 2

       # 4. Check coverage calculation
       response = await async_client.get(
           "/api/v1/areas/coverage",
           headers=auth_headers
       )
       assert response.status_code == 200
       coverage = response.json()
       assert coverage["area_km2"] > 0

       # 5. Clean up
       os.remove("test.gpx")
   ```

## Part 3. Frontend Test Configuration

frontend/package.json:

```json
{
  "scripts": {
    "test": "vitest",
    "test:coverage": "vitest run --coverage",
    "test:ui": "vitest --ui",
    "test:e2e": "cypress run",
    "test:e2e:dev": "cypress open"
  },
  "devDependencies": {
    "@testing-library/react": "^14.0.0",
    "@testing-library/jest-dom": "^6.1.0",
    "@testing-library/user-event": "^14.0.0",
    "@vitest/coverage-v8": "^0.34.0",
    "@vitest/ui": "^0.34.0",
    "vitest": "^0.34.0",
    "jsdom": "^22.1.0",
    "cypress": "^13.0.0",
    "@types/testing-library__jest-dom": "^5.14.9"
  }
}
```

2. Vitest Configuration:
   frontend/vitest.config.ts:

   ```typescript
   import { defineConfig } from "vitest/config";
   import react from "@vitejs/plugin-react";
   import tsconfigPaths from "vite-tsconfig-paths";

   export default defineConfig({
     plugins: [react(), tsconfigPaths()],
     test: {
       environment: "jsdom",
       globals: true,
       setupFiles: ["./src/test/setup.ts"],
       coverage: {
         provider: "v8",
         reporter: ["text", "html"],
         exclude: ["node_modules/", "src/test/", "**/*.d.ts", "**/*.test.*"],
       },
     },
   });
   ```

3. Test Setup:
   frontend/src/test/setup.ts:

   ```typescript
   import "@testing-library/jest-dom";
   import { expect, afterEach } from "vitest";
   import { cleanup } from "@testing-library/react";
   import * as matchers from "@testing-library/jest-dom/matchers";

   expect.extend(matchers);

   // Clean up after each test
   afterEach(() => {
     cleanup();
   });
   ```

4. Test Utils:
   frontend/src/test/utils.tsx:

   ```typescript
   import React from "react";
   import { render } from "@testing-library/react";
   import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
   import { MemoryRouter } from "react-router-dom";
   import { vi } from "vitest";

   // Create a custom render function that includes providers
   export function renderWithProviders(
     ui: React.ReactElement,
     {
       route = "/",
       queryClient = new QueryClient({
         defaultOptions: {
           queries: {
             retry: false,
           },
         },
       }),
     } = {}
   ) {
     return {
       queryClient,
       ...render(ui, {
         wrapper: ({ children }) => (
           <QueryClientProvider client={queryClient}>
             <MemoryRouter initialEntries={[route]}>{children}</MemoryRouter>
           </QueryClientProvider>
         ),
       }),
     };
   }

   // Mock Leaflet map container
   vi.mock("react-leaflet", () => ({
     MapContainer: ({ children }: { children: React.ReactNode }) => (
       <div data-testid="map-container">{children}</div>
     ),
     TileLayer: () => null,
     Marker: () => null,
     Popup: () => null,
   }));

   // Mock IntersectionObserver
   const mockIntersectionObserver = vi.fn();
   mockIntersectionObserver.mockReturnValue({
     observe: () => null,
     unobserve: () => null,
     disconnect: () => null,
   });
   window.IntersectionObserver = mockIntersectionObserver;
   ```

5. Component Tests:
   frontend/src/components/TrackList.test.tsx:

   ```typescript
   import { describe, it, expect } from "vitest";
   import { screen, waitFor } from "@testing-library/react";
   import userEvent from "@testing-library/user-event";
   import { renderWithProviders } from "../test/utils";
   import TrackList from "./TrackList";
   import { Track } from "../types";

   const mockTracks: Track[] = [
     {
       id: "1",
       filename: "test1.gpx",
       status: "completed",
       totalDistance: 1000,
       totalElevationGain: 100,
       startTime: "2024-01-01T10:00:00Z",
       endTime: "2024-01-01T11:00:00Z",
       createdAt: "2024-01-01T10:00:00Z",
     },
     {
       id: "2",
       filename: "test2.gpx",
       status: "processing",
       totalDistance: 0,
       totalElevationGain: 0,
       startTime: null,
       endTime: null,
       createdAt: "2024-01-01T12:00:00Z",
     },
   ];

   describe("TrackList", () => {
     it("renders tracks correctly", () => {
       renderWithProviders(<TrackList tracks={mockTracks} />);

       expect(screen.getByText("test1.gpx")).toBeInTheDocument();
       expect(screen.getByText("test2.gpx")).toBeInTheDocument();
       expect(screen.getByText("1 km")).toBeInTheDocument();
       expect(screen.getByText("Processing...")).toBeInTheDocument();
     });

     it("allows sorting tracks", async () => {
       const user = userEvent.setup();
       renderWithProviders(<TrackList tracks={mockTracks} />);

       await user.click(screen.getByText("Date"));

       const tracks = screen.getAllByTestId("track-item");
       expect(tracks[0]).toHaveTextContent("test2.gpx");
       expect(tracks[1]).toHaveTextContent("test1.gpx");
     });

     it("filters tracks by status", async () => {
       const user = userEvent.setup();
       renderWithProviders(<TrackList tracks={mockTracks} />);

       await user.selectOptions(screen.getByLabelText("Status"), "completed");

       await waitFor(() => {
         expect(screen.queryByText("test2.gpx")).not.toBeInTheDocument();
         expect(screen.getByText("test1.gpx")).toBeInTheDocument();
       });
     });
   });
   ```

## Part 4: Advanced Frontend Testing

1. Map Component Tests:
   frontend/src/components/Map.test.tsx:

   ```typescript
   import { describe, it, expect, vi } from "vitest";
   import { screen, waitFor } from "@testing-library/react";
   import userEvent from "@testing-library/user-event";
   import { renderWithProviders } from "../test/utils";
   import Map from "./Map";
   import { useMapState } from "../stores/mapStore";

   // Mock GeoJSON data
   const mockGeoJSON = {
     type: "FeatureCollection",
     features: [
       {
         type: "Feature",
         geometry: {
           type: "MultiPolygon",
           coordinates: [
             [
               [
                 [0, 0],
                 [0, 1],
                 [1, 1],
                 [1, 0],
                 [0, 0],
               ],
             ],
           ],
         },
         properties: {
           area_km2: 100,
           date: "2024-01-01",
         },
       },
     ],
   };

   // Mock API calls
   vi.mock("../services/api", () => ({
     getCoverageGeoJSON: vi.fn().mockResolvedValue(mockGeoJSON),
     getTrackPoints: vi.fn().mockResolvedValue([
       { latitude: 0, longitude: 0, elevation: 100 },
       { latitude: 1, longitude: 1, elevation: 200 },
     ]),
   }));

   describe("Map Component", () => {
     it("renders map with controls", () => {
       renderWithProviders(<Map />);

       expect(screen.getByTestId("map-container")).toBeInTheDocument();
       expect(
         screen.getByRole("button", { name: /zoom in/i })
       ).toBeInTheDocument();
       expect(
         screen.getByRole("button", { name: /zoom out/i })
       ).toBeInTheDocument();
     });

     it("toggles layer visibility", async () => {
       const user = userEvent.setup();
       renderWithProviders(<Map />);

       const coverageToggle = screen.getByLabelText(/coverage layer/i);
       await user.click(coverageToggle);

       expect(useMapState.getState().showCoverage).toBeFalsy();
     });

     it("displays area information on click", async () => {
       const user = userEvent.setup();
       renderWithProviders(<Map />);

       // Simulate clicking on an area
       const mapArea = screen.getByTestId("coverage-layer");
       await user.click(mapArea);

       await waitFor(() => {
         expect(screen.getByText("100 km²")).toBeInTheDocument();
         expect(screen.getByText("January 1, 2024")).toBeInTheDocument();
       });
     });
   });
   ```

2. Upload Component Tests:
   frontend/src/components/Upload.test.tsx:

   ```typescript
   import { describe, it, expect, vi } from "vitest";
   import { screen, waitFor } from "@testing-library/react";
   import userEvent from "@testing-library/user-event";
   import { renderWithProviders } from "../test/utils";
   import Upload from "./Upload";

   describe("Upload Component", () => {
     it("handles file upload", async () => {
       const user = userEvent.setup();
       const file = new File(["test gpx content"], "track.gpx", {
         type: "application/gpx+xml",
       });
       const uploadMock = vi
         .fn()
         .mockResolvedValue({ id: "123", status: "processing" });

       vi.mock("../services/api", () => ({
         uploadTrack: uploadMock,
       }));

       renderWithProviders(<Upload />);

       const input = screen.getByLabelText(/choose file/i);
       await user.upload(input, file);

       await waitFor(() => {
         expect(uploadMock).toHaveBeenCalled();
         expect(screen.getByText(/processing/i)).toBeInTheDocument();
       });
     });

     it("validates file type", async () => {
       const user = userEvent.setup();
       const file = new File(["invalid"], "test.txt", { type: "text/plain" });

       renderWithProviders(<Upload />);

       const input = screen.getByLabelText(/choose file/i);
       await user.upload(input, file);

       expect(screen.getByText(/must be a gpx file/i)).toBeInTheDocument();
     });

     it("shows upload progress", async () => {
       const user = userEvent.setup();
       const file = new File(["test gpx content"], "track.gpx", {
         type: "application/gpx+xml",
       });

       // Mock XMLHttpRequest for progress events
       const xhrMock = {
         upload: {
           addEventListener: vi.fn((event, handler) => {
             if (event === "progress") {
               handler({ loaded: 50, total: 100 });
             }
           }),
         },
         open: vi.fn(),
         send: vi.fn(),
         setRequestHeader: vi.fn(),
       };

       window.XMLHttpRequest = vi.fn(() => xhrMock);

       renderWithProviders(<Upload />);

       const input = screen.getByLabelText(/choose file/i);
       await user.upload(input, file);

       expect(screen.getByText("50%")).toBeInTheDocument();
     });
   });
   ```

3. End-to-End Tests:
   frontend/cypress/e2e/track-workflow.cy.ts:

   ```typescript
   describe("Track Upload and Viewing Workflow", () => {
     beforeEach(() => {
       cy.login("test@example.com", "password123");
     });

     it("completes full track workflow", () => {
       // Upload track
       cy.visit("/upload");
       cy.fixture("sample.gpx").then((fileContent) => {
         cy.get("input[type=file]").attachFile({
           fileContent,
           fileName: "sample.gpx",
           mimeType: "application/gpx+xml",
         });
       });

       // Wait for processing
       cy.get("[data-testid=progress-indicator]", { timeout: 10000 }).should(
         "not.exist"
       );

       // View track details
       cy.get("[data-testid=track-list]").contains("sample.gpx").click();

       // Verify track information
       cy.get("[data-testid=track-distance]").should("not.be.empty");
       cy.get("[data-testid=track-elevation]").should("not.be.empty");

       // Check map visualization
       cy.get("[data-testid=map-container]").should("be.visible");
       cy.get("[data-testid=track-path]").should("exist");

       // Verify coverage calculation
       cy.get("[data-testid=coverage-stats]").should("contain", "km²");
     });

     it("handles upload errors gracefully", () => {
       cy.visit("/upload");
       cy.fixture("invalid.txt").then((fileContent) => {
         cy.get("input[type=file]").attachFile({
           fileContent,
           fileName: "invalid.txt",
           mimeType: "text/plain",
         });
       });

       cy.get("[data-testid=error-message]")
         .should("be.visible")
         .and("contain", "Invalid file type");
     });
   });
   ```

## Part 5: Advanced E2E Testing and API Mocking

1. Cypress Custom Commands:
   frontend/cypress/support/commands.ts:

   ```typescript
   import "@testing-library/cypress/add-commands";

   declare global {
     namespace Cypress {
       interface Chainable {
         login(email: string, password: string): void;
         uploadGpxFile(filePath: string): void;
         interceptApi(route: string, fixture: string): void;
       }
     }
   }

   Cypress.Commands.add("login", (email: string, password: string) => {
     cy.intercept("POST", "/api/v1/auth/login").as("loginRequest");

     cy.visit("/login");
     cy.findByLabelText(/email/i).type(email);
     cy.findByLabelText(/password/i).type(password);
     cy.findByRole("button", { name: /sign in/i }).click();

     cy.wait("@loginRequest");
     cy.url().should("include", "/dashboard");
   });

   Cypress.Commands.add("uploadGpxFile", (filePath: string) => {
     cy.intercept("POST", "/api/v1/tracks/upload").as("uploadRequest");
     cy.intercept("GET", "/api/v1/tracks/*").as("trackRequest");

     cy.visit("/upload");
     cy.get("input[type=file]").attachFile(filePath);

     cy.wait("@uploadRequest");
     cy.wait("@trackRequest");
   });

   Cypress.Commands.add("interceptApi", (route: string, fixture: string) => {
     cy.intercept("GET", `/api/v1/${route}`, { fixture }).as(route);
   });
   ```

2. Advanced E2E Tests:
   frontend/cypress/e2e/map-interactions.cy.ts:

   ```typescript
   describe("Map Interactions", () => {
     beforeEach(() => {
       cy.login("test@example.com", "password123");
       cy.interceptApi("areas/coverage", "coverage.json");
       cy.interceptApi("tracks", "tracks.json");
     });

     it("allows map navigation and interaction", () => {
       cy.visit("/map");

       // Test zoom controls
       cy.get("[data-testid=zoom-in]").click();
       cy.get("[data-testid=map-container]")
         .should("have.attr", "data-zoom")
         .and("not.equal", "2");

       // Test layer toggling
       cy.get("[data-testid=layer-control]").click();
       cy.get("[data-testid=coverage-toggle]").click();
       cy.get("[data-testid=coverage-layer]").should("not.exist");

       // Test area clicking
       cy.get("[data-testid=map-container]").click(100, 100);
       cy.get("[data-testid=area-popup]")
         .should("be.visible")
         .and("contain", "km²");

       // Test track display
       cy.get("[data-testid=track-toggle]").click();
       cy.get("[data-testid=track-layer]").should("be.visible");
     });

     it("syncs map state with URL", () => {
       cy.visit("/map?lat=0&lng=0&zoom=5");

       cy.get("[data-testid=map-container]")
         .should("have.attr", "data-center", "0,0")
         .and("have.attr", "data-zoom", "5");

       // Move map
       cy.get("[data-testid=map-container]")
         .trigger("mousedown", 100, 100)
         .trigger("mousemove", 200, 200)
         .trigger("mouseup");

       cy.url().should("include", "lat=").and("include", "lng=");
     });
   });
   ```

3. API Mocking Setup:
   frontend/src/test/mocks/handlers.ts:

   ```typescript
   import { rest } from "msw";
   import { API_URL } from "../../config";

   export const handlers = [
     // Auth endpoints
     rest.post(`${API_URL}/auth/login`, (req, res, ctx) => {
       return res(
         ctx.status(200),
         ctx.json({
           access_token: "mock-token",
           token_type: "bearer",
         })
       );
     }),

     // Tracks endpoints
     rest.get(`${API_URL}/tracks`, (req, res, ctx) => {
       return res(
         ctx.status(200),
         ctx.json([
           {
             id: "1",
             filename: "test.gpx",
             status: "completed",
             totalDistance: 1000,
             totalElevationGain: 100,
             startTime: "2024-01-01T10:00:00Z",
             endTime: "2024-01-01T11:00:00Z",
           },
         ])
       );
     }),

     rest.post(`${API_URL}/tracks/upload`, async (req, res, ctx) => {
       const file = await req.json();
       return res(
         ctx.status(200),
         ctx.json({
           id: "2",
           filename: file.name,
           status: "processing",
         })
       );
     }),

     // Coverage endpoints
     rest.get(`${API_URL}/areas/coverage`, (req, res, ctx) => {
       return res(
         ctx.status(200),
         ctx.json({
           area_km2: 1000,
           percentage: 0.0002,
           geojson: {
             type: "FeatureCollection",
             features: [],
           },
         })
       );
     }),
   ];
   ```

4. Mock Service Worker Setup:
   frontend/src/test/setup-msw.ts:

   ```typescript
   import { setupServer } from "msw/node";
   import { handlers } from "./mocks/handlers";

   export const server = setupServer(...handlers);

   beforeAll(() => server.listen({ onUnhandledRequest: "error" }));
   afterEach(() => server.resetHandlers());
   afterAll(() => server.close());
   ```
