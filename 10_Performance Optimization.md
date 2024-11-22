# Performance Optimization

## Part 1: Database Optimizations

1. Database Query Optimizations:
   backend/app/db/optimizations.py:

   ```python
   from typing import List, Dict, Any
   from sqlalchemy import text, Index, func
   from sqlalchemy.orm import Session, joinedload, contains_eager
   from app.models import Track, TrackPoint, CoveredArea
   from geoalchemy2.types import Geography
   from datetime import datetime, timedelta

   class QueryOptimizer:
       def __init__(self, db: Session):
           self.db = db

       def get_user_tracks_optimized(self, user_id: str, limit: int = 10) -> List[Track]:
           """Optimized query for user tracks with related data"""
           return self.db.query(Track)\\
               .options(
                   joinedload(Track.points).load_only(
                       TrackPoint.latitude,
                       TrackPoint.longitude,
                       TrackPoint.elevation
                   )
               )\\
               .filter(Track.user_id == user_id)\\
               .order_by(Track.created_at.desc())\\
               .limit(limit)\\
               .all()

       def get_covered_areas_optimized(self, user_id: str) -> Dict[str, Any]:
           """Optimized spatial query for covered areas"""
           # Use spatial index for efficient geometry operations
           result = self.db.query(
               func.ST_Union(CoveredArea.geom).label('combined_geom'),
               func.sum(CoveredArea.area_km2).label('total_area')
           ).filter(
               CoveredArea.user_id == user_id
           ).first()

           return {
               'geom': result.combined_geom,
               'area_km2': float(result.total_area or 0)
           }

       def get_track_points_chunked(self, track_id: str, chunk_size: int = 1000) -> List[TrackPoint]:
           """Get track points in chunks for memory efficiency"""
           points = []
           offset = 0

           while True:
               chunk = self.db.query(TrackPoint)\\
                   .filter(TrackPoint.track_id == track_id)\\
                   .order_by(TrackPoint.time)\\
                   .offset(offset)\\
                   .limit(chunk_size)\\
                   .all()

               if not chunk:
                   break

               points.extend(chunk)
               offset += chunk_size

           return points

       def create_track_points_bulk(self, points: List[Dict[str, Any]]) -> None:
           """Bulk insert track points efficiently"""
           if not points:
               return

           # Use COPY command for PostgreSQL bulk insert
           points_values = ','.join(
               self.db.bind.execute(
                   text(
                       """INSERT INTO track_points
                          (track_id, latitude, longitude, elevation, time)
                          VALUES (:track_id, :latitude, :longitude, :elevation, :time)"""
                   ),
                   points
               )
           )

   # Create optimized indexes
   def create_optimized_indexes(engine):
       # Spatial index for track points
       Index(
           'idx_track_points_location',
           TrackPoint.__table__.c.location,
           postgresql_using='gist'
       )

       # Composite index for common queries
       Index(
           'idx_tracks_user_status',
           Track.__table__.c.user_id,
           Track.__table__.c.status
       )

       # Partial index for completed tracks
       Index(
           'idx_tracks_completed',
           Track.__table__.c.status,
           postgresql_where=Track.__table__.c.status == 'completed'
       )

       # B-tree index for timestamp range queries
       Index(
           'idx_track_points_time',
           TrackPoint.__table__.c.time
       )
   ```

