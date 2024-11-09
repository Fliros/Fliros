# Map Visualization Implementation

## Part 1: Base Setup

1. Create Map Components Structure:

   ```bash
   mkdir -p src/features/map/components
   touch src/features/map/Map.tsx
   touch src/features/map/components/MapControls.tsx
   touch src/features/map/components/AreaOverlay.tsx
   touch src/features/map/components/TrackOverlay.tsx
   touch src/features/map/components/InfoPopup.tsx
   touch src/features/map/hooks/useMapState.ts
   ```

2. Install Additional Dependencies:

   ```bash
   npm install leaflet @types/leaflet react-leaflet @turf/turf
   ```

3. Map Component Setup:
   src/features/map/Map.tsx:

   ```typescript
   import React, { useEffect } from "react";
   import {
     MapContainer,
     TileLayer,
     ZoomControl,
     ScaleControl,
   } from "react-leaflet";
   import { LatLngBounds, LatLng } from "leaflet";
   import { useQuery } from "@tanstack/react-query";
   import { useSettingsStore } from "@/stores/settingsStore";
   import MapControls from "./components/MapControls";
   import AreaOverlay from "./components/AreaOverlay";
   import TrackOverlay from "./components/TrackOverlay";
   import { useMapState } from "./hooks/useMapState";
   import "leaflet/dist/leaflet.css";

   const DEFAULT_CENTER: [number, number] = [0, 0];
   const DEFAULT_ZOOM = 2;
   const MIN_ZOOM = 2;
   const MAX_ZOOM = 18;

   const Map: React.FC = () => {
     const { mapStyle } = useSettingsStore();
     const { bounds, setBounds, selectedTrackId } = useMapState();

     // Fetch coverage data
     const { data: coverageData } = useQuery(["coverage-geojson"], async () => {
       const response = await api.get("/areas/coverage/details");
       return response.data;
     });

     // Fetch selected track data if any
     const { data: trackData } = useQuery(
       ["track", selectedTrackId],
       async () => {
         if (!selectedTrackId) return null;
         const response = await api.get(`/tracks/${selectedTrackId}/points`);
         return response.data;
       },
       { enabled: !!selectedTrackId }
     );

     const getTileLayer = () => {
       switch (mapStyle) {
         case "satellite":
           return {
             url: "https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}",
             attribution:
               "Tiles &copy; Esri &mdash; Source: Esri, i-cubed, USDA, USGS, AEX, GeoEye, Getmapping, Aerogrid, IGN, IGP, UPR-EGP, and the GIS User Community",
           };
         case "terrain":
           return {
             url: "https://{s}.tile.opentopomap.org/{z}/{x}/{y}.png",
             attribution:
               "Map data: &copy; OpenStreetMap contributors, SRTM | Map style: &copy; OpenTopoMap",
           };
         default:
           return {
             url: "https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",
             attribution:
               '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
           };
       }
     };

     const handleBoundsChange = (newBounds: LatLngBounds) => {
       setBounds(newBounds);
     };

     return (
       <div className="h-[calc(100vh-4rem)] relative">
         <MapContainer
           center={DEFAULT_CENTER}
           zoom={DEFAULT_ZOOM}
           minZoom={MIN_ZOOM}
           maxZoom={MAX_ZOOM}
           className="h-full w-full"
           zoomControl={false}
           whenReady={({ target }) => handleBoundsChange(target.getBounds())}
           whenMoveEnd={({ target }) => handleBoundsChange(target.getBounds())}
         >
           <TileLayer {...getTileLayer()} />
           <ZoomControl position="topright" />
           <ScaleControl position="bottomleft" imperial={false} />

           {coverageData && (
             <AreaOverlay geojson={coverageData.geojson} bounds={bounds} />
           )}

           {trackData && (
             <TrackOverlay points={trackData} selected={!!selectedTrackId} />
           )}

           <MapControls />
         </MapContainer>
       </div>
     );
   };

   export default Map;
   ```

4. Map State Hook:
   src/features/map/hooks/useMapState.ts:

   ```typescript
   import { create } from "zustand";
   import { LatLngBounds } from "leaflet";

   interface MapState {
     bounds: LatLngBounds | null;
     selectedTrackId: string | null;
     selectedArea: GeoJSON.Feature | null;
     setBounds: (bounds: LatLngBounds) => void;
     setSelectedTrackId: (id: string | null) => void;
     setSelectedArea: (area: GeoJSON.Feature | null) => void;
   }

   export const useMapState = create<MapState>((set) => ({
     bounds: null,
     selectedTrackId: null,
     selectedArea: null,
     setBounds: (bounds) => set({ bounds }),
     setSelectedTrackId: (id) => set({ selectedTrackId: id }),
     setSelectedArea: (area) => set({ selectedArea: area }),
   }));
   ```

## Part 2: Overlay Components

1. Area Overlay Component:
   src/features/map/components/AreaOverlay.tsx:

   ```typescript
   import React, { useMemo } from "react";
   import { GeoJSON, Tooltip } from "react-leaflet";
   import { LatLngBounds } from "leaflet";
   import * as turf from "@turf/turf";
   import { useMapState } from "../hooks/useMapState";

   interface AreaOverlayProps {
     geojson: GeoJSON.FeatureCollection;
     bounds: LatLngBounds | null;
   }

   const AreaOverlay: React.FC<AreaOverlayProps> = ({ geojson, bounds }) => {
     const { setSelectedArea } = useMapState();

     const filteredGeojson = useMemo(() => {
       if (!bounds) return geojson;

       // Create a bounding box for filtering
       const bbox = [
         bounds.getWest(),
         bounds.getSouth(),
         bounds.getEast(),
         bounds.getNorth(),
       ];

       // Filter features within the current bounds
       const filtered = turf.bboxClip(geojson, bbox as turf.BBox);

       return filtered;
     }, [geojson, bounds]);

     const getAreaStyle = (feature: GeoJSON.Feature) => ({
       fillColor: "#3b82f6",
       fillOpacity: 0.3,
       color: "#2563eb",
       weight: 2,
       opacity: 0.8,
     });

     const handleAreaClick = (feature: GeoJSON.Feature) => {
       setSelectedArea(feature);
     };

     const calculateArea = (feature: GeoJSON.Feature) => {
       const area = turf.area(feature);
       return (area / 1000000).toFixed(2); // Convert to km²
     };

     return (
       <GeoJSON
         data={filteredGeojson}
         style={getAreaStyle}
         eventHandlers={{
           click: (e) => handleAreaClick(e.target.feature),
         }}
       >
         <Tooltip sticky>
           {(feature) => (
             <div className="text-sm">
               <strong>Area:</strong> {calculateArea(feature)} km²
             </div>
           )}
         </Tooltip>
       </GeoJSON>
     );
   };

   export default AreaOverlay;
   ```

