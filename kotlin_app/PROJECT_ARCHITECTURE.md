# Madhu-Marga: Complete Architecture Documentation

## Table of Contents
1. [Project Overview](#project-overview)
2. [Architecture Pattern](#architecture-pattern)
3. [Technology Stack](#technology-stack)
4. [Database Schema](#database-schema)
5. [Data Flow](#data-flow)
6. [Component Structure](#component-structure)
7. [Module Organization](#module-organization)
8. [Key Features Architecture](#key-features-architecture)
9. [Security & Performance](#security--performance)

---

## Project Overview

**Madhu-Marga** is an AI-guided beekeeping assistant built with Kotlin for Android. It enables farmers to:
- Register and monitor individual hives
- Log hive inspections with photo evidence
- Track honey harvest production
- Receive AI-powered recommendations via Gemini API
- Sync data offline-first to Supabase backend

**Target User**: Small-scale farmers and beekeeping hobbyists in India

---

## Architecture Pattern

### MVVM + Clean Architecture
```
┌─────────────────────────────────────────┐
│       UI Layer (Jetpack Compose)        │
│  - Screens, Composables, State          │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│    ViewModel Layer (State Management)    │
│  - StateFlow, LiveData, ViewModel        │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│    Repository Layer (Data Access)       │
│  - Business Logic, Data Aggregation     │
└──────────┬──────────────┬────────────────┘
           │              │
    ┌──────▼────┐    ┌────▼─────────┐
    │ Local DB  │    │ Remote API   │
    │ (Room)    │    │ (Supabase)   │
    └───────────┘    └──────────────┘
```

### Data Flow Pattern
```
User Action → Composable → ViewModel → Repository → DataSource → API/DB
                                    ↑                              ↓
                          ← StateFlow Updates ←────────────────────
```

---

## Technology Stack

### Frontend
| Technology | Version | Purpose |
|-----------|---------|---------|
| Kotlin | Latest | Primary Language |
| Jetpack Compose | Latest | Modern UI Framework |
| Material 3 | Latest | Design System |
| Navigation Compose | Latest | Screen Navigation |
| ViewModel | Latest | State Management |
| StateFlow/LiveData | Latest | Reactive Data Binding |

### Database & Storage
| Technology | Version | Purpose |
|-----------|---------|---------|
| Room | 2.6.1 | Local SQLite Database |
| Supabase | - | Remote PostgreSQL + Auth |
| Firebase Storage | - | Image/Document Storage |

### External APIs & Services
| Service | Purpose |
|---------|---------|
| Google Gemini API | AI Recommendations |
| Firebase Cloud Messaging | Push Notifications |
| Firebase Crashlytics | Error Tracking |
| Google OAuth | User Authentication |

### Background & Infrastructure
| Technology | Purpose |
|-----------|---------|
| WorkManager | Scheduled Sync Tasks |
| Hilt | Dependency Injection |
| Retrofit | REST API Calls |
| Timber | Logging |

---

## Database Schema

### Core Tables

#### 1. **Hives Table**
Stores individual hive information.
```sql
CREATE TABLE hives (
    id TEXT PRIMARY KEY,
    userId TEXT NOT NULL,
    name TEXT NOT NULL,
    location TEXT,
    latitude REAL,
    longitude REAL,
    hiveType TEXT,          -- 'Langstroth', 'TopBar', 'Warre'
    installationDate TEXT,
    queenBreedType TEXT,
    status TEXT,            -- 'Active', 'Inactive', 'Dead'
    healthScore REAL,       -- 0-100
    lastInspectionDate TEXT,
    notes TEXT,
    createdAt TEXT,
    updatedAt TEXT,
    syncStatus TEXT         -- 'synced', 'pending', 'failed'
);
```

#### 2. **Inspections Table**
Detailed inspection logs with observations.
```sql
CREATE TABLE inspections (
    id TEXT PRIMARY KEY,
    hiveId TEXT NOT NULL,
    inspectionDate TEXT,
    queenPresent BOOLEAN,
    eggsPresent BOOLEAN,
    larvaePresent BOOLEAN,
    cappedBrood INTEGER,    -- Count
    openBrood INTEGER,
    pestPresent BOOLEAN,
    pestType TEXT,          -- 'Mites', 'Beetles', 'None'
    pestCount INTEGER,
    honeyFrames INTEGER,
    polleneFrames INTEGER,
    diseasePresent BOOLEAN,
    diseaseType TEXT,
    activityLevel TEXT,     -- 'Low', 'Moderate', 'High'
    temperatureInside REAL,
    humidity REAL,
    flightBoard TEXT,       -- 'Weak', 'Normal', 'Strong'
    notes TEXT,
    photosCount INTEGER,
    aiRecommendation TEXT,
    createdAt TEXT,
    updatedAt TEXT
);
```

#### 3. **Harvests Table**
Honey production tracking.
```sql
CREATE TABLE harvests (
    id TEXT PRIMARY KEY,
    hiveId TEXT NOT NULL,
    harvestDate TEXT,
    quantityKg REAL,
    qualityRating INTEGER,  -- 1-5
    sugarContentBrix REAL,
    moistureLevel REAL,
    harvestNotes TEXT,
    weatherCondition TEXT,
    createdAt TEXT,
    updatedAt TEXT
);
```

#### 4. **UserProfile Table**
User account and preferences.
```sql
CREATE TABLE userProfile (
    id TEXT PRIMARY KEY,
    email TEXT NOT NULL UNIQUE,
    userName TEXT NOT NULL,
    phoneNumber TEXT,
    state TEXT,
    district TEXT,
    farmSize REAL,          -- In acres
    yearsOfExperience INTEGER,
    farmingMethod TEXT,     -- 'Organic', 'Conventional'
    preferredLanguage TEXT,
    pushNotificationsEnabled BOOLEAN,
    profilePhotoUrl TEXT,
    createdAt TEXT,
    updatedAt TEXT
);
```

#### 5. **HiveImages Table**
Photo attachments for inspections.
```sql
CREATE TABLE hiveImages (
    id TEXT PRIMARY KEY,
    inspectionId TEXT NOT NULL,
    imageUrl TEXT,
    localPath TEXT,
    uploadStatus TEXT,      -- 'pending', 'uploading', 'completed'
    uploadedAt TEXT,
    createdAt TEXT
);
```

#### 6. **Reminders Table**
Scheduled tasks and notifications.
```sql
CREATE TABLE reminders (
    id TEXT PRIMARY KEY,
    userId TEXT NOT NULL,
    hiveId TEXT NOT NULL,
    reminderType TEXT,      -- 'Inspection', 'Harvest', 'Treatment'
    dueDate TEXT,
    description TEXT,
    isCompleted BOOLEAN,
    completedDate TEXT,
    createdAt TEXT
);
```

### Relationships
```
User (1) ──→ (Many) Hives
Hive (1) ──→ (Many) Inspections
Hive (1) ──→ (Many) Harvests
Hive (1) ──→ (Many) Reminders
Inspection (1) ──→ (Many) HiveImages
```

---

## Data Flow

### User Inspection Flow
```
1. User opens "New Inspection" screen
   ↓
2. Selects hive from list (queries local Hives table)
   ↓
3. Fills inspection checklist form
   ↓
4. Takes photos (stored locally + Firebase)
   ↓
5. Submits form → Room INSERT into Inspections table
   ↓
6. ViewModel notifies Repository
   ↓
7. Repository triggers AI Recommendation (Gemini API)
   ↓
8. WorkManager schedules sync to Supabase (15-min interval)
   ↓
9. UI updates via StateFlow with new data
```

### Offline-First Sync Mechanism
```
User creates data (Online/Offline)
   ↓
1. Saved locally to Room DB
2. Mark as "pending" in syncStatus
   ↓
3. If Online:
   - Immediate attempt to sync to Supabase
   - Update syncStatus to "synced"
   ↓
4. If Offline:
   - WorkManager checks connectivity every 15 min
   - Syncs accumulated changes once connection available
   - Implements conflict resolution (timestamp-based)
```

### AI Recommendation Flow
```
User submits inspection
   ↓
1. Repository sends data to Gemini API with prompt:
   "Analyze this hive inspection and provide recommendations"
   ↓
2. Gemini processes:
   - Pest presence/type
   - Brood pattern
   - Food stores
   - Health indicators
   ↓
3. Returns AI recommendation
   ↓
4. Stored in inspections.aiRecommendation
   ↓
5. Displayed on inspection detail screen
```

---

## Component Structure

### Key Packages

#### `com.madhumarga.ui.screens`
- `DashboardScreen` - Main home screen
- `HiveRegisterScreen` - Add/edit hives
- `InspectionLogScreen` - Create inspections
- `HarvestTrackerScreen` - Log honey production
- `FloraCalendarScreen` - Seasonal flowers guide
- `SettingsScreen` - User preferences
- `AuthScreen` - Login/signup

#### `com.madhumarga.viewmodel`
- `DashboardViewModel` - Dashboard state
- `HiveViewModel` - Hive CRUD operations
- `InspectionViewModel` - Inspection management
- `HarvestViewModel` - Harvest tracking
- `AuthViewModel` - User authentication

#### `com.madhumarga.repository`
- `HiveRepository` - Hive data access
- `InspectionRepository` - Inspection data
- `HarvestRepository` - Harvest data
- `UserRepository` - User profile
- `SyncRepository` - Remote sync logic

#### `com.madhumarga.data.local`
- `AppDatabase` - Room database instance
- `HiveDao` - Hive queries
- `InspectionDao` - Inspection queries
- `HarvestDao` - Harvest queries

#### `com.madhumarga.data.remote`
- `SupabaseClient` - API configuration
- `GeminiApiClient` - AI API calls
- `FirebaseStorageHelper` - Image uploads

#### `com.madhumarga.worker`
- `SyncWorker` - Periodic sync with backend
- `NotificationWorker` - Push notification handling
- `ReminderWorker` - Reminder scheduling

#### `com.madhumarga.util`
- `Constants` - App-wide constants
- `DateTimeUtils` - Date/time helpers
- `ImageUtils` - Image processing
- `LoggingUtils` - Timber configuration

---

## Key Features Architecture

### 1. Hive Register
```
Register Screen
  ├─ Input: Hive ID, Location, Type
  ├─ Storage: Room DB (Hives table)
  ├─ Display: List with status indicators
  └─ Sync: WorkManager → Supabase
```

### 2. Inspection Logger
```
Inspection Screen
  ├─ Checklist: Queen, Pests, Disease, Activity
  ├─ Photos: Attached to inspection record
  ├─ Storage: Room (Inspections + HiveImages)
  ├─ AI: Sends to Gemini for recommendations
  └─ Display: History with AI suggestions
```

### 3. Harvest Tracker
```
Harvest Screen
  ├─ Input: Date, Quantity, Quality Rating
  ├─ Storage: Room DB (Harvests table)
  ├─ Analytics: Seasonal trends, yield per hive
  └─ Export: CSV/PDF reports
```

### 4. AI Recommendations
```
Gemini Integration
  ├─ Trigger: On inspection submission
  ├─ Input: Inspection data + hive history
  ├─ Processing: AI analyzes patterns
  ├─ Output: Actionable recommendations
  └─ Storage: Saved in inspection record
```

### 5. Push Notifications
```
Firebase Messaging
  ├─ Triggers:
  │  ├─ High temperature alert
  │  ├─ Humidity anomaly
  │  ├─ Inspection reminder
  │  └─ Harvest season alert
  ├─ Processing: NotificationWorker
  └─ Display: Android Notification channels
```

### 6. User Authentication
```
Auth Flow
  ├─ Methods:
  │  ├─ Google OAuth
  │  ├─ Email/Password (Supabase)
  │  └─ Biometric (Optional)
  ├─ Storage: Encrypted SharedPreferences
  └─ Session: Token-based with refresh
```

---

## Security & Performance

### Security Measures
1. **Data Encryption**
   - Encrypted SharedPreferences for tokens
   - SSL/TLS for all network calls
   - Supabase Row Level Security (RLS)

2. **Authentication**
   - OAuth 2.0 with Google
   - JWT tokens with 1-hour expiry
   - Refresh token rotation

3. **API Rate Limiting**
   - 2-5 second delays between requests
   - Prevents abuse of Gemini API
   - Exponential backoff on failures

4. **Data Validation**
   - Input sanitization in all forms
   - Type-safe Room database
   - Network response validation

### Performance Optimizations
1. **Database**
   - Indexed queries on frequently accessed columns
   - Pagination for large lists
   - Eager loading with Room relations

2. **Caching**
   - In-memory cache for user profile
   - StateFlow prevents unnecessary recompositions
   - Image caching with Coil/Picasso

3. **Network**
   - Batch sync operations
   - Offline-first reduces network calls
   - WorkManager prevents duplicate requests

4. **UI**
   - Lazy loading in lists
   - Compose recomposition optimization
   - Efficient state management

---

## Build Configuration

### Gradle Dependencies
```gradle
// Core AndroidX
androidx.core:core:1.12.0
androidx.appcompat:appcompat:1.6.1
androidx.lifecycle:lifecycle-runtime-ktx:2.6.2

// Jetpack Compose
androidx.compose.ui:ui:1.5.4
androidx.compose.material3:material3:1.1.2
androidx.compose.navigation:navigation-compose:2.7.5

// Database
androidx.room:room-runtime:2.6.1
androidx.room:room-ktx:2.6.1

// Dependency Injection
com.google.dagger:hilt-android:2.48

// API & Networking
com.squareup.retrofit2:retrofit:2.10.0
io.supabase:supabase-kt:1.4.12

// Firebase
com.google.firebase:firebase-messaging:23.3.1
com.google.firebase:firebase-crashlytics:18.6.0

// Image Loading
io.coil-kt:coil-compose:2.5.0

// Logging
com.jakewharton.timber:timber:5.0.1
```

---

## Deployment

### Build Process
```bash
# Debug Build
./gradlew assembleDebug

# Release Build (signed)
./gradlew assembleRelease

# Install on device
./gradlew installDebug
```

### Release Checklist
- [ ] Update version code/name in build.gradle
- [ ] Update README with new features
- [ ] Test on multiple Android versions (API 21+)
- [ ] Verify offline functionality
- [ ] Test sync mechanism
- [ ] Firebase crash reporting enabled
- [ ] ProGuard rules configured
- [ ] Sign APK with release key
- [ ] Upload to Play Store

---

## Future Enhancements

1. **ML Integration**
   - Pest detection via image recognition
   - Predictive harvest timing
   - Disease early warning

2. **Social Features**
   - Community forum for beekeepers
   - Knowledge sharing platform
   - Experience rating system

3. **IoT Integration**
   - Temperature/humidity sensors
   - Real-time hive monitoring
   - Automated alerts

4. **Advanced Analytics**
   - Machine learning for yield prediction
   - Carbon offset calculations
   - Economic ROI analysis

5. **Localization**
   - Support for Indian regional languages
   - Region-specific flora calendar
   - Local market price tracking

---

## Conclusion

Madhu-Marga demonstrates **production-grade Android development** with:
- ✅ Clean architecture principles
- ✅ Reactive programming with Compose
- ✅ Offline-first data strategy
- ✅ AI-powered features
- ✅ Enterprise-grade error handling
- ✅ User-centric design

The architecture scales well for adding new features while maintaining code quality and testability.
