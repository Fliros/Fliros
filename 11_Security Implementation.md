# Security Implementation

## Part 1: Authentication and Authorization

1. Enhanced Authentication System:
   backend/app/core/security.py:

   ```python
   from datetime import datetime, timedelta
   from typing import Any, Optional
   from passlib.context import CryptContext
   from jose import JWTError, jwt
   import secrets
   from app.core.config import settings

   pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

   class SecurityManager:
       def __init__(self):
           self.secret_key = settings.SECRET_KEY
           self.algorithm = "HS256"
           self.access_token_expire_minutes = settings.ACCESS_TOKEN_EXPIRE_MINUTES
           self.refresh_token_expire_days = 30

       def create_password_hash(self, password: str) -> str:
           """Create a secure password hash"""
           return pwd_context.hash(password)

       def verify_password(self, plain_password: str, hashed_password: str) -> bool:
           """Verify password against hash"""
           return pwd_context.verify(plain_password, hashed_password)

       def create_access_token(self, data: dict, expires_delta: Optional[timedelta] = None) -> str:
           """Create JWT access token"""
           to_encode = data.copy()
           if expires_delta:
               expire = datetime.utcnow() + expires_delta
           else:
               expire = datetime.utcnow() + timedelta(minutes=self.access_token_expire_minutes)

           to_encode.update({"exp": expire, "type": "access"})
           encoded_jwt = jwt.encode(to_encode, self.secret_key, algorithm=self.algorithm)
           return encoded_jwt

       def create_refresh_token(self, data: dict) -> str:
           """Create refresh token"""
           to_encode = data.copy()
           expire = datetime.utcnow() + timedelta(days=self.refresh_token_expire_days)
           to_encode.update({"exp": expire, "type": "refresh"})
           return jwt.encode(to_encode, self.secret_key, algorithm=self.algorithm)

       def verify_token(self, token: str) -> dict:
           """Verify JWT token"""
           try:
               payload = jwt.decode(token, self.secret_key, algorithms=[self.algorithm])
               return payload
           except JWTError:
               return None

       def generate_reset_token(self) -> str:
           """Generate secure reset token"""
           return secrets.token_urlsafe(32)

   security = SecurityManager()
   ```

2. Authentication Middleware:
   backend/app/middleware/auth.py:

   ```python
   from fastapi import Request, HTTPException
   from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
   from app.core.security import security
   from app.models.user import User
   from app.services.rate_limiter import RateLimiter
   from typing import Optional
   import time

   class JWTAuthMiddleware(HTTPBearer):
       def __init__(self, auto_error: bool = True):
           super().__init__(auto_error=auto_error)
           self.rate_limiter = RateLimiter()

       async def __call__(self, request: Request) -> Optional[HTTPAuthorizationCredentials]:
           # Rate limiting check
           if not self.rate_limiter.is_allowed(request.client.host):
               raise HTTPException(
                   status_code=429,
                   detail="Too many requests"
               )

           credentials: HTTPAuthorizationCredentials = await super().__call__(request)

           if not credentials:
               raise HTTPException(
                   status_code=403,
                   detail="Could not validate credentials"
               )

           try:
               payload = security.verify_token(credentials.credentials)
               if payload is None:
                   raise HTTPException(
                       status_code=403,
                       detail="Could not validate credentials"
                   )

               # Check token type
               if payload.get("type") != "access":
                   raise HTTPException(
                       status_code=403,
                       detail="Invalid token type"
                   )

               # Add user info to request state
               request.state.user_id = payload.get("sub")
               request.state.token_payload = payload

               return credentials

           except Exception as e:
               raise HTTPException(
                   status_code=403,
                   detail=str(e)
               )

   # Rate limiting implementation
   class RateLimiter:
       def __init__(self, max_requests: int = 100, window_seconds: int = 60):
           self.max_requests = max_requests
           self.window_seconds = window_seconds
           self.requests = {}

       def is_allowed(self, client_ip: str) -> bool:
           current_time = time.time()
           client_requests = self.requests.get(client_ip, [])

           # Clean old requests
           client_requests = [
               req_time for req_time in client_requests
               if current_time - req_time < self.window_seconds
           ]

           if len(client_requests) >= self.max_requests:
               return False

           client_requests.append(current_time)
           self.requests[client_ip] = client_requests
           return True
   ```

