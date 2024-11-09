Frontend Development

## Part 1: Project Setup and Core Structure",

1. Project Structure Setup:

   ```bash
   cd frontend

   # Initialize additional folders
   mkdir -p src/{components,features,hooks,services,utils,types,styles,layouts,stores}

   # Create core type definitions
   touch src/types/index.ts
   ```

2. Configure TypeScript and Path Aliases:
   tsconfig.json:

   ```json
   {
     \"compilerOptions\": {
       \"target\": \"ES2020\",
       \"useDefineForClassFields\": true,
       \"lib\": [\"ES2020\", \"DOM\", \"DOM.Iterable\"],
       \"module\": \"ESNext\",
       \"skipLibCheck\": true,
       \"moduleResolution\": \"bundler\",
       \"allowImportingTsExtensions\": true,
       \"resolveJsonModule\": true,
       \"isolatedModules\": true,
       \"noEmit\": true,
       \"jsx\": \"react-jsx\",
       \"strict\": true,
       \"noUnusedLocals\": true,
       \"noUnusedParameters\": true,
       \"noFallthroughCasesInSwitch\": true,
       \"baseUrl\": \".\",
       \"paths\": {
         \"@/*\": [\"src/*\"],
         \"@components/*\": [\"src/components/*\"],
         \"@features/*\": [\"src/features/*\"],
         \"@hooks/*\": [\"src/hooks/*\"],
         \"@services/*\": [\"src/services/*\"],
         \"@utils/*\": [\"src/utils/*\"],
         \"@types/*\": [\"src/types/*\"],
         \"@stores/*\": [\"src/stores/*\"]
       }
     },
     \"include\": [\"src\"],
     \"references\": [{ \"path\": \"./tsconfig.node.json\" }]
   }
   ```

3. Setup API Client:
   src/services/api.ts:

   ```typescript
   import axios from "axios";
   import { getStoredToken } from "@utils/auth";

   const API_URL = import.meta.env.VITE_API_URL || "http://localhost:8000";

   const api = axios.create({
     baseURL: `${API_URL}/api/v1`,
     headers: {
       "Content-Type": "application/json",
     },
   });

   api.interceptors.request.use((config) => {
     const token = getStoredToken();
     if (token) {
       config.headers.Authorization = `Bearer ${token}`;
     }
     return config;
   });

   api.interceptors.response.use(
     (response) => response,
     async (error) => {
       if (error.response?.status === 401) {
         // Handle token expiration
         window.location.href = "/login";
       }
       return Promise.reject(error);
     }
   );

   export default api;
   ```

4. Type Definitions:
   src/types/index.ts:

   ```typescript
   export interface User {
     id: string;
     email: string;
     fullName: string;
   }

   export interface Track {
     id: string;
     filename: string;
     status: "processing" | "completed" | "error";
     totalDistance: number;
     totalElevationGain: number;
     startTime: string;
     endTime: string;
     createdAt: string;
   }

   export interface CoverageStats {
     areaKm2: number;
     percentage: number;
   }

   export interface GeoJSON {
     type: string;
     coordinates: number[][][];
   }

   export interface CoverageDetails extends CoverageStats {
     geojson: GeoJSON;
   }

   export interface ApiError {
     message: string;
     status: number;
   }
   ```

5. Authentication Store:
   src/stores/authStore.ts:

   ```typescript
   import { create } from "zustand";
   import { persist } from "zustand/middleware";
   import { User } from "@/types";
   import api from "@/services/api";

   interface AuthState {
     user: User | null;
     token: string | null;
     isAuthenticated: boolean;
     login: (email: string, password: string) => Promise<void>;
     logout: () => void;
     signup: (
       email: string,
       password: string,
       fullName: string
     ) => Promise<void>;
   }

   export const useAuthStore = create<AuthState>(
     persist(
       (set) => ({
         user: null,
         token: null,
         isAuthenticated: false,

         login: async (email, password) => {
           const response = await api.post("/auth/login", {
             username: email,
             password,
           });
           const { access_token } = response.data;
           const userResponse = await api.get("/users/me", {
             headers: { Authorization: `Bearer ${access_token}` },
           });

           set({
             token: access_token,
             user: userResponse.data,
             isAuthenticated: true,
           });
         },

         logout: () => {
           set({
             user: null,
             token: null,
             isAuthenticated: false,
           });
         },

         signup: async (email, password, fullName) => {
           await api.post("/auth/signup", {
             email,
             password,
             full_name: fullName,
           });
           await useAuthStore.getState().login(email, password);
         },
       }),
       {
         name: "auth-storage",
       }
     )
   );
   ```

6. Create Layout Components:
   src/layouts/MainLayout.tsx:

   ```typescript
   import React from 'react';
   import { Outlet, Navigate } from 'react-router-dom';
   import { useAuthStore } from '@/stores/authStore';
   import Navbar from '@/components/Navbar';
   import Sidebar from '@/components/Sidebar';

   const MainLayout: React.FC = () => {
     const { isAuthenticated } = useAuthStore();

     if (!isAuthenticated) {
       return <Navigate to=\"/login\" replace />;
     }

     return (
       <div className=\"min-h-screen bg-gray-100\">
         <Navbar />
         <div className=\"flex\">
           <Sidebar />
           <main className=\"flex-1 p-6\">
             <Outlet />
           </main>
         </div>
       </div>
     );
   };

   export default MainLayout;
   ```

## Part 2: Authentication and Routing",

1. Router Setup:
   src/App.tsx:

   ```typescript
   import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
   import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
   import MainLayout from '@/layouts/MainLayout';
   import Login from '@/features/auth/Login';
   import Signup from '@/features/auth/Signup';
   import Dashboard from '@/features/dashboard/Dashboard';
   import Tracks from '@/features/tracks/Tracks';
   import Upload from '@/features/upload/Upload';
   import Map from '@/features/map/Map';

   const queryClient = new QueryClient();

   const App = () => {
     return (
       <QueryClientProvider client={queryClient}>
         <BrowserRouter>
           <Routes>
             <Route path=\"/login\" element={<Login />} />
             <Route path=\"/signup\" element={<Signup />} />
             <Route element={<MainLayout />}>
               <Route path=\"/\" element={<Navigate to=\"/dashboard\" replace />} />
               <Route path=\"/dashboard\" element={<Dashboard />} />
               <Route path=\"/tracks\" element={<Tracks />} />
               <Route path=\"/upload\" element={<Upload />} />
               <Route path=\"/map\" element={<Map />} />
             </Route>
           </Routes>
         </BrowserRouter>
       </QueryClientProvider>
     );
   };

   export default App;
   ```