2. Track Overlay Component:
   src/features/map/components/TrackOverlay.tsx:

   ```typescript
   import React, { useMemo } from "react";
   import { Polyline, Marker, Popup } from "react-leaflet";
   import { Icon } from "leaflet";
   import { formatDistance, formatDate } from "@/utils/formatting";

   interface TrackPoint {
     latitude: number;
     longitude: number;
     elevation: number;
     time: string;
   }

   interface TrackOverlayProps {
     points: TrackPoint[];
     selected: boolean;
   }

   const TrackOverlay: React.FC<TrackOverlayProps> = ({ points, selected }) => {
     const positions = useMemo(
       () => points.map((p) => [p.latitude, p.longitude] as [number, number]),
       [points]
     );

     const startIcon = new Icon({
       iconUrl: "/icons/start-marker.svg",
       iconSize: [32, 32],
       iconAnchor: [16, 32],
     });

     const endIcon = new Icon({
       iconUrl: "/icons/end-marker.svg",
       iconSize: [32, 32],
       iconAnchor: [16, 32],
     });

     if (!positions.length) return null;

     return (
       <>
         <Polyline
           positions={positions}
           color={selected ? "#2563eb" : "#64748b"}
           weight={selected ? 4 : 2}
           opacity={selected ? 1 : 0.7}
         />

         <Marker position={positions[0]} icon={startIcon}>
           <Popup>
             <div className="text-sm">
               <strong>Start Point</strong>
               <br />
               Time: {formatDate(points[0].time)}
               <br />
               Elevation: {points[0].elevation}m
             </div>
           </Popup>
         </Marker>

         <Marker position={positions[positions.length - 1]} icon={endIcon}>
           <Popup>
             <div className="text-sm">
               <strong>End Point</strong>
               <br />
               Time: {formatDate(points[points.length - 1].time)}
               <br />
               Elevation: {points[points.length - 1].elevation}m
             </div>
           </Popup>
         </Marker>
       </>
     );
   };

   export default TrackOverlay;
   ```

3. Track Cluster Component:
   src/features/map/components/TrackCluster.tsx:

   ```typescript
   import React from "react";
   import { Marker, Circle } from "react-leaflet";
   import { divIcon } from "leaflet";

   interface TrackClusterProps {
     position: [number, number];
     pointCount: number;
     clusterRadius: number;
   }

   const TrackCluster: React.FC<TrackClusterProps> = ({
     position,
     pointCount,
     clusterRadius,
   }) => {
     const getClusterSize = (count: number) => {
       if (count < 10) return "small";
       if (count < 100) return "medium";
       return "large";
     };

     const createClusterIcon = (count: number) => {
       const size = getClusterSize(count);
       const iconSize = {
         small: 40,
         medium: 50,
         large: 60,
       }[size];

       return divIcon({
         html: `<div class="cluster-icon cluster-icon-${size}">${count}</div>`,
         className: "custom-cluster-icon",
         iconSize: [iconSize, iconSize],
         iconAnchor: [iconSize / 2, iconSize / 2],
       });
     };

     return (
       <>
         <Circle
           center={position}
           radius={clusterRadius}
           fillColor="#2563eb"
           fillOpacity={0.1}
           stroke={false}
         />
         <Marker
           position={position}
           icon={createClusterIcon(pointCount)}
           eventHandlers={{
             click: () => {
               // Handle cluster click - e.g., zoom to bounds
             },
           }}
         />
       </>
     );
   };

   export default TrackCluster;
   ```

4. Elevation Profile Overlay:
   src/features/map/components/ElevationOverlay.tsx:

   ```typescript
   import React, { useMemo } from "react";
   import { Polyline, Tooltip } from "react-leaflet";
   import * as turf from "@turf/turf";

   interface ElevationOverlayProps {
     points: Array<{
       latitude: number;
       longitude: number;
       elevation: number;
     }>;
     selected: boolean;
   }

   const ElevationOverlay: React.FC<ElevationOverlayProps> = ({
     points,
     selected,
   }) => {
     const colorGradient = useMemo(() => {
       const elevations = points.map((p) => p.elevation);
       const minElevation = Math.min(...elevations);
       const maxElevation = Math.max(...elevations);
       const range = maxElevation - minElevation;

       return points
         .map((point, index) => {
           if (index === points.length - 1) return null;

           const startPercent = (point.elevation - minElevation) / range;
           const endPercent =
             (points[index + 1].elevation - minElevation) / range;

           return {
             start: point,
             end: points[index + 1],
             color: `hsl(${200 + startPercent * 60}, 70%, ${
               50 - startPercent * 20
             }%)`,
             gradient: endPercent - startPercent,
           };
         })
         .filter(Boolean);
     }, [points]);

     return (
       <>
         {colorGradient.map((segment, index) => (
           <Polyline
             key={index}
             positions={[
               [segment.start.latitude, segment.start.longitude],
               [segment.end.latitude, segment.end.longitude],
             ]}
             color={segment.color}
             weight={selected ? 4 : 2}
             opacity={selected ? 1 : 0.7}
           >
             <Tooltip sticky>
               <div className="text-sm">
                 <strong>Elevation:</strong>{" "}
                 {Math.round(segment.start.elevation)}m
                 <br />
                 <strong>Gradient:</strong>{" "}
                 {(segment.gradient * 100).toFixed(1)}%
               </div>
             </Tooltip>
           </Polyline>
         ))}
       </>
     );
   };

   export default ElevationOverlay;
   ```