3. User Session Management:
   backend/app/services/session.py:

   ```python
   from typing import Optional
   from datetime import datetime
   import redis
   from app.core.config import settings

   class SessionManager:
       def __init__(self):
           self.redis_client = redis.Redis.from_url(settings.REDIS_URL)
           self.session_expire_time = 3600 * 24  # 24 hours

       async def create_session(self, user_id: str, device_info: dict) -> str:
           """Create new user session"""
           session_id = secrets.token_urlsafe(32)
           session_data = {
               "user_id": user_id,
               "created_at": datetime.utcnow().isoformat(),
               "device_info": device_info,
               "last_activity": datetime.utcnow().isoformat()
           }

           await self.redis_client.setex(
               f"session:{session_id}",
               self.session_expire_time,
               json.dumps(session_data)
           )

           return session_id

       async def get_session(self, session_id: str) -> Optional[dict]:
           """Get session data"""
           session_data = await self.redis_client.get(f"session:{session_id}")
           if session_data:
               return json.loads(session_data)
           return None

       async def update_session_activity(self, session_id: str) -> None:
           """Update last activity timestamp"""
           session_data = await self.get_session(session_id)
           if session_data:
               session_data["last_activity"] = datetime.utcnow().isoformat()
               await self.redis_client.setex(
                   f"session:{session_id}",
                   self.session_expire_time,
                   json.dumps(session_data)
               )

       async def invalidate_session(self, session_id: str) -> None:
           """Invalidate user session"""
           await self.redis_client.delete(f"session:{session_id}")

       async def invalidate_all_user_sessions(self, user_id: str) -> None:
           """Invalidate all sessions for a user"""
           pattern = f"session:*"
           all_sessions = await self.redis_client.keys(pattern)

           for session_key in all_sessions:
               session_data = await self.get_session(session_key.decode().split(":")[1])
               if session_data and session_data["user_id"] == user_id:
                   await self.redis_client.delete(session_key)
   ```

## Part 2: Data Encryption

1. Encryption Service:
   backend/app/services/encryption.py:

   ```python
   from cryptography.fernet import Fernet
   from cryptography.hazmat.primitives import hashes
   from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC
   from cryptography.hazmat.primitives.asymmetric import rsa, padding
   from cryptography.hazmat.primitives import serialization
   import base64
   import os
   from typing import Union
   from app.core.config import settings

   class EncryptionService:
       def __init__(self):
           self.master_key = base64.b64decode(settings.ENCRYPTION_MASTER_KEY)
           self._fernet = None
           self.initialize_encryption()

       def initialize_encryption(self) -> None:
           """Initialize encryption with master key"""
           kdf = PBKDF2HMAC(
               algorithm=hashes.SHA256(),
               length=32,
               salt=settings.ENCRYPTION_SALT.encode(),
               iterations=100000,
           )
           key = base64.b64encode(kdf.derive(self.master_key))
           self._fernet = Fernet(key)

       def encrypt_value(self, value: Union[str, bytes]) -> str:
           """Encrypt a single value"""
           if isinstance(value, str):
               value = value.encode()
           encrypted = self._fernet.encrypt(value)
           return base64.b64encode(encrypted).decode()

       def decrypt_value(self, encrypted_value: Union[str, bytes]) -> str:
           """Decrypt a single value"""
           if isinstance(encrypted_value, str):
               encrypted_value = base64.b64decode(encrypted_value)
           decrypted = self._fernet.decrypt(encrypted_value)
           return decrypted.decode()

       def generate_data_key(self) -> tuple[bytes, bytes]:
           """Generate a new data key for envelope encryption"""
           data_key = Fernet.generate_key()
           encrypted_data_key = self._fernet.encrypt(data_key)
           return data_key, encrypted_data_key

       def encrypt_large_data(self, data: bytes) -> tuple[bytes, bytes]:
           """Encrypt large data using envelope encryption"""
           data_key, encrypted_data_key = self.generate_data_key()
           f = Fernet(data_key)
           encrypted_data = f.encrypt(data)
           return encrypted_data, encrypted_data_key

       def decrypt_large_data(self, encrypted_data: bytes, encrypted_data_key: bytes) -> bytes:
           """Decrypt data using envelope encryption"""
           data_key = self._fernet.decrypt(encrypted_data_key)
           f = Fernet(data_key)
           return f.decrypt(encrypted_data)

   encryption_service = EncryptionService()
   ```