2. Login Component:
   src/features/auth/Login.tsx:

   ```typescript
   import React from 'react';
   import { useNavigate, Link } from 'react-router-dom';
   import { useForm } from 'react-hook-form';
   import { useAuthStore } from '@/stores/authStore';

   interface LoginForm {
     email: string;
     password: string;
   }

   const Login: React.FC = () => {
     const navigate = useNavigate();
     const { login } = useAuthStore();
     const { register, handleSubmit, formState: { errors } } = useForm<LoginForm>();

     const onSubmit = async (data: LoginForm) => {
       try {
         await login(data.email, data.password);
         navigate('/dashboard');
       } catch (error) {
         console.error('Login failed:', error);
       }
     };

     return (
       <div className=\"min-h-screen flex items-center justify-center bg-gray-100\">
         <div className=\"max-w-md w-full p-6 bg-white rounded-lg shadow-md\">
           <h2 className=\"text-2xl font-bold text-center mb-6\">Login</h2>
           <form onSubmit={handleSubmit(onSubmit)} className=\"space-y-4\">
             <div>
               <label className=\"block text-sm font-medium text-gray-700\">Email</label>
               <input
                 type=\"email\"
                 {...register('email', { required: true, pattern: /^\\S+@\\S+\\.\\S+$/ })}
                 className=\"mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500\"
               />
               {errors.email && (
                 <span className=\"text-red-500 text-sm\">Valid email is required</span>
               )}
             </div>
             <div>
               <label className=\"block text-sm font-medium text-gray-700\">Password</label>
               <input
                 type=\"password\"
                 {...register('password', { required: true, minLength: 6 })}
                 className=\"mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500\"
               />
               {errors.password && (
                 <span className=\"text-red-500 text-sm\">Password must be at least 6 characters</span>
               )}
             </div>
             <button
               type=\"submit\"
               className=\"w-full py-2 px-4 bg-blue-600 hover:bg-blue-700 text-white rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2\"
             >
               Login
             </button>
           </form>
           <p className=\"mt-4 text-center text-sm text-gray-600\">
             Don't have an account?{' '}
             <Link to=\"/signup\" className=\"text-blue-600 hover:text-blue-500\">
               Sign up
             </Link>
           </p>
         </div>
       </div>
     );
   };

   export default Login;
   ```

3. Protected Route Hook:
   src/hooks/useAuth.ts:

   ```typescript
   import { useEffect } from "react";
   import { useNavigate } from "react-router-dom";
   import { useAuthStore } from "@/stores/authStore";

   export const useAuth = () => {
     const { isAuthenticated, user } = useAuthStore();
     const navigate = useNavigate();

     useEffect(() => {
       if (!isAuthenticated) {
         navigate("/login", { replace: true });
       }
     }, [isAuthenticated, navigate]);

     return { isAuthenticated, user };
   };
   ```

4. Navigation Components:
   src/components/Navbar.tsx:

   ```typescript
   import React from 'react';
   import { Link } from 'react-router-dom';
   import { useAuthStore } from '@/stores/authStore';

   const Navbar: React.FC = () => {
     const { user, logout } = useAuthStore();

     return (
       <nav className=\"bg-white shadow-sm\">
         <div className=\"max-w-7xl mx-auto px-4 sm:px-6 lg:px-8\">
           <div className=\"flex justify-between h-16\">
             <div className=\"flex\">
               <div className=\"flex-shrink-0 flex items-center\">
                 <Link to=\"/\" className=\"text-xl font-bold text-blue-600\">
                   GPX Tracker
                 </Link>
               </div>
               <div className=\"hidden sm:ml-6 sm:flex sm:space-x-8\">
                 <Link
                   to=\"/dashboard\"
                   className=\"border-transparent text-gray-500 hover:border-gray-300 hover:text-gray-700 inline-flex items-center px-1 pt-1 border-b-2 text-sm font-medium\"
                 >
                   Dashboard
                 </Link>
                 <Link
                   to=\"/tracks\"
                   className=\"border-transparent text-gray-500 hover:border-gray-300 hover:text-gray-700 inline-flex items-center px-1 pt-1 border-b-2 text-sm font-medium\"
                 >
                   Tracks
                 </Link>
                 <Link
                   to=\"/map\"
                   className=\"border-transparent text-gray-500 hover:border-gray-300 hover:text-gray-700 inline-flex items-center px-1 pt-1 border-b-2 text-sm font-medium\"
                 >
                   Map
                 </Link>
               </div>
             </div>
             <div className=\"flex items-center\">
               <span className=\"text-gray-700 mr-4\">{user?.fullName}</span>
               <button
                 onClick={logout}
                 className=\"bg-blue-600 hover:bg-blue-700 text-white px-4 py-2 rounded-md text-sm font-medium\"
               >
                 Logout
               </button>
             </div>
           </div>
         </div>
       </nav>
     );
   };

   export default Navbar;
   ```

5. Dashboard Component:
   src/features/dashboard/Dashboard.tsx:

   ```typescript
   import React from 'react';
   import { useQuery } from '@tanstack/react-query';
   import api from '@/services/api';
   import { CoverageStats, Track } from '@/types';
   import StatsCard from './components/StatsCard';
   import RecentTracks from './components/RecentTracks';
   import CoverageChart from './components/CoverageChart';

   const Dashboard: React.FC = () => {
     const { data: stats } = useQuery<CoverageStats>(
       ['coverage-stats'],
       async () => {
         const response = await api.get('/areas/coverage');
         return response.data;
       }
     );

     const { data: recentTracks } = useQuery<Track[]>(
       ['recent-tracks'],
       async () => {
         const response = await api.get('/tracks', {
           params: { limit: 5 }
         });
         return response.data;
       }
     );

     return (
       <div className=\"space-y-6\">
         <h1 className=\"text-2xl font-bold\">Dashboard</h1>

         <div className=\"grid grid-cols-1 md:grid-cols-3 gap-6\">
           <StatsCard
             title=\"Total Area Covered\"
             value={`${stats?.areaKm2.toFixed(2)} km²`}
             description=\"Total area covered by your tracks\"
           />
           <StatsCard
             title=\"Earth Coverage\"
             value={`${stats?.percentage.toFixed(6)}%`}
             description=\"Percentage of Earth's surface covered\"
           />
           <StatsCard
             title=\"Total Tracks\"
             value={recentTracks?.length || 0}
             description=\"Number of uploaded tracks\"
           />
         </div>

         <div className=\"grid grid-cols-1 lg:grid-cols-2 gap-6\">
           <CoverageChart />
           <RecentTracks tracks={recentTracks || []} />
         </div>
       </div>
     );
   };

   export default Dashboard;
   ```