2. Query Analysis Tool:
   backend/app/utils/query_analyzer.py:

   ```python
   from typing import List, Dict, Any
   from sqlalchemy import text
   from sqlalchemy.orm import Session
   import logging

   class QueryAnalyzer:
       def __init__(self, db: Session):
           self.db = db

       def analyze_query(self, query) -> Dict[str, Any]:
           """Analyze query execution plan"""
           explain_query = f"EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) {query}"
           result = self.db.execute(text(explain_query)).scalar()

           return self._parse_explain_result(result[0])

       def _parse_explain_result(self, explain_data: Dict) -> Dict[str, Any]:
           """Parse EXPLAIN ANALYZE output"""
           plan = explain_data.get('Plan', {})
           return {
               'execution_time': plan.get('Actual Total Time'),
               'planning_time': explain_data.get('Planning Time'),
               'rows_processed': plan.get('Actual Rows'),
               'buffers_hit': plan.get('Shared Hit Blocks', 0),
               'buffers_read': plan.get('Shared Read Blocks', 0),
               'scan_type': plan.get('Node Type'),
               'index_used': plan.get('Index Name')
           }

       def get_slow_queries(self, threshold_ms: float = 1000) -> List[Dict[str, Any]]:
           """Get list of slow queries from pg_stat_statements"""
           slow_queries = self.db.execute(text("""
               SELECT query, calls,
                      total_exec_time/calls as avg_time,
                      rows/calls as avg_rows,
                      shared_blks_hit + shared_blks_read as io_blocks
               FROM pg_stat_statements
               WHERE total_exec_time/calls > :threshold
               ORDER BY avg_time DESC
               LIMIT 10
           """), {'threshold': threshold_ms}).fetchall()

           return [
               {
                   'query': row[0],
                   'calls': row[1],
                   'avg_time_ms': row[2],
                   'avg_rows': row[3],
                   'io_blocks': row[4]
               }
               for row in slow_queries
           ]

       def suggest_indexes(self) -> List[Dict[str, Any]]:
           """Suggest missing indexes based on query patterns"""
           missing_indexes = self.db.execute(text("""
               SELECT schemaname, tablename,
                      reltuples::bigint as rows,
                      seq_scan, idx_scan,
                      seq_tup_read + idx_tup_read as total_reads
               FROM pg_stat_user_tables
               WHERE seq_scan > idx_scan
               AND reltuples > 1000
               ORDER BY seq_scan DESC
           """)).fetchall()

           return [
               {
                   'schema': row[0],
                   'table': row[1],
                   'rows': row[2],
                   'sequential_scans': row[3],
                   'index_scans': row[4],
                   'total_reads': row[5],
                   'recommendation': 'Consider adding index'
               }
               for row in missing_indexes
           ]
   ```

## Part 2: Frontend Optimizations

1. Performance-Optimized Components:
   frontend/src/components/optimized/VirtualizedTrackList.tsx:

   ```typescript
   import React, { useCallback, useMemo } from "react";
   import { useVirtualizer } from "@tanstack/react-virtual";
   import { Track } from "@/types";
   import { useTrackStore } from "@/stores/trackStore";

   interface VirtualizedTrackListProps {
     tracks: Track[];
     rowHeight?: number;
     overscan?: number;
   }

   const VirtualizedTrackList: React.FC<VirtualizedTrackListProps> = ({
     tracks,
     rowHeight = 60,
     overscan = 5,
   }) => {
     const parentRef = React.useRef<HTMLDivElement>(null);

     const virtualizer = useVirtualizer({
       count: tracks.length,
       getScrollElement: () => parentRef.current,
       estimateSize: useCallback(() => rowHeight, [rowHeight]),
       overscan,
     });

     const items = useMemo(
       () =>
         virtualizer.getVirtualItems().map((virtualRow) => ({
           virtualRow,
           track: tracks[virtualRow.index],
         })),
       [virtualizer, tracks]
     );

     return (
       <div
         ref={parentRef}
         className="h-full overflow-auto"
         style={{
           height: `calc(100vh - 200px)`,
           width: "100%",
         }}
       >
         <div
           style={{
             height: `${virtualizer.getTotalSize()}px`,
             width: "100%",
             position: "relative",
           }}
         >
           {items.map(({ virtualRow, track }) => (
             <div
               key={track.id}
               data-index={virtualRow.index}
               ref={virtualizer.measureElement}
               className="absolute top-0 left-0 w-full"
               style={{
                 transform: `translateY(${virtualRow.start}px)`,
               }}
             >
               <TrackListItem track={track} />
             </div>
           ))}
         </div>
       </div>
     );
   };

   // Memoized list item component
   const TrackListItem = React.memo<{ track: Track }>(
     ({ track }) => (
       <div className="p-4 border-b hover:bg-gray-50">
         <h3 className="font-medium">{track.filename}</h3>
         <div className="text-sm text-gray-500">
           {formatDistance(track.totalDistance)} •{formatDate(track.createdAt)}
         </div>
       </div>
     ),
     (prev, next) => prev.track.id === next.track.id
   );

   export default React.memo(VirtualizedTrackList);
   ```