2. Database Field Encryption:
   backend/app/models/encrypted_fields.py:

   ```python
   from sqlalchemy.types import TypeDecorator, String
   from app.services.encryption import encryption_service

   class EncryptedString(TypeDecorator):
       impl = String

       def process_bind_param(self, value: str, dialect) -> str:
           if value is not None:
               return encryption_service.encrypt_value(value)
           return None

       def process_result_value(self, value: str, dialect) -> str:
           if value is not None:
               return encryption_service.decrypt_value(value)
           return None

   class EncryptedJSON(TypeDecorator):
       impl = String

       def process_bind_param(self, value: dict, dialect) -> str:
           if value is not None:
               json_str = json.dumps(value)
               return encryption_service.encrypt_value(json_str)
           return None

       def process_result_value(self, value: str, dialect) -> dict:
           if value is not None:
               json_str = encryption_service.decrypt_value(value)
               return json.loads(json_str)
           return None

   # Example usage in models
   class User(Base):
       __tablename__ = "users"

       id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
       email = Column(String, unique=True, index=True)
       hashed_password = Column(String)
       full_name = Column(String)
       sensitive_data = Column(EncryptedJSON)  # Stored encrypted
       phone_number = Column(EncryptedString)  # Stored encrypted
   ```

3. Secure Key Management:
   backend/app/services/key_management.py:

   ```python
   from cryptography.fernet import Fernet
   from typing import Optional
   import boto3
   from botocore.exceptions import ClientError
   import base64

   class KeyManagementService:
       def __init__(self):
           self.kms_client = boto3.client('kms')
           self.key_alias = 'alias/gpx-tracker-key'
           self._data_key_cache = {}

       async def get_data_key(self, key_id: str) -> Optional[bytes]:
           """Get data key from cache or KMS"""
           if key_id in self._data_key_cache:
               return self._data_key_cache[key_id]

           try:
               response = self.kms_client.generate_data_key(
                   KeyId=self.key_alias,
                   KeySpec='AES_256'
               )

               data_key = response['Plaintext']
               encrypted_data_key = response['CiphertextBlob']

               # Cache the data key for reuse
               self._data_key_cache[key_id] = {
                   'data_key': data_key,
                   'encrypted_data_key': encrypted_data_key
               }

               return data_key

           except ClientError as e:
               logger.error(f"Error generating data key: {e}")
               return None

       async def rotate_data_key(self, key_id: str) -> bool:
           """Rotate data key"""
           try:
               # Remove from cache to force new key generation
               self._data_key_cache.pop(key_id, None)
               await self.get_data_key(key_id)
               return True
           except Exception as e:
               logger.error(f"Error rotating data key: {e}")
               return False

       async def encrypt_data_key(self, data_key: bytes) -> bytes:
           """Encrypt data key using KMS"""
           try:
               response = self.kms_client.encrypt(
                   KeyId=self.key_alias,
                   Plaintext=data_key
               )
               return response['CiphertextBlob']
           except ClientError as e:
               logger.error(f"Error encrypting data key: {e}")
               raise

       async def decrypt_data_key(self, encrypted_data_key: bytes) -> bytes:
           """Decrypt data key using KMS"""
           try:
               response = self.kms_client.decrypt(
                   CiphertextBlob=encrypted_data_key,
                   KeyId=self.key_alias
               )
               return response['Plaintext']
           except ClientError as e:
               logger.error(f"Error decrypting data key: {e}")
               raise

   key_management_service = KeyManagementService()
   ```

4. Encryption Middleware:
   backend/app/middleware/encryption.py:

   ```python
   from fastapi import Request, Response
   from typing import Callable
   import json
   from app.services.encryption import encryption_service

   async def encryption_middleware(request: Request, call_next: Callable) -> Response:
       """Middleware to handle encryption/decryption of sensitive data"""

       # Check if request contains sensitive data that needs encryption
       if request.method in ['POST', 'PUT', 'PATCH']:
           try:
               body = await request.json()
               if 'sensitive_data' in body:
                   body['sensitive_data'] = encryption_service.encrypt_value(
                       json.dumps(body['sensitive_data'])
                   )
                   # Override request body with encrypted data
                   setattr(request, '_json', body)
           except ValueError:
               pass

       response = await call_next(request)

       # Check if response contains encrypted data that needs decryption
       if response.headers.get('content-type') == 'application/json':
           try:
               body = json.loads(response.body)
               if 'sensitive_data' in body:
                   body['sensitive_data'] = json.loads(
                       encryption_service.decrypt_value(body['sensitive_data'])
                   )
                   response.body = json.dumps(body).encode()
           except ValueError:
               pass

       return response
   ```