6. Dashboard Components:
   src/features/dashboard/components/StatsCard.tsx:

   ```typescript
   import React from 'react';

   interface StatsCardProps {
     title: string;
     value: string | number;
     description: string;
   }

   const StatsCard: React.FC<StatsCardProps> = ({ title, value, description }) => {
     return (
       <div className=\"bg-white rounded-lg shadow p-6\">
         <h3 className=\"text-lg font-medium text-gray-900\">{title}</h3>
         <p className=\"mt-2 text-3xl font-semibold text-blue-600\">{value}</p>
         <p className=\"mt-2 text-sm text-gray-500\">{description}</p>
       </div>
     );
   };

   export default StatsCard;
   ```

   src/features/dashboard/components/CoverageChart.tsx:

   ```typescript
   import React from 'react';
   import { useQuery } from '@tanstack/react-query';
   import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer } from 'recharts';
   import api from '@/services/api';

   interface CoverageData {
     date: string;
     area: number;
   }

   const CoverageChart: React.FC = () => {
     const { data } = useQuery<CoverageData[]>(
       ['coverage-history'],
       async () => {
         const response = await api.get('/areas/coverage/history');
         return response.data;
       }
     );

     return (
       <div className=\"bg-white rounded-lg shadow p-6\">
         <h3 className=\"text-lg font-medium text-gray-900 mb-4\">Coverage Growth</h3>
         <div className=\"h-80\">
           <ResponsiveContainer width=\"100%\" height=\"100%\">
             <LineChart data={data || []}>
               <CartesianGrid strokeDasharray=\"3 3\" />
               <XAxis dataKey=\"date\" />
               <YAxis />
               <Tooltip />
               <Line
                 type=\"monotone\"
                 dataKey=\"area\"
                 stroke=\"#2563eb\"
                 strokeWidth={2}
               />
             </LineChart>
           </ResponsiveContainer>
         </div>
       </div>
     );
   };

   export default CoverageChart;
   ```

   src/features/dashboard/components/RecentTracks.tsx:

   ```typescript
   import React from 'react';
   import { Link } from 'react-router-dom';
   import { Track } from '@/types';
   import { formatDate, formatDistance } from '@/utils/formatting';

   interface RecentTracksProps {
     tracks: Track[];
   }

   const RecentTracks: React.FC<RecentTracksProps> = ({ tracks }) => {
     return (
       <div className=\"bg-white rounded-lg shadow\">
         <div className=\"p-6\">
           <h3 className=\"text-lg font-medium text-gray-900\">Recent Tracks</h3>
         </div>
         <div className=\"border-t border-gray-200\">
           <ul className=\"divide-y divide-gray-200\">
             {tracks.map((track) => (
               <li key={track.id} className=\"p-6\">
                 <div className=\"flex items-center justify-between\">
                   <div>
                     <h4 className=\"text-sm font-medium text-gray-900\">
                       {track.filename}
                     </h4>
                     <p className=\"mt-1 text-sm text-gray-500\">
                       {formatDistance(track.totalDistance)} •{' '}
                       {formatDate(track.createdAt)}
                     </p>
                   </div>
                   <Link
                     to={`/tracks/${track.id}`}
                     className=\"text-blue-600 hover:text-blue-500 text-sm font-medium\"
                   >
                     View Details
                   </Link>
                 </div>
               </li>
             ))}
           </ul>
         </div>
       </div>
     );
   };

   export default RecentTracks;
   ```

## Part 3: Map and File Upload

1. Map Component:
   src/features/map/Map.tsx:

   ```typescript
   import React, { useEffect } from 'react';
   import { useQuery } from '@tanstack/react-query';
   import { MapContainer, TileLayer, GeoJSON } from 'react-leaflet';
   import 'leaflet/dist/leaflet.css';
   import api from '@/services/api';
   import { CoverageDetails } from '@/types';
   import MapControls from './components/MapControls';
   import MapLegend from './components/MapLegend';

   const Map: React.FC = () => {
     const { data: coverage } = useQuery<CoverageDetails>(
       ['coverage-details'],
       async () => {
         const response = await api.get('/areas/coverage/details');
         return response.data;
       }
     );

     return (
       <div className=\"h-[calc(100vh-4rem)]\">
         <MapContainer
           center={[0, 0]}
           zoom={2}
           className=\"h-full w-full\"
           zoomControl={false}
         >
           <TileLayer
             url=\"https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png\"
             attribution='&copy; <a href=\"https://www.openstreetmap.org/copyright\">OpenStreetMap</a> contributors'
           />
           {coverage && (
             <GeoJSON
               data={coverage.geojson}
               style={() => ({
                 color: '#2563eb',
                 weight: 2,
                 opacity: 0.6,
                 fillColor: '#3b82f6',
                 fillOpacity: 0.3,
               })}
             />
           )}
           <MapControls />
           <MapLegend coverage={coverage} />
         </MapContainer>
       </div>
     );
   };

   export default Map;
   ```

   src/features/map/components/MapControls.tsx:

   ```typescript
   import React from 'react';
   import { useMap } from 'react-leaflet';

   const MapControls: React.FC = () => {
     const map = useMap();

     const handleZoomIn = () => map.zoomIn();
     const handleZoomOut = () => map.zoomOut();
     const handleCenter = () => map.setView([0, 0], 2);

     return (
       <div className=\"absolute top-4 right-4 bg-white rounded-lg shadow-lg p-2 z-[1000]\">
         <button
           onClick={handleZoomIn}
           className=\"p-2 hover:bg-gray-100 rounded-lg\"
           title=\"Zoom In\"
         >
           <svg className=\"w-6 h-6\" fill=\"none\" stroke=\"currentColor\" viewBox=\"0 0 24 24\">
             <path strokeLinecap=\"round\" strokeLinejoin=\"round\" strokeWidth={2} d=\"M12 6v6m0 0v6m0-6h6m-6 0H6\" />
           </svg>
         </button>
         <button
           onClick={handleZoomOut}
           className=\"p-2 hover:bg-gray-100 rounded-lg\"
           title=\"Zoom Out\"
         >
           <svg className=\"w-6 h-6\" fill=\"none\" stroke=\"currentColor\" viewBox=\"0 0 24 24\">
             <path strokeLinecap=\"round\" strokeLinejoin=\"round\" strokeWidth={2} d=\"M20 12H4\" />
           </svg>
         </button>
         <button
           onClick={handleCenter}
           className=\"p-2 hover:bg-gray-100 rounded-lg\"
           title=\"Center Map\"
         >
           <svg className=\"w-6 h-6\" fill=\"none\" stroke=\"currentColor\" viewBox=\"0 0 24 24\">
             <path strokeLinecap=\"round\" strokeLinejoin=\"round\" strokeWidth={2} d=\"M3.055 11H5a2 2 0 012 2v1a2 2 0 002 2 2 2 0 012 2v2.945M8 3.935V5.5A2.5 2.5 0 0010.5 8h.5a2 2 0 012 2 2 2 0 104 0 2 2 0 012-2h1.064M15 20.488V18a2 2 0 012-2h3.064M21 12a9 9 0 11-18 0 9 9 0 0118 0z\" />
           </svg>
         </button>
       </div>
     );
   };

   export default MapControls;
   ```