5. Distance Markers Overlay:
   src/features/map/components/DistanceMarkersOverlay.tsx:

   ```typescript
   import React, { useMemo } from "react";
   import { Circle, Tooltip } from "react-leaflet";
   import * as turf from "@turf/turf";
   import { formatDistance } from "@/utils/formatting";

   interface DistanceMarkersOverlayProps {
     points: Array<{
       latitude: number;
       longitude: number;
     }>;
     interval: number; // Distance interval in meters
   }

   const DistanceMarkersOverlay: React.FC<DistanceMarkersOverlayProps> = ({
     points,
     interval = 1000, // Default to 1km intervals
   }) => {
     const markers = useMemo(() => {
       const lineString = turf.lineString(
         points.map((p) => [p.longitude, p.latitude])
       );
       const length = turf.length(lineString, { units: "meters" });
       const markerCount = Math.floor(length / interval);

       return Array.from({ length: markerCount }, (_, index) => {
         const distance = interval * (index + 1);
         const point = turf.along(lineString, distance, { units: "meters" });
         return {
           position: point.geometry.coordinates,
           distance,
         };
       });
     }, [points, interval]);

     return (
       <>
         {markers.map((marker, index) => (
           <Circle
             key={index}
             center={[marker.position[1], marker.position[0]]}
             radius={5}
             color="#2563eb"
             fillColor="#ffffff"
             fillOpacity={1}
             weight={2}
           >
             <Tooltip permanent direction="top" offset={[0, -10]}>
               <div className="text-xs font-medium">
                 {formatDistance(marker.distance)}
               </div>
             </Tooltip>
           </Circle>
         ))}
       </>
     );
   };

   export default DistanceMarkersOverlay;
   ```

6. Heat Map Overlay:
   src/features/map/components/HeatMapOverlay.tsx:

   ```typescript
   import React, { useEffect } from "react";
   import { useMap } from "react-leaflet";
   import "leaflet.heat";
   import { LatLng } from "leaflet";

   interface HeatMapOverlayProps {
     points: Array<{
       latitude: number;
       longitude: number;
       intensity?: number;
     }>;
     radius?: number;
     blur?: number;
     maxZoom?: number;
   }

   const HeatMapOverlay: React.FC<HeatMapOverlayProps> = ({
     points,
     radius = 25,
     blur = 15,
     maxZoom = 18,
   }) => {
     const map = useMap();

     useEffect(() => {
       const data = points.map(
         (p) =>
           [p.latitude, p.longitude, p.intensity || 1] as [
             number,
             number,
             number
           ]
       );

       const heatLayer = L.heatLayer(data, {
         radius,
         blur,
         maxZoom,
         gradient: {
           0.4: "#3b82f6",
           0.6: "#2563eb",
           0.8: "#1d4ed8",
           1.0: "#1e40af",
         },
       }).addTo(map);

       return () => {
         map.removeLayer(heatLayer);
       };
     }, [map, points, radius, blur, maxZoom]);

     return null;
   };

   export default HeatMapOverlay;
   ```

## Part 3: Controls and Interactivity

1. Complete Map Controls Component:
   src/features/map/components/MapControls.tsx (continued):

   ```typescript
               <option value="terrain">Terrain</option>
             </select>
           </div>

           <div className="border-t border-gray-200 pt-2">
             <label className="block text-sm font-medium text-gray-700 mb-1">
               Layer Visibility
             </label>
             <div className="space-y-1">
               <label className="flex items-center space-x-2">
                 <input
                   type="checkbox"
                   checked={showCoveredAreas}
                   onChange={(e) => setShowCoveredAreas(e.target.checked)}
                   className="rounded border-gray-300 text-blue-600 focus:ring-blue-500"
                 />
                 <span className="text-sm">Covered Areas</span>
               </label>
               <label className="flex items-center space-x-2">
                 <input
                   type="checkbox"
                   checked={showTracks}
                   onChange={(e) => setShowTracks(e.target.checked)}
                   className="rounded border-gray-300 text-blue-600 focus:ring-blue-500"
                 />
                 <span className="text-sm">Tracks</span>
               </label>
             </div>
           </div>
         </div>
       </div>
     );
   };

   export default MapControls;
   ```

2. Add Info Popup Component:
   src/features/map/components/InfoPopup.tsx:

   ```typescript
   import React from "react";
   import { Popup } from "react-leaflet";
   import { formatDate, formatDistance } from "@/utils/formatting";

   interface InfoPopupProps {
     position: [number, number];
     data: {
       type: "area" | "track";
       properties: any;
     };
     onClose?: () => void;
   }

   const InfoPopup: React.FC<InfoPopupProps> = ({
     position,
     data,
     onClose,
   }) => {
     const renderAreaInfo = (properties: any) => (
       <div className="space-y-2">
         <h3 className="font-medium">Area Information</h3>
         <div className="text-sm">
           <p>
             <strong>Total Area:</strong> {properties.area_km2.toFixed(2)} km²
           </p>
           <p>
             <strong>First Tracked:</strong>{" "}
             {formatDate(properties.first_tracked)}
           </p>
           <p>
             <strong>Last Updated:</strong>{" "}
             {formatDate(properties.last_updated)}
           </p>
         </div>
       </div>
     );

     const renderTrackInfo = (properties: any) => (
       <div className="space-y-2">
         <h3 className="font-medium">
           {properties.name || "Track Information"}
         </h3>
         <div className="text-sm">
           <p>
             <strong>Distance:</strong> {formatDistance(properties.distance)}
           </p>
           <p>
             <strong>Elevation Gain:</strong> {properties.elevation_gain}m
           </p>
           <p>
             <strong>Date:</strong> {formatDate(properties.date)}
           </p>
         </div>
       </div>
     );

     return (
       <Popup position={position} onClose={onClose}>
         <div className="min-w-[200px]">
           {data.type === "area"
             ? renderAreaInfo(data.properties)
             : renderTrackInfo(data.properties)}
         </div>
       </Popup>
     );
   };

   export default InfoPopup;
   ```