## Part 3: Input Validation and Sanitization

1. Request Validation:
   backend/app/validation/validators.py:

   ```python
   from pydantic import BaseModel, EmailStr, constr, validator
   from typing import Optional, List
   from datetime import datetime
   import re
   from uuid import UUID
   from fastapi import HTTPException
   import bleach
   from email_validator import validate_email, EmailNotValidError

   class ValidatorBase:
       @classmethod
       def sanitize_string(cls, value: str) -> str:
           """Sanitize string input"""
           if not value:
               return value
           # Remove any control characters
           value = "".join(char for char in value if ord(char) >= 32)
           # Use bleach to clean HTML/scripts
           return bleach.clean(value, strip=True)

       @classmethod
       def validate_email(cls, email: str) -> str:
           """Validate email format and domain"""
           try:
               valid = validate_email(email)
               return valid.email
           except EmailNotValidError as e:
               raise ValueError(str(e))

       @classmethod
       def validate_password_strength(cls, password: str) -> str:
           """Validate password meets security requirements"""
           if len(password) < 8:
               raise ValueError("Password must be at least 8 characters long")
           if not re.search(r'[A-Z]', password):
               raise ValueError("Password must contain at least one uppercase letter")
           if not re.search(r'[a-z]', password):
               raise ValueError("Password must contain at least one lowercase letter")
           if not re.search(r'[0-9]', password):
               raise ValueError("Password must contain at least one number")
           if not re.search(r'[!@#$%^&*(),.?":{}|<>]', password):
               raise ValueError("Password must contain at least one special character")
           return password

    # User-related validation models
    class UserCreate(BaseModel):
        email: EmailStr
        password: constr(min_length=8)
        full_name: constr(min_length=1, max_length=100)

        @validator('email')
        def validate_email_format(cls, v):
            return ValidatorBase.validate_email(v)

        @validator('password')
        def validate_password(cls, v):
            return ValidatorBase.validate_password_strength(v)

        @validator('full_name')
        def sanitize_full_name(cls, v):
            return ValidatorBase.sanitize_string(v)

    # GPX track validation models
    class TrackCreate(BaseModel):
        name: Optional[constr(max_length=100)]
        description: Optional[constr(max_length=1000)]
        activity_type: Optional[constr(regex='^[A-Za-z_]+$')]

        @validator('name', 'description')
        def sanitize_text_fields(cls, v):
            if v:
                return ValidatorBase.sanitize_string(v)
            return v

        @validator('activity_type')
        def validate_activity_type(cls, v):
            allowed_types = {'hiking', 'running', 'cycling', 'walking'}
            if v and v not in allowed_types:
                raise ValueError(f'Activity type must be one of: {allowed_types}')
            return v
   ```

2. Input Sanitization Middleware:
   backend/app/middleware/sanitization.py:

   ```python
   from fastapi import Request, Response
   from typing import Callable, Dict, Any
   import json
   from html import escape
   import re

   class InputSanitizer:
       @staticmethod
       def sanitize_value(value: Any) -> Any:
           """Recursively sanitize input values"""
           if isinstance(value, str):
               # Escape HTML entities
               value = escape(value)
               # Remove potentially dangerous patterns
               value = re.sub(r'javascript:', '', value, flags=re.IGNORECASE)
               value = re.sub(r'data:', '', value, flags=re.IGNORECASE)
               # Remove null bytes
               value = value.replace('\\x00', '')
               return value
           elif isinstance(value, dict):
               return {k: InputSanitizer.sanitize_value(v) for k, v in value.items()}
           elif isinstance(value, list):
               return [InputSanitizer.sanitize_value(item) for item in value]
           return value

       @staticmethod
       async def sanitize_request_body(request: Request) -> Dict[str, Any]:
           """Sanitize entire request body"""
           try:
               body = await request.json()
               return InputSanitizer.sanitize_value(body)
           except ValueError:
               return {}

   async def sanitization_middleware(request: Request, call_next: Callable) -> Response:
       """Middleware to sanitize incoming requests"""
       if request.method in ['POST', 'PUT', 'PATCH']:
           try:
               sanitized_body = await InputSanitizer.sanitize_request_body(request)
               setattr(request, '_json', sanitized_body)
           except ValueError:
               pass

       response = await call_next(request)
       return response
   ```