2. Optimized Map Rendering:
   frontend/src/components/optimized/OptimizedMap.tsx:

   ```typescript
   import React, { useCallback, useMemo, useRef, useEffect } from "react";
   import { MapContainer, TileLayer, useMap } from "react-leaflet";
   import { LatLngBounds, LatLng } from "leaflet";
   import { debounce } from "lodash";
   import { useMapState } from "@/stores/mapStore";

   // Map update component to prevent unnecessary rerenders
   const MapUpdater: React.FC<{
     center?: LatLng;
     zoom?: number;
     bounds?: LatLngBounds;
   }> = React.memo(({ center, zoom, bounds }) => {
     const map = useMap();

     useEffect(() => {
       if (bounds) {
         map.fitBounds(bounds);
       } else if (center && zoom) {
         map.setView(center, zoom);
       }
     }, [map, center, zoom, bounds]);

     return null;
   });

   // Optimized track path renderer
   const TrackPathLayer: React.FC<{
     points: [number, number][];
     color?: string;
   }> = React.memo(
     ({ points, color = "#3b82f6" }) => {
       const map = useMap();
       const canvasRef = useRef<HTMLCanvasElement>(null);

       useEffect(() => {
         if (!canvasRef.current) return;

         const ctx = canvasRef.current.getContext("2d");
         if (!ctx) return;

         // Clear canvas
         ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);

         // Draw path
         ctx.beginPath();
         ctx.strokeStyle = color;
         ctx.lineWidth = 2;

         points.forEach((point, index) => {
           const pixel = map.latLngToContainerPoint(point);
           if (index === 0) {
             ctx.moveTo(pixel.x, pixel.y);
           } else {
             ctx.lineTo(pixel.x, pixel.y);
           }
         });

         ctx.stroke();
       }, [map, points, color]);

       return (
         <canvas
           ref={canvasRef}
           className="absolute top-0 left-0 z-10"
           width={map.getSize().x}
           height={map.getSize().y}
         />
       );
     },
     (prev, next) => {
       return (
         prev.color === next.color &&
         prev.points.length === next.points.length &&
         prev.points.every(
           (point, i) =>
             point[0] === next.points[i][0] && point[1] === next.points[i][1]
         )
       );
     }
   );

   const OptimizedMap: React.FC = () => {
     const { bounds, center, zoom, tracks } = useMapState();

     const debouncedSetBounds = useCallback(
       debounce((bounds: LatLngBounds) => {
         useMapState.setState({ bounds });
       }, 100),
       []
     );

     // Memoize track points for rendering
     const trackPoints = useMemo(
       () =>
         tracks.map((track) => ({
           id: track.id,
           points: track.points.map((p) => [p.latitude, p.longitude]),
         })),
       [tracks]
     );

     return (
       <MapContainer
         className="h-full w-full"
         whenCreated={(map) => {
           map.on("moveend", () => {
             debouncedSetBounds(map.getBounds());
           });
         }}
       >
         <TileLayer
           url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"
           maxZoom={19}
         />

         <MapUpdater center={center} zoom={zoom} bounds={bounds} />

         {trackPoints.map(({ id, points }) => (
           <TrackPathLayer key={id} points={points} />
         ))}
       </MapContainer>
     );
   };

   export default React.memo(OptimizedMap);
   ```

## Part 3: State Management and Code Splitting