2. File Upload Component:
   src/features/upload/Upload.tsx:

   ```typescript
   import React, { useCallback, useState } from 'react';
   import { useNavigate } from 'react-router-dom';
   import { useDropzone } from 'react-dropzone';
   import { useMutation } from '@tanstack/react-query';
   import api from '@/services/api';
   import UploadProgress from './components/UploadProgress';

   const Upload: React.FC = () => {
     const navigate = useNavigate();
     const [uploadProgress, setUploadProgress] = useState(0);

     const uploadMutation = useMutation(
       async (file: File) => {
         const formData = new FormData();
         formData.append('file', file);

         const response = await api.post('/tracks/upload', formData, {
           headers: { 'Content-Type': 'multipart/form-data' },
           onUploadProgress: (progressEvent) => {
             if (progressEvent.total) {
               const progress = (progressEvent.loaded / progressEvent.total) * 100;
               setUploadProgress(progress);
             }
           },
         });

         return response.data;
       },
       {
         onSuccess: () => {
           navigate('/tracks');
         },
       }
     );

     const onDrop = useCallback(
       (acceptedFiles: File[]) => {
         const file = acceptedFiles[0];
         if (file) {
           uploadMutation.mutate(file);
         }
       },
       [uploadMutation]
     );

     const { getRootProps, getInputProps, isDragActive } = useDropzone({
       onDrop,
       accept: { 'application/gpx+xml': ['.gpx'] },
       maxFiles: 1,
     });

     return (
       <div className=\"max-w-2xl mx-auto py-8\">
         <h1 className=\"text-2xl font-bold mb-6\">Upload GPX Track</h1>

         <div
           {...getRootProps()}
           className={`border-2 border-dashed rounded-lg p-12 text-center ${
             isDragActive ? 'border-blue-500 bg-blue-50' : 'border-gray-300'
           }`}
         >
           <input {...getInputProps()} />
           <svg
             className=\"mx-auto h-12 w-12 text-gray-400\"
             fill=\"none\"
             viewBox=\"0 0 24 24\"
             stroke=\"currentColor\"
           >
           <path
               strokeLinecap=\"round\"
               strokeLinejoin=\"round\"
               strokeWidth={2}
               d=\"M7 16a4 4 0 01-.88-7.903A5 5 0 1115.9 6L16 6a5 5 0 011 9.9M15 13l-3-3m0 0l-3 3m3-3v12\"
             />
           </svg>
           <p className=\"mt-4 text-lg text-gray-600\">
             {isDragActive
               ? 'Drop your GPX file here...'
               : 'Drag & drop your GPX file here, or click to select'}
           </p>
           <p className=\"mt-2 text-sm text-gray-500\">
             GPX files only (max size: 10MB)
           </p>
         </div>

         {uploadMutation.isLoading && (
           <UploadProgress progress={uploadProgress} />
         )}

         {uploadMutation.isError && (
           <div className=\"mt-4 p-4 bg-red-50 rounded-md\">
             <p className=\"text-red-700\">
               Error uploading file: {uploadMutation.error?.message}
             </p>
           </div>
         )}
       </div>
     );
   };

   export default Upload;
   ```

   src/features/upload/components/UploadProgress.tsx:

   ```typescript
   import React from 'react';

   interface UploadProgressProps {
     progress: number;
   }

   const UploadProgress: React.FC<UploadProgressProps> = ({ progress }) => {
     return (
       <div className=\"mt-6\">
         <div className=\"relative pt-1\">
           <div className=\"flex mb-2 items-center justify-between\">
             <div>
               <span className=\"text-xs font-semibold inline-block py-1 px-2 uppercase rounded-full text-blue-600 bg-blue-200\">
                 Uploading
               </span>
             </div>
             <div className=\"text-right\">
               <span className=\"text-xs font-semibold inline-block text-blue-600\">
                 {Math.round(progress)}%
               </span>
             </div>
           </div>
           <div className=\"overflow-hidden h-2 mb-4 text-xs flex rounded bg-blue-200\">
             <div
               style={{ width: `${progress}%` }}
               className=\"shadow-none flex flex-col text-center whitespace-nowrap text-white justify-center bg-blue-500 transition-all duration-300\"
             />
           </div>
         </div>
       </div>
     );
   };

   export default UploadProgress;
   ```

3. Track List Component:
   src/features/tracks/Tracks.tsx:

   ```typescript
   import React from 'react';
   import { useQuery } from '@tanstack/react-query';
   import { Link } from 'react-router-dom';
   import api from '@/services/api';
   import { Track } from '@/types';
   import TrackCard from './components/TrackCard';
   import TrackFilters from './components/TrackFilters';

   const Tracks: React.FC = () => {
     const { data: tracks, isLoading } = useQuery<Track[]>(
       ['tracks'],
       async () => {
         const response = await api.get('/tracks');
         return response.data;
       }
     );

     return (
       <div>
         <div className=\"flex justify-between items-center mb-6\">
           <h1 className=\"text-2xl font-bold\">My Tracks</h1>
           <Link
             to=\"/upload\"
             className=\"bg-blue-600 hover:bg-blue-700 text-white px-4 py-2 rounded-md\"
           >
             Upload Track
           </Link>
         </div>

         <TrackFilters />

         {isLoading ? (
           <div className=\"text-center py-12\">
             <div className=\"animate-spin rounded-full h-12 w-12 border-b-2 border-blue-600 mx-auto\" />
           </div>
         ) : (
           <div className=\"grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6\">
             {tracks?.map((track) => (
               <TrackCard key={track.id} track={track} />
             ))}
           </div>
         )}
       </div>
     );
   };

   export default Tracks;
   ```

   src/features/tracks/components/TrackCard.tsx:

   ```typescript
   import React from 'react';
   import { Link } from 'react-router-dom';
   import { Track } from '@/types';
   import { formatDate, formatDistance } from '@/utils/formatting';

   interface TrackCardProps {
     track: Track;
   }

   const TrackCard: React.FC<TrackCardProps> = ({ track }) => {
     const getStatusColor = (status: string) => {
       switch (status) {
         case 'completed':
           return 'bg-green-100 text-green-800';
         case 'processing':
           return 'bg-yellow-100 text-yellow-800';
         case 'error':
           return 'bg-red-100 text-red-800';
         default:
           return 'bg-gray-100 text-gray-800';
       }
     };

     return (
       <div className=\"bg-white rounded-lg shadow-md overflow-hidden\">
         <div className=\"p-6\">
           <div className=\"flex items-center justify-between mb-4\">
             <h3 className=\"text-lg font-medium text-gray-900 truncate\">
               {track.filename}
             </h3>
             <span
               className={`px-2 py-1 text-xs font-medium rounded-full ${getStatusColor(
                 track.status
               )}`}
             >
               {track.status}
             </span>
           </div>

           <div className=\"space-y-2\">
             <p className=\"text-sm text-gray-500\">
               Distance: {formatDistance(track.totalDistance)}
             </p>
             <p className=\"text-sm text-gray-500\">
               Elevation Gain: {track.totalElevationGain}m
             </p>
             <p className=\"text-sm text-gray-500\">
               Uploaded: {formatDate(track.createdAt)}
             </p>
           </div>

           <div className=\"mt-4\">
             <Link
               to={`/tracks/${track.id}`}
               className=\"text-blue-600 hover:text-blue-500 text-sm font-medium\"
             >
               View Details →
             </Link>
           </div>
         </div>
       </div>
     );
   };

   export default TrackCard;
   ```