3. GPX File Validation:
   backend/app/validation/gpx_validator.py:

   ```python
   from typing import Tuple, Optional
   import xml.etree.ElementTree as ET
   from datetime import datetime
   import gpxpy
   from fastapi import UploadFile
   import magic

   class GPXValidator:
       MAX_FILE_SIZE = 10 * 1024 * 1024  # 10MB
       ALLOWED_MIME_TYPES = ['application/gpx+xml', 'text/xml']

       @staticmethod
       async def validate_gpx_file(file: UploadFile) -> Tuple[bool, Optional[str]]:
           """Validate GPX file format and content"""
           try:
               # Check file size
               file.file.seek(0, 2)
               size = file.file.tell()
               file.file.seek(0)

               if size > GPXValidator.MAX_FILE_SIZE:
                   return False, "File size exceeds maximum limit"

               # Check file type
               content = await file.read(2048)  # Read first 2KB for MIME detection
               mime_type = magic.from_buffer(content, mime=True)
               if mime_type not in GPXValidator.ALLOWED_MIME_TYPES:
                   return False, "Invalid file type"

               # Reset file pointer
               file.file.seek(0)
               content = await file.read()

               # Parse GPX content
               gpx = gpxpy.parse(content.decode())

               # Validate GPX structure
               if not gpx.tracks:
                   return False, "No tracks found in GPX file"

               # Validate track points
               total_points = sum(len(segment.points)
                   for track in gpx.tracks
                   for segment in track.segments)

               if total_points == 0:
                   return False, "No track points found"

               if total_points > 50000:  # Limit number of points
                   return False, "Too many track points"

               # Validate coordinates
               for track in gpx.tracks:
                   for segment in track.segments:
                       for point in segment.points:
                           if not (-90 <= point.latitude <= 90) or \\
                              not (-180 <= point.longitude <= 180):
                               return False, "Invalid coordinates found"

               return True, None

           except ET.ParseError:
               return False, "Invalid GPX format"
           except Exception as e:
               return False, f"Validation error: {str(e)}"
           finally:
               file.file.seek(0)

       @staticmethod
       def sanitize_gpx_metadata(gpx_content: str) -> str:
           """Sanitize GPX metadata fields"""
           try:
               tree = ET.ElementTree(ET.fromstring(gpx_content))
               root = tree.getroot()

               # Sanitize metadata elements
               metadata = root.find('{*}metadata')
               if metadata is not None:
                   for element in metadata.iter():
                       if element.text:
                           element.text = InputSanitizer.sanitize_value(element.text)

               return ET.tostring(root, encoding='unicode')
           except ET.ParseError:
               raise ValueError("Invalid GPX format")
   ```

## Part 4: Protection Against Common Vulnerabilities