3. Add Map Context for Shared State:
   src/features/map/context/MapContext.tsx:

   ```typescript
   import React, { createContext, useContext, useState } from "react";
   import { LatLngBounds } from "leaflet";

   interface MapContextType {
     bounds: LatLngBounds | null;
     showCoveredAreas: boolean;
     showTracks: boolean;
     selectedFeature: any | null;
     setBounds: (bounds: LatLngBounds) => void;
     setShowCoveredAreas: (show: boolean) => void;
     setShowTracks: (show: boolean) => void;
     setSelectedFeature: (feature: any | null) => void;
   }

   const MapContext = createContext<MapContextType | undefined>(undefined);

   export const MapProvider: React.FC<{ children: React.ReactNode }> = ({
     children,
   }) => {
     const [bounds, setBounds] = useState<LatLngBounds | null>(null);
     const [showCoveredAreas, setShowCoveredAreas] = useState(true);
     const [showTracks, setShowTracks] = useState(true);
     const [selectedFeature, setSelectedFeature] = useState<any | null>(null);

     return (
       <MapContext.Provider
         value={{
           bounds,
           showCoveredAreas,
           showTracks,
           selectedFeature,
           setBounds,
           setShowCoveredAreas,
           setShowTracks,
           setSelectedFeature,
         }}
       >
         {children}
       </MapContext.Provider>
     );
   };

   export const useMapContext = () => {
     const context = useContext(MapContext);
     if (context === undefined) {
       throw new Error("useMapContext must be used within a MapProvider");
     }
     return context;
   };
   ```

4. Add Map Legend Component:
   src/features/map/components/MapLegend.tsx:

   ```typescript
   import React from "react";
   import { useMapContext } from "../context/MapContext";

   const MapLegend: React.FC = () => {
     const { showCoveredAreas, showTracks } = useMapContext();

     if (!showCoveredAreas && !showTracks) return null;

     return (
       <div className="absolute bottom-8 right-4 bg-white rounded-lg shadow-lg p-4 z-[1000]">
         <h4 className="text-sm font-medium mb-2">Legend</h4>
         <div className="space-y-2">
           {showCoveredAreas && (
             <div className="flex items-center space-x-2">
               <div className="w-6 h-4 bg-blue-500 opacity-30 border border-blue-600" />
               <span className="text-sm">Covered Areas</span>
             </div>
           )}
           {showTracks && (
             <div className="flex items-center space-x-2">
               <div className="w-6 h-4 flex items-center">
                 <div className="w-full h-0.5 bg-blue-600" />
               </div>
               <span className="text-sm">Tracks</span>
             </div>
           )}
         </div>
       </div>
     );
   };

   export default MapLegend;
   ```

## Part 4: Optimizations and Utilities

1. Map Utilities:
   src/features/map/utils/mapUtils.ts:

   ```typescript
   import * as turf from "@turf/turf";
   import { LatLngBounds, LatLng } from "leaflet";

   export const simplifyGeoJSON = (
     geojson: GeoJSON.FeatureCollection,
     tolerance: number = 0.0001
   ) => {
     const simplified = {
       type: "FeatureCollection",
       features: geojson.features.map((feature) => ({
         ...feature,
         geometry: turf.simplify(feature.geometry, {
           tolerance,
           highQuality: true,
         }),
       })),
     };
     return simplified;
   };

   export const filterGeoJSONByBounds = (
     geojson: GeoJSON.FeatureCollection,
     bounds: LatLngBounds
   ) => {
     const bbox = [
       bounds.getWest(),
       bounds.getSouth(),
       bounds.getEast(),
       bounds.getNorth(),
     ] as turf.BBox;

     return {
       type: "FeatureCollection",
       features: geojson.features.filter((feature) =>
         turf.booleanIntersects(feature, turf.bboxPolygon(bbox))
       ),
     };
   };

   export const calculateVisibleArea = (
     geojson: GeoJSON.FeatureCollection,
     bounds: LatLngBounds
   ) => {
     const filtered = filterGeoJSONByBounds(geojson, bounds);
     const area = filtered.features.reduce((acc, feature) => {
       return acc + turf.area(feature);
     }, 0);
     return area / 1000000; // Convert to km²
   };

   export const createClusterPoints = (
     points: Array<{ lat: number; lng: number }>,
     radius: number = 50
   ) => {
     const features = points.map((point) => turf.point([point.lng, point.lat]));
     const collection = turf.featureCollection(features);
     return turf.clustersKmeans(collection, {
       numberOfClusters: Math.ceil(points.length / 10),
       mutate: true,
     });
   };
   ```

2. Performance Optimizations Hook:
   src/features/map/hooks/useMapOptimizations.ts:

   ```typescript
   import { useState, useEffect, useCallback } from "react";
   import { LatLngBounds } from "leaflet";
   import { useDebounce } from "@/utils/performance";
   import { simplifyGeoJSON, filterGeoJSONByBounds } from "../utils/mapUtils";

   export const useMapOptimizations = (
     geojson: GeoJSON.FeatureCollection | null,
     bounds: LatLngBounds | null,
     zoom: number
   ) => {
     const [optimizedData, setOptimizedData] =
       useState<GeoJSON.FeatureCollection | null>(null);
     const debouncedBounds = useDebounce(bounds, 300);
     const debouncedZoom = useDebounce(zoom, 300);

     const getSimplificationTolerance = useCallback((zoom: number) => {
       if (zoom < 5) return 0.01;
       if (zoom < 10) return 0.001;
       if (zoom < 15) return 0.0001;
       return 0.00001;
     }, []);

     useEffect(() => {
       if (!geojson || !debouncedBounds) return;

       const optimize = async () => {
         // Filter by bounds
         const filtered = filterGeoJSONByBounds(geojson, debouncedBounds);

         // Simplify geometry based on zoom level
         const tolerance = getSimplificationTolerance(debouncedZoom);
         const simplified = simplifyGeoJSON(filtered, tolerance);

         setOptimizedData(simplified);
       };

       optimize();
     }, [geojson, debouncedBounds, debouncedZoom, getSimplificationTolerance]);

     return optimizedData;
   };
   ```