## Part 4: Track Details and Utilities

1. Track Details Component:
   src/features/tracks/TrackDetails.tsx:

   ```typescript
   import React from 'react';
   import { useParams } from 'react-router-dom';
   import { useQuery } from '@tanstack/react-query';
   import { MapContainer, TileLayer, Polyline } from 'react-leaflet';
   import api from '@/services/api';
   import { Track } from '@/types';
   import TrackStats from './components/TrackStats';
   import TrackElevationChart from './components/TrackElevationChart';

   const TrackDetails: React.FC = () => {
     const { id } = useParams<{ id: string }>();

     const { data: track, isLoading } = useQuery<Track>(
       ['track', id],
       async () => {
         const response = await api.get(`/tracks/${id}`);
         return response.data;
       }
     );

     const { data: trackPoints } = useQuery(
       ['track-points', id],
       async () => {
         const response = await api.get(`/tracks/${id}/points`);
         return response.data;
       },
       {
         enabled: !!track,
       }
     );

     if (isLoading) {
       return (
         <div className=\"flex justify-center items-center h-64\">
           <div className=\"animate-spin rounded-full h-12 w-12 border-b-2 border-blue-600\" />
         </div>
       );
     }

     if (!track) {
       return (
         <div className=\"text-center py-12\">
           <h2 className=\"text-2xl font-bold text-gray-900\">Track not found</h2>
         </div>
       );
     }

     const positions = trackPoints?.map((point: any) => [
       point.latitude,
       point.longitude,
     ]);

     return (
       <div className=\"space-y-6\">
         <div className=\"flex justify-between items-center\">
           <h1 className=\"text-2xl font-bold\">{track.filename}</h1>
           <span
             className={`px-3 py-1 text-sm font-medium rounded-full ${
               track.status === 'completed'
                 ? 'bg-green-100 text-green-800'
                 : 'bg-yellow-100 text-yellow-800'
             }`}
           >
             {track.status}
           </span>
         </div>

         <div className=\"grid grid-cols-1 lg:grid-cols-2 gap-6\">
           <div className=\"bg-white rounded-lg shadow-md overflow-hidden\">
             <div className=\"h-96\">
               <MapContainer
                 bounds={positions}
                 className=\"h-full w-full\"
                 zoomControl={false}
               >
                 <TileLayer
                   url=\"https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png\"
                   attribution='&copy; <a href=\"https://www.openstreetmap.org/copyright\">OpenStreetMap</a> contributors'
                 />
                 {positions && (
                   <Polyline
                     positions={positions}
                     color=\"#2563eb\"
                     weight={3}
                   />
                 )}
               </MapContainer>
             </div>
           </div>

           <TrackStats track={track} />
         </div>

         <TrackElevationChart points={trackPoints || []} />
       </div>
     );
   };

   export default TrackDetails;
   ```

2. Track Statistics Component:
   src/features/tracks/components/TrackStats.tsx:

   ```typescript
   import React from 'react';
   import { Track } from '@/types';
   import { formatDate, formatDistance, formatDuration } from '@/utils/formatting';

   interface TrackStatsProps {
     track: Track;
   }

   const TrackStats: React.FC<TrackStatsProps> = ({ track }) => {
     return (
       <div className=\"bg-white rounded-lg shadow-md p-6\">
         <h2 className=\"text-lg font-medium mb-4\">Track Statistics</h2>
         <div className=\"grid grid-cols-2 gap-4\">
           <div>
             <p className=\"text-sm text-gray-500\">Distance</p>
             <p className=\"text-lg font-medium\">
               {formatDistance(track.totalDistance)}
             </p>
           </div>
           <div>
             <p className=\"text-sm text-gray-500\">Elevation Gain</p>
             <p className=\"text-lg font-medium\">
               {track.totalElevationGain}m
             </p>
           </div>
           <div>
             <p className=\"text-sm text-gray-500\">Duration</p>
             <p className=\"text-lg font-medium\">
               {formatDuration(track.startTime, track.endTime)}
             </p>
           </div>
           <div>
             <p className=\"text-sm text-gray-500\">Date</p>
             <p className=\"text-lg font-medium\">
               {formatDate(track.startTime)}
             </p>
           </div>
         </div>
       </div>
     );
   };

   export default TrackStats;
   ```

3. Utility Functions:
   src/utils/formatting.ts:

   ```typescript
   import { format, formatDistanceToNow, differenceInSeconds } from "date-fns";

   export const formatDistance = (meters: number): string => {
     if (meters < 1000) {
       return `${meters.toFixed(0)}m`;
     }
     return `${(meters / 1000).toFixed(2)}km`;
   };

   export const formatDate = (date: string): string => {
     return format(new Date(date), "PPP");
   };

   export const formatDuration = (start: string, end: string): string => {
     const seconds = differenceInSeconds(new Date(end), new Date(start));
     const hours = Math.floor(seconds / 3600);
     const minutes = Math.floor((seconds % 3600) / 60);

     if (hours > 0) {
       return `${hours}h ${minutes}m`;
     }
     return `${minutes}m`;
   };

   export const formatRelativeTime = (date: string): string => {
     return formatDistanceToNow(new Date(date), { addSuffix: true });
   };

   export const formatElevation = (meters: number): string => {
     return `${meters.toFixed(0)}m`;
   };
   ```

## Part 5: Elevation Chart and Global State

1. Elevation Chart Component:
   src/features/tracks/components/TrackElevationChart.tsx:

```typescript
 import React from 'react';
 import {
   AreaChart,
   Area,
   XAxis,
   YAxis,
   CartesianGrid,
   Tooltip,
   ResponsiveContainer
 } from 'recharts';
 import { formatDistance, formatElevation } from '@/utils/formatting';

 interface TrackPoint {
   distance: number;
   elevation: number;
 }

 interface TrackElevationChartProps {
   points: TrackPoint[];
 }

 const TrackElevationChart: React.FC<TrackElevationChartProps> = ({ points }) => {
   const data = points.map((point, index) => ({
     distance: point.distance,
     elevation: point.elevation,
   }));

   return (
     <div className=\"bg-white rounded-lg shadow-md p-6\">
       <h2 className=\"text-lg font-medium mb-4\">Elevation Profile</h2>
       <div className=\"h-64\">
         <ResponsiveContainer width=\"100%\" height=\"100%\">
           <AreaChart data={data}>
             <CartesianGrid strokeDasharray=\"3 3\" />
             <XAxis
               dataKey=\"distance\"
               tickFormatter={(value) => formatDistance(value)}
               label={{ value: 'Distance', position: 'insideBottom', offset: -10 }}
             />
             <YAxis
               tickFormatter={(value) => formatElevation(value)}
               label={{ value: 'Elevation', angle: -90, position: 'insideLeft' }}
             />
             <Tooltip
               formatter={(value: number, name: string) =>
                 name === 'elevation'
                   ? formatElevation(value)
                   : formatDistance(value)
               }
               labelFormatter={(value) => `Distance: ${formatDistance(value)}`}
             />
             <Area
               type=\"monotone\"
               dataKey=\"elevation\"
               stroke=\"#2563eb\"
               fill=\"#93c5fd\"
               strokeWidth={2}
             />
           </AreaChart>
         </ResponsiveContainer>
       </div>
     </div>
   );
 };

 export default TrackElevationChart;
```