1. Security Middleware Configuration:
   backend/app/middleware/security.py:

   ```python
   from fastapi import FastAPI, Request, Response
   from fastapi.middleware.cors import CORSMiddleware
   from fastapi.middleware.trustedhost import TrustedHostMiddleware
   from starlette.middleware.sessions import SessionMiddleware
   from starlette.datastructures import Headers
   import secrets
   from typing import Callable
   import time
   from app.core.config import settings

   def setup_security_middleware(app: FastAPI) -> None:
       """Configure all security middleware"""

       # CORS middleware
       app.add_middleware(
           CORSMiddleware,
           allow_origins=settings.CORS_ORIGINS,
           allow_credentials=True,
           allow_methods=["GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"],
           allow_headers=["*"],
           expose_headers=["Content-Disposition"],
           max_age=3600,
       )

       # Trusted hosts middleware
       app.add_middleware(
           TrustedHostMiddleware,
           allowed_hosts=settings.ALLOWED_HOSTS
       )

       # Session middleware for CSRF token
       app.add_middleware(
           SessionMiddleware,
           secret_key=settings.SECRET_KEY,
           same_site="lax",  # Prevents CSRF in modern browsers
           https_only=settings.USE_HTTPS
       )

       # Custom security headers middleware
       @app.middleware("http")
       async def security_headers_middleware(request: Request, call_next: Callable) -> Response:
           response = await call_next(request)

           # Security headers
           headers = {
               "X-Content-Type-Options": "nosniff",
               "X-Frame-Options": "DENY",
               "X-XSS-Protection": "1; mode=block",
               "Strict-Transport-Security": "max-age=31536000; includeSubDomains",
               "Content-Security-Policy": (
                   "default-src 'self'; "
                   "script-src 'self' 'unsafe-inline' 'unsafe-eval'; "
                   "style-src 'self' 'unsafe-inline'; "
                   "img-src 'self' data: https:; "
                   "font-src 'self'; "
                   "connect-src 'self' https:;"
               ),
               "Referrer-Policy": "strict-origin-when-cross-origin",
               "Permissions-Policy": "geolocation=(), microphone=(), camera=()"
           }

           for header_name, header_value in headers.items():
               response.headers[header_name] = header_value

           return response

   class CSRFProtection:
       def __init__(self):
           self.token_length = 32

       def generate_csrf_token(self) -> str:
           """Generate a secure CSRF token"""
           return secrets.token_urlsafe(self.token_length)

       async def validate_csrf_token(self, request: Request) -> bool:
           """Validate CSRF token"""
           if request.method in ["GET", "HEAD", "OPTIONS"]:
               return True

           request_token = request.headers.get("X-CSRF-Token")
           session_token = request.session.get("csrf_token")

           if not request_token or not session_token:
               return False

           return secrets.compare_digest(request_token, session_token)

   csrf_protection = CSRFProtection()

   class RateLimiter:
       def __init__(self):
           self.requests = {}
           self.window_size = 60  # 1 minute window
           self.max_requests = 100  # Maximum requests per window
           self.block_duration = 300  # 5 minutes block duration
           self.blocked_ips = {}

       def is_blocked(self, ip: str) -> bool:
           """Check if IP is blocked"""
           if ip in self.blocked_ips:
               block_time = self.blocked_ips[ip]
               if time.time() - block_time > self.block_duration:
                   del self.blocked_ips[ip]
                   return False
               return True
           return False

       def is_rate_limited(self, ip: str) -> bool:
           """Check if request should be rate limited"""
           current_time = time.time()

           # Clean up old requests
           self.requests[ip] = [
               req_time for req_time in self.requests.get(ip, [])
               if current_time - req_time < self.window_size
           ]

           # Check rate limit
           if len(self.requests[ip]) >= self.max_requests:
               self.blocked_ips[ip] = current_time
               return True

           self.requests[ip].append(current_time)
           return False

   rate_limiter = RateLimiter()

   @app.middleware("http")
   async def rate_limit_middleware(request: Request, call_next: Callable) -> Response:
       client_ip = request.client.host

       if rate_limiter.is_blocked(client_ip):
           return Response(
               status_code=403,
               content='{"detail":"You have been blocked due to too many requests"}',
               media_type="application/json"
           )

       if rate_limiter.is_rate_limited(client_ip):
           return Response(
               status_code=429,
               content='{"detail":"Too many requests"}',
               media_type="application/json"
           )

       response = await call_next(request)
       return response
   ```

2. HTML Sanitization for XSS Prevention:
   backend/app/utils/sanitizer.py:

   ```python
   import bleach
   from typing import Any, Dict
   import json

   class HTMLSanitizer:
       ALLOWED_TAGS = [
           'a', 'abbr', 'acronym', 'b', 'blockquote', 'code',
           'em', 'i', 'li', 'ol', 'p', 'strong', 'ul'
       ]

       ALLOWED_ATTRIBUTES = {
           'a': ['href', 'title'],
           '*': ['class']
       }

       @classmethod
       def sanitize_html(cls, content: str) -> str:
           """Sanitize HTML content to prevent XSS"""
           return bleach.clean(
               content,
               tags=cls.ALLOWED_TAGS,
               attributes=cls.ALLOWED_ATTRIBUTES,
               strip=True
           )

       @classmethod
       def sanitize_json(cls, data: Dict[str, Any]) -> Dict[str, Any]:
           """Recursively sanitize JSON data"""
           if isinstance(data, str):
               return cls.sanitize_html(data)
           elif isinstance(data, dict):
               return {k: cls.sanitize_json(v) for k, v in data.items()}
           elif isinstance(data, list):
               return [cls.sanitize_json(item) for item in data]
           return data

       @classmethod
       def clean_user_input(cls, input_data: Any) -> Any:
           """Clean any user input"""
           try:
               if isinstance(input_data, str):
                   # Try to parse as JSON first
                   try:
                       json_data = json.loads(input_data)
                       return cls.sanitize_json(json_data)
                   except json.JSONDecodeError:
                       return cls.sanitize_html(input_data)
               elif isinstance(input_data, (dict, list)):
                   return cls.sanitize_json(input_data)
               return input_data
           except Exception as e:
               logger.error(f"Error sanitizing input: {e}")
               return input_data
   ```