1. Optimized State Management:
   frontend/src/stores/optimizedStore.ts:

   ```typescript
   import create from "zustand";
   import { devtools, persist } from "zustand/middleware";
   import { immer } from "zustand/middleware/immer";
   import { Track, CoverageStats } from "@/types";

   interface AppState {
     // Normalized state structure
     tracks: {
       byId: Record<string, Track>;
       allIds: string[];
       selectedId: string | null;
       loadingIds: string[];
     };
     coverage: {
       stats: CoverageStats | null;
       lastUpdated: number | null;
       loading: boolean;
     };
     ui: {
       sidebarOpen: boolean;
       activeView: "map" | "list" | "stats";
       filters: {
         dateRange: [Date | null, Date | null];
         minDistance: number | null;
         maxDistance: number | null;
       };
     };

     // Actions
     actions: {
       // Track actions
       addTrack: (track: Track) => void;
       updateTrack: (id: string, updates: Partial<Track>) => void;
       removeTrack: (id: string) => void;
       setSelectedTrack: (id: string | null) => void;
       setTrackLoading: (id: string, loading: boolean) => void;

       // Coverage actions
       updateCoverageStats: (stats: CoverageStats) => void;
       setCoverageLoading: (loading: boolean) => void;

       // UI actions
       toggleSidebar: () => void;
       setActiveView: (view: "map" | "list" | "stats") => void;
       updateFilters: (updates: Partial<AppState["ui"]["filters"]>) => void;
     };
   }

   export const useStore = create<AppState>(
     devtools(
       persist(
         immer((set) => ({
           tracks: {
             byId: {},
             allIds: [],
             selectedId: null,
             loadingIds: [],
           },
           coverage: {
             stats: null,
             lastUpdated: null,
             loading: false,
           },
           ui: {
             sidebarOpen: true,
             activeView: "map",
             filters: {
               dateRange: [null, null],
               minDistance: null,
               maxDistance: null,
             },
           },

           actions: {
             addTrack: (track) =>
               set((state) => {
                 state.tracks.byId[track.id] = track;
                 if (!state.tracks.allIds.includes(track.id)) {
                   state.tracks.allIds.push(track.id);
                 }
               }),

             updateTrack: (id, updates) =>
               set((state) => {
                 if (state.tracks.byId[id]) {
                   state.tracks.byId[id] = {
                     ...state.tracks.byId[id],
                     ...updates,
                   };
                 }
               }),

             removeTrack: (id) =>
               set((state) => {
                 delete state.tracks.byId[id];
                 state.tracks.allIds = state.tracks.allIds.filter(
                   (trackId) => trackId !== id
                 );
                 if (state.tracks.selectedId === id) {
                   state.tracks.selectedId = null;
                 }
               }),

             setSelectedTrack: (id) =>
               set((state) => {
                 state.tracks.selectedId = id;
               }),

             setTrackLoading: (id, loading) =>
               set((state) => {
                 if (loading) {
                   state.tracks.loadingIds.push(id);
                 } else {
                   state.tracks.loadingIds = state.tracks.loadingIds.filter(
                     (loadingId) => loadingId !== id
                   );
                 }
               }),

             updateCoverageStats: (stats) =>
               set((state) => {
                 state.coverage.stats = stats;
                 state.coverage.lastUpdated = Date.now();
               }),

             setCoverageLoading: (loading) =>
               set((state) => {
                 state.coverage.loading = loading;
               }),

             toggleSidebar: () =>
               set((state) => {
                 state.ui.sidebarOpen = !state.ui.sidebarOpen;
               }),

             setActiveView: (view) =>
               set((state) => {
                 state.ui.activeView = view;
               }),

             updateFilters: (updates) =>
               set((state) => {
                 state.ui.filters = {
                   ...state.ui.filters,
                   ...updates,
                 };
               }),
           },
         })),
         {
           name: "gpx-tracker-storage",
           partialize: (state) => ({
             ui: state.ui,
             // Only persist UI state, not data
           }),
         }
       )
     )
   );

   // Selectors
   export const selectTrackById = (id: string) =>
     useStore((state) => state.tracks.byId[id]);

   export const selectFilteredTracks = () =>
     useStore((state) => {
       const tracks = state.tracks.allIds.map((id) => state.tracks.byId[id]);
       const filters = state.ui.filters;

       return tracks.filter((track) => {
         if (filters.dateRange[0] && track.createdAt < filters.dateRange[0]) {
           return false;
         }
         if (filters.dateRange[1] && track.createdAt > filters.dateRange[1]) {
           return false;
         }
         if (filters.minDistance && track.totalDistance < filters.minDistance) {
           return false;
         }
         if (filters.maxDistance && track.totalDistance > filters.maxDistance) {
           return false;
         }
         return true;
       });
     });
   ```