3. Track Clustering Hook:
   src/features/map/hooks/useTrackClustering.ts:

   ```typescript
   import { useState, useEffect } from "react";
   import { LatLngBounds } from "leaflet";
   import { createClusterPoints } from "../utils/mapUtils";

   export const useTrackClustering = (
     tracks: Array<{ id: string; lat: number; lng: number }>,
     bounds: LatLngBounds | null,
     zoom: number
   ) => {
     const [clusters, setClusters] = useState<any[]>([]);

     useEffect(() => {
       if (!tracks.length || !bounds) return;

       // Only cluster at lower zoom levels
       if (zoom > 12) {
         setClusters(
           tracks.map((track) => ({
             type: "single",
             ...track,
           }))
         );
         return;
       }

       // Filter visible tracks
       const visibleTracks = tracks.filter((track) =>
         bounds.contains([track.lat, track.lng])
       );

       // Create clusters
       const clustered = createClusterPoints(visibleTracks);
       const clusterData = clustered.features.map((feature) => ({
         type: "cluster",
         id: feature.properties.cluster,
         count: feature.properties.point_count,
         coordinates: feature.geometry.coordinates,
       }));

       setClusters(clusterData);
     }, [tracks, bounds, zoom]);

     return clusters;
   };
   ```

4. Update Map Component with Optimizations:
   src/features/map/Map.tsx (update):

```typescript
import React, { useState, useCallback, useRef } from "react";
import {
  MapContainer,
  TileLayer,
  ZoomControl,
  ScaleControl,
  useMap,
} from "react-leaflet";
import { useQuery } from "@tanstack/react-query";
import { useMapOptimizations } from "./hooks/useMapOptimizations";
import { useTrackClustering } from "./hooks/useTrackClustering";
import { useMapState } from "./hooks/useMapState";
import MapControls from "./components/MapControls";
import AreaOverlay from "./components/AreaOverlay";
import TrackOverlay from "./components/TrackOverlay";
import TrackCluster from "./components/TrackCluster";
import HeatMapOverlay from "./components/HeatMapOverlay";
import { debounce } from "@/utils/performance";

// Map update frequency limiter component
const MapUpdater = ({ onUpdate }: { onUpdate: (map: L.Map) => void }) => {
  const map = useMap();

  const debouncedUpdate = useCallback(
    debounce((m: L.Map) => onUpdate(m), 100),
    [onUpdate]
  );

  React.useEffect(() => {
    map.on("moveend", () => debouncedUpdate(map));
    map.on("zoomend", () => debouncedUpdate(map));

    return () => {
      map.off("moveend", () => debouncedUpdate(map));
      map.off("zoomend", () => debouncedUpdate(map));
    };
  }, [map, debouncedUpdate]);

  return null;
};

const Map: React.FC = () => {
  const mapRef = useRef<L.Map | null>(null);
  const [zoom, setZoom] = useState(DEFAULT_ZOOM);
  const { bounds, setBounds } = useMapState();

  const { data: coverageData, isLoading: isLoadingCoverage } = useQuery(
    ["coverage-geojson"],
    async () => {
      const response = await api.get("/areas/coverage/details");
      return response.data;
    },
    {
      staleTime: 5 * 60 * 1000, // 5 minutes
      cacheTime: 30 * 60 * 1000, // 30 minutes
    }
  );

  const { data: trackData, isLoading: isLoadingTracks } = useQuery(
    ["tracks-points"],
    async () => {
      const response = await api.get("/tracks/points");
      return response.data;
    },
    {
      staleTime: 5 * 60 * 1000,
      cacheTime: 30 * 60 * 1000,
    }
  );

  const optimizedCoverage = useMapOptimizations(
    coverageData?.geojson,
    bounds,
    zoom
  );

  const trackClusters = useTrackClustering(
    trackData?.points || [],
    bounds,
    zoom
  );

  const handleMapUpdate = useCallback(
    (map: L.Map) => {
      setZoom(map.getZoom());
      setBounds(map.getBounds());
      mapRef.current = map;
    },
    [setBounds]
  );

  const handleViewportChange = useCallback(() => {
    if (mapRef.current) {
      const map = mapRef.current;
      const center = map.getCenter();
      const zoom = map.getZoom();
      // Update URL with current view state
      const params = new URLSearchParams(window.location.search);
      params.set("lat", center.lat.toFixed(6));
      params.set("lng", center.lng.toFixed(6));
      params.set("zoom", zoom.toString());
      window.history.replaceState(
        {},
        "",
        `${window.location.pathname}?${params.toString()}`
      );
    }
  }, []);

  return (
    <div className="h-[calc(100vh-4rem)] relative">
      {(isLoadingCoverage || isLoadingTracks) && (
        <div className="absolute top-4 right-4 z-[1000] bg-white rounded-lg shadow-lg p-2">
          <div className="flex items-center space-x-2">
            <div className="animate-spin h-4 w-4 border-2 border-blue-600 rounded-full border-t-transparent" />
            <span className="text-sm">Loading data...</span>
          </div>
        </div>
      )}

      <MapContainer
        center={DEFAULT_CENTER}
        zoom={DEFAULT_ZOOM}
        minZoom={MIN_ZOOM}
        maxZoom={MAX_ZOOM}
        className="h-full w-full"
        zoomControl={false}
        whenCreated={(map) => {
          mapRef.current = map;
          handleMapUpdate(map);
        }}
      >
        <MapUpdater onUpdate={handleMapUpdate} />

        <TileLayer {...getTileLayer()} />
        <ZoomControl position="topright" />
        <ScaleControl position="bottomleft" imperial={false} />

        {optimizedCoverage && (
          <AreaOverlay geojson={optimizedCoverage} bounds={bounds} />
        )}

        {trackClusters.map((cluster) =>
          cluster.type === "cluster" ? (
            <TrackCluster
              key={cluster.id}
              position={[cluster.coordinates[1], cluster.coordinates[0]]}
              pointCount={cluster.count}
              clusterRadius={50}
            />
          ) : (
            <TrackOverlay
              key={cluster.id}
              points={cluster.points}
              selected={cluster.id === selectedTrackId}
            />
          )
        )}

        {showHeatmap && trackData && (
          <HeatMapOverlay points={trackData.points} radius={25} blur={15} />
        )}

        <MapControls />
      </MapContainer>
    </div>
  );
};

export default Map;
```