## Part 5: Secure File Handling

1. Secure File Upload Handler:
   backend/app/services/file_handling.py:

   ```python
   from fastapi import UploadFile, HTTPException
   import magic
   import hashlib
   import os
   from typing import List, Optional, Tuple
   from datetime import datetime
   import aiofiles
   import asyncio
   from app.core.config import settings
   import logging
   from pathlib import Path

   class SecureFileHandler:
       ALLOWED_EXTENSIONS = {'.gpx'}
       ALLOWED_MIME_TYPES = ['application/gpx+xml', 'text/xml']
       MAX_FILE_SIZE = 10 * 1024 * 1024  # 10MB

       def __init__(self):
           self.upload_dir = Path(settings.UPLOAD_DIR)
           self.temp_dir = Path(settings.TEMP_DIR)
           self.ensure_directories()

       def ensure_directories(self) -> None:
           """Ensure required directories exist with proper permissions"""
           self.upload_dir.mkdir(mode=0o750, parents=True, exist_ok=True)
           self.temp_dir.mkdir(mode=0o750, parents=True, exist_ok=True)

       async def save_uploaded_file(self, file: UploadFile) -> Tuple[str, str]:
           """Securely save uploaded file"""
           try:
               # Validate file before processing
               await self.validate_file(file)

               # Generate secure filename
               original_filename = file.filename
               extension = Path(original_filename).suffix.lower()
               timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
               secure_filename = f"{timestamp}_{hashlib.sha256(original_filename.encode()).hexdigest()[:12]}{extension}"

               # Create temporary file first
               temp_path = self.temp_dir / secure_filename
               async with aiofiles.open(temp_path, 'wb') as temp_file:
                   # Read and write in chunks to handle large files
                   chunk_size = 8192
                   while chunk := await file.read(chunk_size):
                       await temp_file.write(chunk)

               # Perform additional security checks on saved file
               await self.verify_saved_file(temp_path)

               # Move to final location
               final_path = self.upload_dir / secure_filename
               temp_path.rename(final_path)

               # Set proper permissions
               os.chmod(final_path, 0o640)

               return str(final_path), secure_filename

           except Exception as e:
               # Clean up temporary file if it exists
               if 'temp_path' in locals() and temp_path.exists():
                   temp_path.unlink()
               logging.error(f"File upload error: {e}")
               raise

       async def validate_file(self, file: UploadFile) -> None:
           """Validate file before processing"""
           # Check file size
           file.file.seek(0, 2)
           size = file.file.tell()
           file.file.seek(0)

           if size > self.MAX_FILE_SIZE:
               raise HTTPException(status_code=413, detail="File too large")

           # Check file extension
           extension = Path(file.filename).suffix.lower()
           if extension not in self.ALLOWED_EXTENSIONS:
               raise HTTPException(status_code=415, detail="Invalid file type")

           # Check MIME type
           content = await file.read(2048)  # Read first 2KB for MIME detection
           mime_type = magic.from_buffer(content, mime=True)
           file.file.seek(0)

           if mime_type not in self.ALLOWED_MIME_TYPES:
               raise HTTPException(status_code=415, detail="Invalid file type")

       async def verify_saved_file(self, file_path: Path) -> None:
           """Verify saved file integrity and content"""
           if not file_path.exists():
               raise HTTPException(status_code=500, detail="File save error")

           # Verify file size again
           if file_path.stat().st_size > self.MAX_FILE_SIZE:
               file_path.unlink()
               raise HTTPException(status_code=413, detail="File too large")

           # Verify file content
           async with aiofiles.open(file_path, 'rb') as f:
               content = await f.read(2048)
               mime_type = magic.from_buffer(content, mime=True)
               if mime_type not in self.ALLOWED_MIME_TYPES:
                   file_path.unlink()
                   raise HTTPException(status_code=415, detail="Invalid file type")

       async def secure_delete(self, file_path: Path) -> None:
           """Securely delete file"""
           try:
               if not file_path.exists():
                   return

               # Overwrite file content before deletion
               file_size = file_path.stat().st_size
               async with aiofiles.open(file_path, 'wb') as f:
                   # First overwrite with zeros
                   await f.write(b'0' * file_size)
                   await f.flush()
                   # Then overwrite with random data
                   await f.write(os.urandom(file_size))
                   await f.flush()

               # Finally delete the file
               file_path.unlink()

           except Exception as e:
               logging.error(f"Secure delete error: {e}")
               raise

   class FileDownloadHandler:
       def __init__(self):
           self.file_handler = SecureFileHandler()

       async def get_file_stream(self, file_path: Path, chunk_size: int = 8192):
           """Stream file for download with security checks"""
           if not file_path.exists():
               raise HTTPException(status_code=404, detail="File not found")

           # Verify file is within allowed directory
           if not str(file_path).startswith(str(self.file_handler.upload_dir)):
               raise HTTPException(status_code=403, detail="Access denied")

           try:
               async with aiofiles.open(file_path, 'rb') as file:
                   while chunk := await file.read(chunk_size):
                       yield chunk
           except Exception as e:
               logging.error(f"File download error: {e}")
               raise HTTPException(status_code=500, detail="Download error")

   class FileScanner:
       async def scan_file(self, file_path: Path) -> bool:
           """Scan file for malware (integration with antivirus)"""
           try:
               # Implement your preferred antivirus integration here
               # Example using ClamAV:
               if settings.ENABLE_VIRUS_SCAN:
                   process = await asyncio.create_subprocess_exec(
                       'clamdscan',
                       str(file_path),
                       '--no-summary',
                       stdout=asyncio.subprocess.PIPE,
                       stderr=asyncio.subprocess.PIPE
                   )
                   stdout, stderr = await process.communicate()
                   return process.returncode == 0
               return True
           except Exception as e:
               logging.error(f"File scan error: {e}")
               return False

   # Initialize handlers
   file_handler = SecureFileHandler()
   download_handler = FileDownloadHandler()
   file_scanner = FileScanner()
   ```