## Part 4: Code Splitting and Dynamic Imports

1. Route Configuration with Code Splitting:
   frontend/src/routes/index.tsx:

   ```typescript
   import React, { Suspense } from "react";
   import { Routes, Route } from "react-router-dom";
   import LoadingSpinner from "@/components/common/LoadingSpinner";
   import ErrorBoundary from "@/components/common/ErrorBoundary";

   // Lazy load route components
   const Dashboard = React.lazy(() => import("@/pages/Dashboard"));
   const TrackList = React.lazy(() => import("@/pages/TrackList"));
   const TrackDetail = React.lazy(() => import("@/pages/TrackDetail"));
   const MapView = React.lazy(() => import("@/pages/MapView"));
   const Statistics = React.lazy(() => import("@/pages/Statistics"));
   const Settings = React.lazy(() => import("@/pages/Settings"));

   // Lazy loading wrapper with error boundary
   const LazyWrapper: React.FC<{ children: React.ReactNode }> = ({
     children,
   }) => (
     <ErrorBoundary>
       <Suspense
         fallback={
           <div className="flex items-center justify-center h-screen">
             <LoadingSpinner size="large" />
           </div>
         }
       >
         {children}
       </Suspense>
     </ErrorBoundary>
   );

   const AppRoutes: React.FC = () => {
     return (
       <Routes>
         <Route
           path="/"
           element={
             <LazyWrapper>
               <Dashboard />
             </LazyWrapper>
           }
         />
         <Route
           path="/tracks"
           element={
             <LazyWrapper>
               <TrackList />
             </LazyWrapper>
           }
         />
         <Route
           path="/tracks/:id"
           element={
             <LazyWrapper>
               <TrackDetail />
             </LazyWrapper>
           }
         />
         <Route
           path="/map"
           element={
             <LazyWrapper>
               <MapView />
             </LazyWrapper>
           }
         />
         <Route
           path="/stats"
           element={
             <LazyWrapper>
               <Statistics />
             </LazyWrapper>
           }
         />
         <Route
           path="/settings"
           element={
             <LazyWrapper>
               <Settings />
             </LazyWrapper>
           }
         />
       </Routes>
     );
   };

   export default AppRoutes;
   ```

2. Dynamic Component Loading Utilities:
   frontend/src/utils/dynamicImport.ts:

   ```typescript
   import React from "react";

   interface ImportOptions {
     ssr?: boolean;
     loading?: React.ComponentType;
     error?: React.ComponentType<{ error: Error }>;
   }

   export function loadComponent<T = any>(
     importFn: () => Promise<{ default: React.ComponentType<T> }>,
     options: ImportOptions = {}
   ) {
     const {
       loading: LoadingComponent = () => null,
       error: ErrorComponent = () => null,
       ssr = false,
     } = options;

     const LazyComponent = React.lazy(importFn);

     return function DynamicComponent(props: T) {
       if (!ssr && typeof window === "undefined") {
         return null;
       }

       return (
         <React.Suspense fallback={<LoadingComponent />}>
           <ErrorBoundary fallback={ErrorComponent}>
             <LazyComponent {...props} />
           </ErrorBoundary>
         </React.Suspense>
       );
     };
   }

   // Usage example:
   export const DynamicMap = loadComponent(() => import("@/components/Map"), {
     loading: () => <div>Loading map...</div>,
     error: ({ error }) => <div>Error loading map: {error.message}</div>,
   });
   ```