5. Viewport Management:
   src/features/map/hooks/useViewport.ts:

   ```typescript
   import { useState, useCallback, useEffect } from "react";
   import { LatLngBounds, LatLng } from "leaflet";

   interface Viewport {
     center: [number, number];
     zoom: number;
     bounds: LatLngBounds | null;
   }

   export const useViewport = (
     defaultCenter: [number, number],
     defaultZoom: number
   ) => {
     const [viewport, setViewport] = useState<Viewport>({
       center: defaultCenter,
       zoom: defaultZoom,
       bounds: null,
     });

     useEffect(() => {
       // Restore viewport from URL if available
       const params = new URLSearchParams(window.location.search);
       const lat = parseFloat(params.get("lat") || "");
       const lng = parseFloat(params.get("lng") || "");
       const zoom = parseInt(params.get("zoom") || "");

       if (!isNaN(lat) && !isNaN(lng) && !isNaN(zoom)) {
         setViewport((prev) => ({
           ...prev,
           center: [lat, lng],
           zoom,
         }));
       }
     }, []);

     const updateViewport = useCallback((newViewport: Partial<Viewport>) => {
       setViewport((prev) => ({
         ...prev,
         ...newViewport,
       }));
     }, []);

     return { viewport, updateViewport };
   };
   ```

6. Performance Monitoring:
   src/features/map/utils/performanceMonitoring.ts:

   ```typescript
   interface PerformanceMetrics {
     renderTime: number;
     featureCount: number;
     visibleFeatures: number;
     clusterCount: number;
   }

   class MapPerformanceMonitor {
     private metrics: PerformanceMetrics = {
       renderTime: 0,
       featureCount: 0,
       visibleFeatures: 0,
       clusterCount: 0,
     };

     private startTime: number = 0;

     startMeasurement() {
       this.startTime = performance.now();
     }

     endMeasurement() {
       this.metrics.renderTime = performance.now() - this.startTime;
     }

     updateMetrics(metrics: Partial<PerformanceMetrics>) {
       this.metrics = { ...this.metrics, ...metrics };
     }

     getMetrics(): PerformanceMetrics {
       return this.metrics;
     }

     logMetrics() {
       console.log("Map Performance Metrics:", {
         renderTime: `${this.metrics.renderTime.toFixed(2)}ms`,
         featureCount: this.metrics.featureCount,
         visibleFeatures: this.metrics.visibleFeatures,
         clusterCount: this.metrics.clusterCount,
       });
     }
   }

   export const performanceMonitor = new MapPerformanceMonitor();
   ```

7. Memory Management:
   src/features/map/utils/memoryManagement.ts:

   ```typescript
   export class MemoryManager {
     private cache: Map<string, any> = new Map();
     private maxCacheSize: number;
     private maxCacheAge: number;

     constructor(maxCacheSize = 100, maxCacheAge = 5 * 60 * 1000) {
       this.maxCacheSize = maxCacheSize;
       this.maxCacheAge = maxCacheAge;
     }

     set(key: string, value: any) {
       if (this.cache.size >= this.maxCacheSize) {
         this.cleanup();
       }

       this.cache.set(key, {
         value,
         timestamp: Date.now(),
       });
     }

     get(key: string) {
       const item = this.cache.get(key);
       if (!item) return null;

       if (Date.now() - item.timestamp > this.maxCacheAge) {
         this.cache.delete(key);
         return null;
       }

       return item.value;
     }

     private cleanup() {
       const now = Date.now();
       for (const [key, item] of this.cache.entries()) {
         if (now - item.timestamp > this.maxCacheAge) {
           this.cache.delete(key);
         }
       }

       // If still too large, remove oldest entries
       if (this.cache.size >= this.maxCacheSize) {
         const sortedEntries = Array.from(this.cache.entries()).sort(
           (a, b) => a[1].timestamp - b[1].timestamp
         );

         const entriesToRemove = sortedEntries.slice(
           0,
           Math.floor(this.maxCacheSize * 0.2)
         );

         for (const [key] of entriesToRemove) {
           this.cache.delete(key);
         }
       }
     }

     clear() {
       this.cache.clear();
     }
   }

   export const memoryManager = new MemoryManager();
   ```