2. Global State Management:
   src/stores/settingsStore.ts:

   ```typescript
   import { create } from "zustand";
   import { persist } from "zustand/middleware";

   interface SettingsState {
     theme: "light" | "dark";
     mapStyle: "standard" | "satellite" | "terrain";
     unitSystem: "metric" | "imperial";
     setTheme: (theme: "light" | "dark") => void;
     setMapStyle: (style: "standard" | "satellite" | "terrain") => void;
     setUnitSystem: (system: "metric" | "imperial") => void;
   }

   export const useSettingsStore = create<SettingsState>(
     persist(
       (set) => ({
         theme: "light",
         mapStyle: "standard",
         unitSystem: "metric",
         setTheme: (theme) => set({ theme }),
         setMapStyle: (mapStyle) => set({ mapStyle }),
         setUnitSystem: (unitSystem) => set({ unitSystem }),
       }),
       {
         name: "settings-storage",
       }
     )
   );
   ```

3. Settings Component:
   src/features/settings/Settings.tsx:

   ```typescript
   import React from 'react';
   import { useSettingsStore } from '@/stores/settingsStore';

   const Settings: React.FC = () => {
     const { theme, mapStyle, unitSystem, setTheme, setMapStyle, setUnitSystem } =
       useSettingsStore();

     return (
       <div className=\"max-w-2xl mx-auto py-8\">
         <h1 className=\"text-2xl font-bold mb-6\">Settings</h1>

         <div className=\"bg-white rounded-lg shadow-md p-6 space-y-6\">
           <div>
             <h2 className=\"text-lg font-medium mb-4\">Appearance</h2>
             <div className=\"space-y-4\">
               <div>
                 <label className=\"block text-sm font-medium text-gray-700 mb-2\">
                   Theme
                 </label>
                 <select
                   value={theme}
                   onChange={(e) => setTheme(e.target.value as 'light' | 'dark')}
                   className=\"mt-1 block w-full pl-3 pr-10 py-2 text-base border-gray-300 focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm rounded-md\"
                 >
                   <option value=\"light\">Light</option>
                   <option value=\"dark\">Dark</option>
                 </select>
               </div>

               <div>
                 <label className=\"block text-sm font-medium text-gray-700 mb-2\">
                   Map Style
                 </label>
                 <select
                   value={mapStyle}
                   onChange={(e) =>
                     setMapStyle(
                       e.target.value as 'standard' | 'satellite' | 'terrain'
                     )
                   }
                   className=\"mt-1 block w-full pl-3 pr-10 py-2 text-base border-gray-300 focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm rounded-md\"
                 >
                   <option value=\"standard\">Standard</option>
                   <option value=\"satellite\">Satellite</option>
                   <option value=\"terrain\">Terrain</option>
                 </select>
               </div>
             </div>
           </div>

           <div>
             <h2 className=\"text-lg font-medium mb-4\">Preferences</h2>
             <div>
               <label className=\"block text-sm font-medium text-gray-700 mb-2\">
                 Unit System
               </label>
               <select
                 value={unitSystem}
                 onChange={(e) =>
                   setUnitSystem(e.target.value as 'metric' | 'imperial')
                 }
                 className=\"mt-1 block w-full pl-3 pr-10 py-2 text-base border-gray-300 focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm rounded-md\"
               >
                 <option value=\"metric\">Metric (km, m)</option>
                 <option value=\"imperial\">Imperial (mi, ft)</option>
               </select>
             </div>
           </div>
         </div>
       </div>
     );
   };

   export default Settings;
   ```

## Part 6: Theme Support and Responsive Design

1. Theme Provider:
   src/providers/ThemeProvider.tsx:

   ```typescript
   import React, { createContext, useContext, useEffect } from "react";
   import { useSettingsStore } from "@/stores/settingsStore";

   interface ThemeContextType {
     theme: "light" | "dark";
     toggleTheme: () => void;
   }

   const ThemeContext = createContext<ThemeContextType | undefined>(undefined);

   export const ThemeProvider: React.FC<{ children: React.ReactNode }> = ({
     children,
   }) => {
     const { theme, setTheme } = useSettingsStore();

     useEffect(() => {
       document.documentElement.classList.remove("light", "dark");
       document.documentElement.classList.add(theme);
     }, [theme]);

     const toggleTheme = () => {
       setTheme(theme === "light" ? "dark" : "light");
     };

     return (
       <ThemeContext.Provider value={{ theme, toggleTheme }}>
         {children}
       </ThemeContext.Provider>
     );
   };

   export const useTheme = () => {
     const context = useContext(ThemeContext);
     if (context === undefined) {
       throw new Error("useTheme must be used within a ThemeProvider");
     }
     return context;
   };
   ```

2. Responsive Utilities:
   src/utils/responsive.ts:

   ```typescript
   import { useState, useEffect } from "react";

   export const breakpoints = {
     sm: 640,
     md: 768,
     lg: 1024,
     xl: 1280,
     "2xl": 1536,
   };

   export const useBreakpoint = () => {
     const [breakpoint, setBreakpoint] = useState("");

     useEffect(() => {
       const handleResize = () => {
         const width = window.innerWidth;
         if (width < breakpoints.sm) setBreakpoint("xs");
         else if (width < breakpoints.md) setBreakpoint("sm");
         else if (width < breakpoints.lg) setBreakpoint("md");
         else if (width < breakpoints.xl) setBreakpoint("lg");
         else if (width < breakpoints["2xl"]) setBreakpoint("xl");
         else setBreakpoint("2xl");
       };

       handleResize();
       window.addEventListener("resize", handleResize);
       return () => window.removeEventListener("resize", handleResize);
     }, []);

     return breakpoint;
   };
   ```

3. Dark Mode Styles:
   src/styles/darkMode.css:

   ```css
   /* Dark mode overrides */
   .dark {
     --bg-primary: #1a1a1a;
     --bg-secondary: #2d2d2d;
     --text-primary: #ffffff;
     --text-secondary: #a3a3a3;
     --border-color: #404040;
   }

   .dark body {
     background-color: var(--bg-primary);
     color: var(--text-primary);
   }

   .dark .bg-white {
     background-color: var(--bg-secondary);
   }

   .dark .text-gray-900 {
     color: var(--text-primary);
   }

   .dark .text-gray-500 {
     color: var(--text-secondary);
   }

   .dark .border-gray-200 {
     border-color: var(--border-color);
   }

   .dark .shadow-md {
     box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.2);
   }
   ```