3. Dynamic Feature Loading:
   frontend/src/features/index.ts:

   ```typescript
   import { loadComponent } from "@/utils/dynamicImport";

   // Map features
   export const MapControls = loadComponent(() => import("./map/MapControls"));

   export const TrackLayer = loadComponent(() => import("./map/TrackLayer"));

   export const HeatmapLayer = loadComponent(
     () => import("./map/HeatmapLayer"),
     { ssr: false } // Don't render on server
   );

   // Chart features
   export const ElevationChart = loadComponent(
     () => import("./charts/ElevationChart")
   );

   export const StatisticsCharts = loadComponent(
     () => import("./charts/StatisticsCharts")
   );

   // Advanced features loaded on demand
   export const AdvancedAnalysis = loadComponent(
     () => import("./analysis/AdvancedAnalysis"),
     { ssr: false }
   );

   export const RouteOptimizer = loadComponent(
     () => import("./analysis/RouteOptimizer"),
     { ssr: false }
   );
   ```

4. Webpack Bundle Optimization:
   frontend/vite.config.ts:

   ```typescript
   import { defineConfig } from "vite";
   import react from "@vitejs/plugin-react";
   import { visualizer } from "rollup-plugin-visualizer";
   import { splitVendorChunkPlugin } from "vite";

   export default defineConfig({
     plugins: [
       react(),
       splitVendorChunkPlugin(),
       visualizer({
         filename: "./dist/stats.html",
         open: true,
         gzipSize: true,
       }),
     ],
     build: {
       target: "esnext",
       sourcemap: true,
       rollupOptions: {
         output: {
           manualChunks: {
             "react-vendor": ["react", "react-dom", "react-router-dom"],
             "map-vendor": ["leaflet", "react-leaflet"],
             "chart-vendor": ["recharts", "d3-scale"],
             "utils-vendor": ["date-fns", "lodash"],
           },
         },
       },
       chunkSizeWarningLimit: 1000,
     },
     optimizeDeps: {
       include: ["react", "react-dom", "react-router-dom"],
     },
   });
   ```

## Part 5: Caching Strategies

1. API Cache Implementation:
   frontend/src/services/cache.ts:

   ```typescript
   import { QueryClient } from "@tanstack/react-query";

   export const queryClient = new QueryClient({
     defaultOptions: {
       queries: {
         // Global cache configuration
         staleTime: 5 * 60 * 1000, // Data is fresh for 5 minutes
         cacheTime: 30 * 60 * 1000, // Cache is kept for 30 minutes
         retry: 2,
         refetchOnWindowFocus: false,
         refetchOnReconnect: true,
       },
     },
   });

   // Cache prefetching strategies
   export const prefetchQueries = async (userId: string) => {
     await Promise.all([
       // Prefetch user's tracks
       queryClient.prefetchQuery(["tracks", userId], () =>
         api.getTracks(userId)
       ),
       // Prefetch coverage data
       queryClient.prefetchQuery(["coverage", userId], () =>
         api.getCoverage(userId)
       ),
       // Prefetch recent statistics
       queryClient.prefetchQuery(["statistics", userId], () =>
         api.getStatistics(userId)
       ),
     ]);
   };

   // Cache persistence
   export const persistCache = async () => {
     const cache = queryClient.getQueryData(["tracks"]);
     localStorage.setItem("queryCache", JSON.stringify(cache));
   };

   export const hydrateCacheFromStorage = () => {
     const cache = localStorage.getItem("queryCache");
     if (cache) {
       queryClient.setQueryData(["tracks"], JSON.parse(cache));
     }
   };
   ```