8. Worker Management:
   src/features/map/utils/workerManager.ts:

   ```typescript
   export class WorkerManager {
     private workers: Worker[] = [];
     private taskQueue: Array<{
       task: any;
       resolve: (value: any) => void;
       reject: (reason: any) => void;
     }> = [];

     constructor(
       workerScript: string,
       numWorkers = navigator.hardwareConcurrency || 4
     ) {
       for (let i = 0; i < numWorkers; i++) {
         const worker = new Worker(workerScript);
         worker.onmessage = this.handleWorkerMessage.bind(this, i);
         this.workers.push(worker);
       }
     }

     async processTask(task: any) {
       return new Promise((resolve, reject) => {
         this.taskQueue.push({ task, resolve, reject });
         this.processQueue();
       });
     }

     private processQueue() {
       for (let i = 0; i < this.workers.length; i++) {
         if (this.taskQueue.length === 0) break;

         const task = this.taskQueue.shift();
         if (task) {
           this.workers[i].postMessage(task.task);
         }
       }
     }

     private handleWorkerMessage(workerId: number, event: MessageEvent) {
       // Process result and start next task if available
       this.processQueue();
     }

     terminate() {
       this.workers.forEach((worker) => worker.terminate());
       this.workers = [];
       this.taskQueue = [];
     }
   }
   ```

## Map Implementation Complete

The map visualization implementation now includes:

1. Core Components:

   - Base map with customizable tile layers
   - Area and track overlays
   - Clustering and heat map visualizations

2. Performance Optimizations:

   - Viewport management and URL state persistence
   - Geometry simplification based on zoom levels
   - Feature clustering for large datasets
   - Memory management and caching
   - Web Worker support for heavy computations

3. Monitoring and Debugging:
   - Performance metrics tracking
   - Memory usage monitoring
   - Debug utilities

To use the map component:

```typescript
// In your page or component:
import Map from "@/features/map/Map";
import { useSettingsStore } from "@/stores/settingsStore";

const MapPage: React.FC = () => {
  const { mapStyle } = useSettingsStore();

  return (
    <div className="h-screen">
      <Map
        initialCenter={[0, 0]}
        initialZoom={2}
        mapStyle={mapStyle}
        showControls={true}
        enableClustering={true}
      />
    </div>
  );
};

// Example of custom track visualization:
const CustomTrackView: React.FC = () => {
  const { id } = useParams();
  const { data: track } = useQuery(["track", id], () => fetchTrack(id));

  return (
    <div className="grid grid-cols-1 lg:grid-cols-2 gap-4 p-4">
      <div className="h-[600px]">
        <Map
          initialBounds={track?.bounds}
          showControls={false}
          className="rounded-lg shadow-lg"
        >
          {track && (
            <TrackOverlay
              points={track.points}
              showElevation={true}
              showMarkers={true}
              interactive={true}
            />
          )}
        </Map>
      </div>
      <div className="space-y-4">
        <TrackStats track={track} />
        <ElevationProfile points={track?.points} />
      </div>
    </div>
  );
};

// Example of custom area analysis view:
const AreaAnalysis: React.FC = () => {
  const { bounds, setBounds } = useMapState();
  const { data: coverage } = useQuery(["coverage", bounds], () =>
    fetchCoverageInBounds(bounds)
  );

  return (
    <div className="h-screen">
      <Map onBoundsChange={setBounds} enableHeatmap={true} className="h-full">
        <AreaAnalysisOverlay
          coverage={coverage}
          showStatistics={true}
          interactive={true}
          onAreaClick={(area) => {
            // Handle area click
          }}
        />
      </Map>
    </div>
  );
};
```

Customization Examples:

```typescript
// Custom map styles:
const mapStyles = {
  standard: {
    url: "https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",
    attribution: "© OpenStreetMap contributors",
    maxZoom: 19,
  },
  dark: {
    url: "https://{s}.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}.png",
    attribution: "© CARTO",
    maxZoom: 19,
  },
  satellite: {
    url: "https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}",
    attribution: "© Esri",
    maxZoom: 18,
  },
  terrain: {
    url: "https://{s}.tile.opentopomap.org/{z}/{x}/{y}.png",
    attribution: "© OpenTopoMap",
    maxZoom: 17,
  },
};

// Custom track styling:
const trackStyles = {
  default: {
    color: "#2563eb",
    weight: 3,
    opacity: 0.8,
  },
  selected: {
    color: "#1e40af",
    weight: 4,
    opacity: 1,
  },
  highlighted: {
    color: "#3b82f6",
    weight: 4,
    opacity: 1,
    dashArray: "10, 10",
  },
};

// Custom marker icons:
const createMarkerIcon = (type: "start" | "end" | "waypoint") => {
  const colors = {
    start: "#22c55e",
    end: "#ef4444",
    waypoint: "#f59e0b",
  };

  return new L.Icon({
    iconUrl: `/markers/${type}.svg`,
    iconSize: [32, 32],
    iconAnchor: [16, 32],
    popupAnchor: [0, -32],
    className: `marker-${type}`,
  });
};

// Example of custom controls:
const CustomMapControls: React.FC = () => {
  const map = useMap();
  const { showHeatmap, setShowHeatmap } = useMapState();

  return (
    <div className="absolute bottom-4 left-4 bg-white rounded-lg shadow-lg p-2 z-[1000]">
      <div className="space-y-2">
        <button
          onClick={() => map.fitWorld()}
          className="p-2 hover:bg-gray-100 rounded-lg w-full text-left"
        >
          Reset View
        </button>

        <div className="border-t border-gray-200 pt-2">
          <label className="flex items-center space-x-2">
            <input
              type="checkbox"
              checked={showHeatmap}
              onChange={(e) => setShowHeatmap(e.target.checked)}
              className="rounded border-gray-300 text-blue-600"
            />
            <span className="text-sm">Heat Map</span>
          </label>
        </div>

        <button
          onClick={() => map.locate({ setView: true })}
          className="p-2 hover:bg-gray-100 rounded-lg w-full text-left"
        >
          Find My Location
        </button>
      </div>
    </div>
  );
};

// Integration with external services:
const WeatherOverlay: React.FC = () => {
  const map = useMap();
  const bounds = map.getBounds();

  const { data: weather } = useQuery(
    ["weather", bounds.toString()],
    () => fetchWeatherData(bounds),
    { refetchInterval: 5 * 60 * 1000 } // Refetch every 5 minutes
  );

  if (!weather) return null;

  return (
    <>
      {weather.map((point) => (
        <WeatherMarker
          key={point.id}
          position={[point.lat, point.lng]}
          data={point}
        />
      ))}
    </>
  );
};
```