2. File Storage Configuration:
   backend/app/core/storage.py:

   ```python
   from pathlib import Path
   import shutil
   import os
   from datetime import datetime, timedelta
   import logging

   class StorageManager:
       def __init__(self, base_path: Path):
           self.base_path = base_path
           self.temp_path = base_path / 'temp'
           self.upload_path = base_path / 'uploads'
           self.ensure_storage_structure()

       def ensure_storage_structure(self) -> None:
           """Ensure storage directory structure exists with proper permissions"""
           os.umask(0o077)  # Set restrictive default permissions

           for path in [self.base_path, self.temp_path, self.upload_path]:
               path.mkdir(mode=0o750, parents=True, exist_ok=True)

       def cleanup_temp_files(self, max_age: timedelta = timedelta(hours=1)) -> None:
           """Clean up old temporary files"""
           try:
               current_time = datetime.now()

               for item in self.temp_path.glob('*'):
                   if item.is_file():
                       file_age = datetime.fromtimestamp(item.stat().st_mtime)
                       if current_time - file_age > max_age:
                           self.secure_delete(item)

           except Exception as e:
               logging.error(f"Temp cleanup error: {e}")

       def secure_delete(self, path: Path) -> None:
           """Securely delete a file or directory"""
           try:
               if path.is_file():
                   # Overwrite file before deletion
                   with open(path, 'wb') as f:
                       f.write(os.urandom(path.stat().st_size))
                   path.unlink()
               elif path.is_dir():
                   shutil.rmtree(path)

           except Exception as e:
               logging.error(f"Secure delete error: {e}")
               raise

       def rotate_logs(self) -> None:
           """Rotate and compress old log files"""
           log_dir = self.base_path / 'logs'
           if not log_dir.exists():
               return

           try:
               for log_file in log_dir.glob('*.log'):
                   if log_file.stat().st_size > settings.MAX_LOG_SIZE:
                       timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
                       archived_name = f"{log_file.stem}_{timestamp}.log.gz"
                       self.compress_file(log_file, log_dir / archived_name)

           except Exception as e:
               logging.error(f"Log rotation error: {e}")

       def compress_file(self, source: Path, destination: Path) -> None:
           """Compress a file using gzip"""
           try:
               with open(source, 'rb') as f_in:
                   with gzip.open(destination, 'wb') as f_out:
                       shutil.copyfileobj(f_in, f_out)
               source.unlink()
           except Exception as e:
               logging.error(f"File compression error: {e}")
               raise
   ```