4. Responsive Layout Component:
   src/components/ResponsiveContainer.tsx:

   ```typescript
   import React from "react";
   import { useBreakpoint } from "@/utils/responsive";

   interface ResponsiveContainerProps {
     children: React.ReactNode;
     className?: string;
   }

   const ResponsiveContainer: React.FC<ResponsiveContainerProps> = ({
     children,
     className = "",
   }) => {
     const breakpoint = useBreakpoint();

     const getMaxWidth = () => {
       switch (breakpoint) {
         case "xs":
         case "sm":
           return "max-w-full";
         case "md":
           return "max-w-3xl";
         case "lg":
           return "max-w-5xl";
         case "xl":
         case "2xl":
           return "max-w-7xl";
         default:
           return "max-w-full";
       }
     };

     return (
       <div
         className={`mx-auto px-4 sm:px-6 lg:px-8 ${getMaxWidth()} ${className}`}
       >
         {children}
       </div>
     );
   };

   export default ResponsiveContainer;
   ```

5. Update App Component:
   src/App.tsx:

   ```typescript
   import { ThemeProvider } from "@/providers/ThemeProvider";
   import Router from "@/Router";
   import "@/styles/darkMode.css";

   const App = () => {
     return (
       <ThemeProvider>
         <Router />
       </ThemeProvider>
     );
   };

   export default App;
   ```

## Part 7: Error Handling and Performance

1. Error Boundary Component:
   src/components/ErrorBoundary.tsx:

   ```typescript
   import React, { Component, ErrorInfo } from 'react';

   interface Props {
     children: React.ReactNode;
   }

   interface State {
     hasError: boolean;
     error: Error | null;
   }

   class ErrorBoundary extends Component<Props, State> {
     public state: State = {
       hasError: false,
       error: null,
     };

     public static getDerivedStateFromError(error: Error): State {
       return { hasError: true, error };
     }

     public componentDidCatch(error: Error, errorInfo: ErrorInfo) {
       console.error('Uncaught error:', error, errorInfo);
     }

     public render() {
       if (this.state.hasError) {
         return (
           <div className=\"min-h-screen flex items-center justify-center bg-gray-100\">
             <div className=\"max-w-md w-full p-6 bg-white rounded-lg shadow-md\">
               <h2 className=\"text-2xl font-bold text-red-600 mb-4\">
                 Oops! Something went wrong
               </h2>
               <p className=\"text-gray-600 mb-4\">
                 {this.state.error?.message || 'An unexpected error occurred'}
               </p>
               <button
                 onClick={() => window.location.reload()}
                 className=\"w-full py-2 px-4 bg-blue-600 text-white rounded-md hover:bg-blue-700 transition-colors\"
               >
                 Refresh Page
               </button>
             </div>
           </div>
         );
       }

       return this.props.children;
     }
   }

   export default ErrorBoundary;
   ```

2. Loading States Component:
   src/components/LoadingStates.tsx:

   ```typescript
   import React from 'react';

   interface LoadingSpinnerProps {
     size?: 'small' | 'medium' | 'large';
     className?: string;
   }

   export const LoadingSpinner: React.FC<LoadingSpinnerProps> = ({
     size = 'medium',
     className = '',
   }) => {
     const sizeClasses = {
       small: 'h-4 w-4',
       medium: 'h-8 w-8',
       large: 'h-12 w-12',
     };

     return (
       <div className={`flex justify-center items-center ${className}`}>
         <div
           className={`${sizeClasses[size]} border-4 border-blue-200 border-t-blue-600 rounded-full animate-spin`}
         />
       </div>
     );
   };

   interface LoadingSkeletonProps {
     lines?: number;
     className?: string;
   }

   export const LoadingSkeleton: React.FC<LoadingSkeletonProps> = ({
     lines = 3,
     className = '',
   }) => {
     return (
       <div className={`space-y-4 ${className}`}>
         {Array.from({ length: lines }).map((_, index) => (
           <div
             key={index}
             className=\"h-4 bg-gray-200 rounded animate-pulse dark:bg-gray-700\"
             style={{ width: `${Math.random() * 50 + 50}%` }}
           />
         ))}
       </div>
     );
   };
   ```

3. Performance Optimizations:
   src/utils/performance.ts:

   ```typescript
   import { useCallback, useEffect, useState } from "react";

   // Debounce hook for performance optimization
   export const useDebounce = <T>(value: T, delay: number): T => {
     const [debouncedValue, setDebouncedValue] = useState<T>(value);

     useEffect(() => {
       const handler = setTimeout(() => {
         setDebouncedValue(value);
       }, delay);

       return () => {
         clearTimeout(handler);
       };
     }, [value, delay]);

     return debouncedValue;
   };

   // Intersection Observer hook for lazy loading
   export const useIntersectionObserver = (
     elementRef: React.RefObject<Element>,
     options: IntersectionObserverInit = {}
   ): boolean => {
     const [isVisible, setIsVisible] = useState(false);

     const callback = useCallback((entries: IntersectionObserverEntry[]) => {
       const [entry] = entries;
       setIsVisible(entry.isIntersecting);
     }, []);

     useEffect(() => {
       const observer = new IntersectionObserver(callback, options);
       const element = elementRef.current;

       if (element) {
         observer.observe(element);
       }

       return () => {
         if (element) {
           observer.unobserve(element);
         }
       };
     }, [elementRef, options, callback]);

     return isVisible;
   };

   // Memoized selector for complex state derivations
   export const createSelector = <T, R>(
     selector: (state: T) => R
   ): ((state: T) => R) => {
     let lastState: T | undefined;
     let lastResult: R | undefined;

     return (state: T): R => {
       if (lastState === state) {
         return lastResult as R;
       }

       lastState = state;
       lastResult = selector(state);
       return lastResult;
     };
   };
   ```

4. API Request Caching and Prefetching:
   src/utils/queryClient.ts:

   ```typescript
   import { QueryClient } from "@tanstack/react-query";

   export const queryClient = new QueryClient({
     defaultOptions: {
       queries: {
         staleTime: 5 * 60 * 1000, // 5 minutes
         cacheTime: 30 * 60 * 1000, // 30 minutes
         retry: 3,
         retryDelay: (attemptIndex) =>
           Math.min(1000 * 2 ** attemptIndex, 30000),
         refetchOnWindowFocus: false,
       },
     },
   });

   export const prefetchTrackData = async (trackId: string) => {
     await Promise.all([
       queryClient.prefetchQuery(["track", trackId], async () => {
         const response = await api.get(`/tracks/${trackId}`);
         return response.data;
       }),
       queryClient.prefetchQuery(["track-points", trackId], async () => {
         const response = await api.get(`/tracks/${trackId}/points`);
         return response.data;
       }),
     ]);
   };
   ```