## Map Implementation - Advanced Features

1. Advanced Event Handling:

````typescript
// src/features/map/hooks/useMapEvents.ts
interface MapEventHandlers {
  onClick?: (e: L.LeafletMouseEvent) => void;
  onDragEnd?: (e: L.DragEndEvent) => void;
  onZoomEnd?: (e: L.ZoomAnimEvent) => void;
  onMoveEnd?: (e: L.LeafletEvent) => void;
  onLayerAdd?: (e: L.LayerEvent) => void;
  onLocationFound?: (e: L.LocationEvent) => void;
  onLocationError?: (e: L.ErrorEvent) => void;
}

export const useMapEvents = (map: L.Map | null, handlers: MapEventHandlers) => {
  useEffect(() => {
    if (!map) return;

    // Add event listeners
    if (handlers.onClick) map.on('click', handlers.onClick);
    if (handlers.onDragEnd) map.on('dragend', handlers.onDragEnd);
    if (handlers.onZoomEnd) map.on('zoomend', handlers.onZoomEnd);
    if (handlers.onMoveEnd) map.on('moveend', handlers.onMoveEnd);
    if (handlers.onLayerAdd) map.on('layeradd', handlers.onLayerAdd);
    if (handlers.onLocationFound) map.on('locationfound', handlers.onLocationFound);
    if (handlers.onLocationError) map.on('locationerror', handlers.onLocationError);

    // Cleanup
    return () => {
      if (handlers.onClick) map.off('click', handlers.onClick);
      if (handlers.onDragEnd) map.off('dragend', handlers.onDragEnd);
      if (handlers.onZoomEnd) map.off('zoomend', handlers.onZoomEnd);
      if (handlers.onMoveEnd) map.off('moveend', handlers.onMoveEnd);
      if (handlers.onLayerAdd) map.off('layeradd', handlers.onLayerAdd);
      if (handlers.onLocationFound) map.off('locationfound', handlers.onLocationFound);
      if (handlers.onLocationError) map.off('locationerror', handlers.onLocationError);
    };
  }, [map, handlers]);
};

// Example usage in Map component:
const Map: React.FC<MapProps> = ({ onViewportChange, onFeatureClick }) => {
  const mapRef = useRef<L.Map | null>(null);

  useMapEvents(mapRef.current, {
    onMoveEnd: () => {
      if (mapRef.current && onViewportChange) {
        const center = mapRef.current.getCenter();
        const zoom = mapRef.current.getZoom();
        onViewportChange({ center, zoom });
      }
    },
    onClick: (e) => {
      // Handle map clicks
      const point = [e.latlng.lat, e.latlng.lng];
      const features = findFeaturesAtPoint(point);
      if (features.length && onFeatureClick) {
        onFeatureClick(features[0]);
      }
    },
  });

  // ... rest of the component
};

2. Interactive Features:
```typescript
// src/features/map/components/InteractiveTrack.tsx
interface InteractiveTrackProps {
  points: TrackPoint[];
  selected: boolean;
  onHover?: (point: TrackPoint | null) => void;
  onClick?: (point: TrackPoint) => void;
}

const InteractiveTrack: React.FC<InteractiveTrackProps> = ({
  points,
  selected,
  onHover,
  onClick,
}) => {
  const [hoveredPoint, setHoveredPoint] = useState<TrackPoint | null>(null);

  const handleMouseOver = (point: TrackPoint) => {
    setHoveredPoint(point);
    onHover?.(point);
  };

  const handleMouseOut = () => {
    setHoveredPoint(null);
    onHover?.(null);
  };

  return (
    <>
      <Polyline
        positions={points.map((p) => [p.latitude, p.longitude])}
        {...getTrackStyle(selected, !!hoveredPoint)}
      />
      {points.map((point, index) => (
        <CircleMarker
          key={index}
          center={[point.latitude, point.longitude]}
          radius={hoveredPoint === point ? 6 : 4}
          {...getPointStyle(point === hoveredPoint)}
          eventHandlers={{
            mouseover: () => handleMouseOver(point),
            mouseout: handleMouseOut,
            click: () => onClick?.(point),
          }}
        >
          <Tooltip>
            <div className="space-y-1">
              <div>Elevation: {point.elevation}m</div>
              <div>Time: {formatTime(point.time)}</div>
            </div>
          </Tooltip>
        </CircleMarker>
      ))}
    </>
  );
};

3. Custom Draw Controls:
```typescript
// src/features/map/components/DrawControls.tsx
interface DrawControlsProps {
  onAreaDrawn: (area: GeoJSON.Polygon) => void;
  onLineDrawn: (line: GeoJSON.LineString) => void;
}

const DrawControls: React.FC<DrawControlsProps> = ({ onAreaDrawn, onLineDrawn }) => {
  const map = useMap();
  const drawRef = useRef<L.Control.Draw | null>(null);

  useEffect(() => {
    if (!map) return;

    // Initialize draw controls
    const drawControl = new L.Control.Draw({
      draw: {
        rectangle: true,
        polygon: true,
        circle: false,
        circlemarker: false,
        marker: false,
        polyline: true,
      },
    });

    map.addControl(drawControl);
    drawRef.current = drawControl;

    // Handle draw events
    map.on(L.Draw.Event.CREATED, (e: any) => {
      const layer = e.layer;
      const geoJSON = layer.toGeoJSON();

      if (geoJSON.geometry.type === 'Polygon') {
        onAreaDrawn(geoJSON.geometry);
      } else if (geoJSON.geometry.type === 'LineString') {
        onLineDrawn(geoJSON.geometry);
      }
    });

    return () => {
      if (drawRef.current) {
        map.removeControl(drawRef.current);
      }
    };
  }, [map, onAreaDrawn, onLineDrawn]);

  return null;
};
````