2. Service Worker Caching:
   frontend/src/service-worker.ts:

   ```typescript
   /// <reference lib="webworker" />

   import { precacheAndRoute } from "workbox-precaching";
   import { registerRoute } from "workbox-routing";
   import {
     NetworkFirst,
     StaleWhileRevalidate,
     CacheFirst,
   } from "workbox-strategies";
   import { ExpirationPlugin } from "workbox-expiration";
   import { CacheableResponsePlugin } from "workbox-cacheable-response";

   declare const self: ServiceWorkerGlobalScope;

   // Precache all assets generated by your build process
   precacheAndRoute(self.__WB_MANIFEST);

   // API routes caching
   registerRoute(
     ({ url }) => url.pathname.startsWith("/api/"),
     new NetworkFirst({
       cacheName: "api-cache",
       plugins: [
         new CacheableResponsePlugin({
           statuses: [0, 200],
         }),
         new ExpirationPlugin({
           maxEntries: 100,
           maxAgeSeconds: 30 * 60, // 30 minutes
         }),
       ],
     })
   );

   // Map tiles caching
   registerRoute(
     ({ url }) => url.href.includes("tile.openstreetmap.org"),
     new CacheFirst({
       cacheName: "map-tiles",
       plugins: [
         new CacheableResponsePlugin({
           statuses: [0, 200],
         }),
         new ExpirationPlugin({
           maxEntries: 500,
           maxAgeSeconds: 30 * 24 * 60 * 60, // 30 days
         }),
       ],
     })
   );

   // Static assets caching
   registerRoute(
     ({ request }) =>
       request.destination === "style" ||
       request.destination === "script" ||
       request.destination === "image",
     new StaleWhileRevalidate({
       cacheName: "static-resources",
       plugins: [
         new CacheableResponsePlugin({
           statuses: [0, 200],
         }),
         new ExpirationPlugin({
           maxEntries: 100,
           maxAgeSeconds: 7 * 24 * 60 * 60, // 7 days
         }),
       ],
     })
   );

   // Offline fallback
   const FALLBACK_HTML = "/offline.html";

   self.addEventListener("install", (event) => {
     event.waitUntil(
       caches.open("offline-fallback").then((cache) => {
         return cache.add(FALLBACK_HTML);
       })
     );
   });

   self.addEventListener("fetch", (event) => {
     if (event.request.mode === "navigate") {
       event.respondWith(
         fetch(event.request).catch(() => {
           return caches.match(FALLBACK_HTML);
         })
       );
     }
   });
   ```

3. Cache Control Headers:
   backend/app/middleware/cache.py:

   ```python
   from fastapi import Request, Response
   from typing import Callable
   import time

   async def cache_middleware(request: Request, call_next: Callable):
       """Add cache control headers based on route and content"""
       response = await call_next(request)

       # Define cache rules based on path and method
       if request.method in ['GET', 'HEAD']:
           if 'tracks' in request.url.path:
               # Cache track data for 5 minutes
               response.headers['Cache-Control'] = 'public, max-age=300'
           elif 'coverage' in request.url.path:
               # Cache coverage data for 1 hour
               response.headers['Cache-Control'] = 'public, max-age=3600'
           elif '/static/' in request.url.path:
               # Cache static files for 1 week
               response.headers['Cache-Control'] = 'public, max-age=604800'
           else:
               # Default no-cache for dynamic content
               response.headers['Cache-Control'] = 'no-cache'

       # Add ETag header for caching
       response.headers['ETag'] = f""{hash(str(time.time()))}""

       return response
   ```

## Part 6: Network and Resource Optimizations

1. Network Request Optimization:
   frontend/src/services/api/optimizedClient.ts:

   ```typescript
   import axios from "axios";
   import { retryWithBackoff } from "@/utils/network";
   import { batchRequests } from "@/utils/batchRequests";

   const api = axios.create({
     baseURL: import.meta.env.VITE_API_URL,
     timeout: 10000,
     headers: {
       Accept: "application/json",
       "Content-Type": "application/json",
     },
   });

   // Request batching implementation
   interface BatchConfig {
     maxBatchSize: number;
     batchDelay: number;
   }

   class BatchManager {
     private batchQueue: Map<string, any[]> = new Map();
     private timeouts: Map<string, NodeJS.Timeout> = new Map();

     constructor(private config: BatchConfig) {}

     async addToBatch(key: string, request: any): Promise<any> {
       if (!this.batchQueue.has(key)) {
         this.batchQueue.set(key, []);
       }

       const queue = this.batchQueue.get(key)!;
       queue.push(request);

       if (queue.length >= this.config.maxBatchSize) {
         return this.processBatch(key);
       }

       // Set timeout for batch processing
       if (this.timeouts.has(key)) {
         clearTimeout(this.timeouts.get(key)!);
       }

       return new Promise((resolve) => {
         this.timeouts.set(
           key,
           setTimeout(() => {
             resolve(this.processBatch(key));
           }, this.config.batchDelay)
         );
       });
     }

     private async processBatch(key: string): Promise<any> {
       const queue = this.batchQueue.get(key) || [];
       this.batchQueue.delete(key);
       this.timeouts.delete(key);

       if (queue.length === 0) return;

       return api.post(`/batch/${key}`, { requests: queue });
     }
   }

   const batchManager = new BatchManager({
     maxBatchSize: 10,
     batchDelay: 50,
   });

   // Optimized request interceptor
   api.interceptors.request.use(async (config) => {
     // Add compression headers
     config.headers["Accept-Encoding"] = "gzip, deflate, br";

     // Add connection optimization headers
     if (navigator.connection) {
       const connection = navigator.connection as any;
       config.headers["Save-Data"] = connection.saveData ? "on" : "off";
       if (connection.effectiveType) {
         config.headers["Network-Connection"] = connection.effectiveType;
       }
     }

     return config;
   });

   // Response compression handling
   api.interceptors.response.use(
     (response) => {
       if (response.headers["content-encoding"]) {
         // Handle compressed responses
         return decompress(response.data);
       }
       return response;
     },
     async (error) => {
       if (error.response?.status === 429) {
         // Handle rate limiting with exponential backoff
         return retryWithBackoff(() => {
           return api(error.config);
         });
       }
       throw error;
     }
   );
   ```