5. Global Error Handler:
   src/utils/errorHandler.ts:

   ```typescript
   import { AxiosError } from "axios";
   import { toast } from "react-toastify";

   export const handleApiError = (error: unknown) => {
     if (error instanceof AxiosError) {
       const message = error.response?.data?.message || error.message;
       toast.error(`Error: ${message}`);
     } else if (error instanceof Error) {
       toast.error(`Error: ${error.message}`);
     } else {
       toast.error("An unexpected error occurred");
     }
   };

   export const setupGlobalErrorHandling = () => {
     window.addEventListener("unhandledrejection", (event) => {
       handleApiError(event.reason);
     });

     window.addEventListener("error", (event) => {
       handleApiError(event.error);
     });
   };
   ```

## Part 8: Service Worker and Final Optimizations

1. Service Worker Setup:
   src/service-worker.ts:

   ```typescript
   /// <reference lib=\"webworker\" />

   import { clientsClaim } from 'workbox-core';
   import { precacheAndRoute } from 'workbox-precaching';
   import { registerRoute } from 'workbox-routing';
   import { CacheFirst, StaleWhileRevalidate } from 'workbox-strategies';
   import { ExpirationPlugin } from 'workbox-expiration';

   declare const self: ServiceWorkerGlobalScope;

   clientsClaim();

   // Precache all assets generated by your build process
   precacheAndRoute(self.__WB_MANIFEST);

   // Cache map tiles
   registerRoute(
     ({ url }) => url.href.includes('tile.openstreetmap.org'),
     new CacheFirst({
       cacheName: 'map-tiles',
       plugins: [
         new ExpirationPlugin({
           maxEntries: 500,
           maxAgeSeconds: 30 * 24 * 60 * 60, // 30 days
         }),
       ],
     })
   );

   // Cache API responses
   registerRoute(
     ({ url }) => url.pathname.startsWith('/api'),
     new StaleWhileRevalidate({
       cacheName: 'api-cache',
       plugins: [
         new ExpirationPlugin({
           maxEntries: 100,
           maxAgeSeconds: 24 * 60 * 60, // 24 hours
         }),
       ],
     })
   );

   // Handle offline fallback
   const FALLBACK_HTML = '/offline.html';

   self.addEventListener('install', (event) => {
     event.waitUntil(
       caches.open('offline-fallback').then((cache) => {
         return cache.add(FALLBACK_HTML);
       })
     );
   });

   self.addEventListener('fetch', (event) => {
     if (event.request.mode === 'navigate') {
       event.respondWith(
         fetch(event.request).catch(() => {
           return caches.match(FALLBACK_HTML);
         })
       );
     }
   });
   ```

2. Performance Monitoring:
   src/utils/performance-monitoring.ts:

   ```typescript
   export const measurePerformance = (metricName: string) => {
     const marks: { [key: string]: number } = {};

     return {
       start: () => {
         marks[metricName] = performance.now();
       },
       end: () => {
         if (marks[metricName]) {
           const duration = performance.now() - marks[metricName];
           console.log(`${metricName}: ${duration.toFixed(2)}ms`);
           delete marks[metricName];
           return duration;
         }
         return 0;
       },
     };
   };

   export const trackPageLoad = () => {
     if ("performance" in window) {
       window.addEventListener("load", () => {
         const pageLoadTime =
           performance.timing.loadEventEnd - performance.timing.navigationStart;
         console.log(`Page Load Time: ${pageLoadTime}ms`);

         const paintMetrics = performance.getEntriesByType("paint");
         paintMetrics.forEach((metric) => {
           console.log(`${metric.name}: ${metric.startTime}ms`);
         });
       });
     }
   };
   ```

3. Application Update Handler:
   src/utils/app-update.ts:

   ```typescript
   import { toast } from "react-toastify";

   export const setupAppUpdateHandler = () => {
     if ("serviceWorker" in navigator) {
       let refreshing = false;

       navigator.serviceWorker.addEventListener("controllerchange", () => {
         if (!refreshing) {
           refreshing = true;
           window.location.reload();
         }
       });

       navigator.serviceWorker.ready.then((registration) => {
         registration.addEventListener("updatefound", () => {
           const newWorker = registration.installing;
           if (newWorker) {
             newWorker.addEventListener("statechange", () => {
               if (
                 newWorker.state === "installed" &&
                 navigator.serviceWorker.controller
               ) {
                 toast.info(
                   "A new version is available! Click here to update.",
                   {
                     onClick: () => {
                       newWorker.postMessage({ type: "SKIP_WAITING" });
                     },
                   }
                 );
               }
             });
           }
         });
       });
     }
   };
   ```

4. Initialize Service Worker and Performance Monitoring:
   src/main.tsx:

   ```typescript
   import React from 'react';
   import ReactDOM from 'react-dom/client';
   import { QueryClientProvider } from '@tanstack/react-query';
   import { ToastContainer } from 'react-toastify';
   import App from './App';
   import { queryClient } from './utils/queryClient';
   import { setupGlobalErrorHandling } from './utils/errorHandler';
   import { trackPageLoad } from './utils/performance-monitoring';
   import { setupAppUpdateHandler } from './utils/app-update';
   import 'react-toastify/dist/ReactToastify.css';
   import './index.css';

   // Setup error handling and performance monitoring
   setupGlobalErrorHandling();
   trackPageLoad();
   setupAppUpdateHandler();

   // Register service worker
   if ('serviceWorker' in navigator) {
     window.addEventListener('load', () => {
       navigator.serviceWorker
         .register('/service-worker.js')
         .then((registration) => {
           console.log('SW registered:', registration);
         })
         .catch((error) => {
           console.log('SW registration failed:', error);
         });
     });
   }

   ReactDOM.createRoot(document.getElementById('root')!).render(
     <React.StrictMode>
       <QueryClientProvider client={queryClient}>
         <App />
         <ToastContainer position=\"bottom-right\" />
       </QueryClientProvider>
     </React.StrictMode>
   );
   ```

## Frontend Implementation Complete",

The frontend implementation is now complete with the following features:

1. Core Features:

   - Interactive map visualization with Leaflet.js
   - GPX file upload and processing
   - Track statistics and elevation profiles
   - User dashboard with coverage statistics

2. Technical Implementation:

   - React with TypeScript
   - State management with Zustand
   - API integration with React Query
   - Responsive design with Tailwind CSS
   - Theme support (light/dark modes)

3. Performance Optimizations:

   - Service Worker for offline support
   - Performance monitoring
   - Debounced and memoized operations
   - Efficient data caching

4. User Experience:
   - Loading states and skeletons
   - Error handling and boundaries
   - Toast notifications
   - Responsive layouts

To start the development server:

```bash
# Install dependencies
npm install

# Start development server
npm run dev
```

Next steps could include:

1. Adding end-to-end tests with Cypress or Playwright
2. Implementing more advanced map features (route planning, elevation profiles)
3. Adding social features (sharing tracks, following other users)
4. Implementing data export/import functionality
5. Adding advanced analytics and statistics
6. Implementing real-time tracking features
7. Adding mobile app support with React Native
8. Enhancing offline capabilities with more sophisticated caching strategies",