2. Resource Loading Optimization:
   frontend/src/utils/resourceLoader.ts:

   ```typescript
   export class ResourceLoader {
     private loadedResources: Set<string> = new Set();
     private loading: Map<string, Promise<any>> = new Map();

     async preloadResource(
       url: string,
       type: "image" | "style" | "script"
     ): Promise<void> {
       if (this.loadedResources.has(url)) return;

       if (this.loading.has(url)) {
         return this.loading.get(url);
       }

       const loadPromise = new Promise((resolve, reject) => {
         switch (type) {
           case "image": {
             const img = new Image();
             img.onload = () => {
               this.loadedResources.add(url);
               resolve(undefined);
             };
             img.onerror = reject;
             img.src = url;
             break;
           }
           case "style": {
             const link = document.createElement("link");
             link.rel = "preload";
             link.as = "style";
             link.href = url;
             link.onload = () => {
               this.loadedResources.add(url);
               resolve(undefined);
             };
             link.onerror = reject;
             document.head.appendChild(link);
             break;
           }
           case "script": {
             const script = document.createElement("link");
             script.rel = "preload";
             script.as = "script";
             script.href = url;
             script.onload = () => {
               this.loadedResources.add(url);
               resolve(undefined);
             };
             script.onerror = reject;
             document.head.appendChild(script);
             break;
           }
         }
       });

       this.loading.set(url, loadPromise);
       return loadPromise;
     }

     async preloadResources(
       resources: Array<{ url: string; type: "image" | "style" | "script" }>
     ): Promise<void> {
       await Promise.all(
         resources.map(({ url, type }) => this.preloadResource(url, type))
       );
     }
   }

   export const resourceLoader = new ResourceLoader();
   ```

3. Progressive Loading Implementation:
   frontend/src/components/ProgressiveImage.tsx:

   ```typescript
   import React, { useState, useEffect } from "react";
   import { resourceLoader } from "@/utils/resourceLoader";

   interface ProgressiveImageProps {
     src: string;
     placeholderSrc: string;
     alt: string;
     className?: string;
   }

   const ProgressiveImage: React.FC<ProgressiveImageProps> = ({
     src,
     placeholderSrc,
     alt,
     className,
   }) => {
     const [currentSrc, setCurrentSrc] = useState(placeholderSrc);
     const [loading, setLoading] = useState(true);

     useEffect(() => {
       resourceLoader
         .preloadResource(src, "image")
         .then(() => {
           setCurrentSrc(src);
           setLoading(false);
         })
         .catch((error) => {
           console.error("Error loading image:", error);
           setLoading(false);
         });
     }, [src]);

     return (
       <img
         src={currentSrc}
         alt={alt}
         className={`transition-opacity duration-300 ${
           loading ? "opacity-50" : "opacity-100"
         } ${className}`}
       />
     );
   };

   export default React.memo(ProgressiveImage);
   ```
