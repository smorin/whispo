# Whispo Repository Analysis

## Overview

Whispo is an AI-powered dictation tool built with Electron, React, and Rust. It provides voice-to-text functionality with real-time transcription using various AI providers (OpenAI Whisper, Groq) and optional post-processing with LLMs (OpenAI, Groq, Gemini).

## Architecture

### High-Level Architecture

```mermaid
graph TB
    subgraph "User Interface"
        A[User] -->|Holds Ctrl Key| B[Keyboard Listener<br/>whispo-rs]
        B -->|Key Events| C[Main Process<br/>Electron]
    end
    
    subgraph "Application Core"
        C -->|Show/Hide| D[Panel Window]
        C -->|Manage| E[Main Window]
        C -->|Initial Setup| F[Setup Window]
        
        D -->|Audio Stream| G[Audio Recorder<br/>MediaRecorder API]
        G -->|Audio Data| H[IPC Router<br/>tipc]
    end
    
    subgraph "AI Processing"
        H -->|Audio Buffer| I[STT Providers<br/>OpenAI/Groq Whisper]
        I -->|Transcript| J[LLM Post-Processing<br/>OpenAI/Groq/Gemini]
        J -->|Final Text| H
    end
    
    subgraph "Output"
        H -->|Text| K[Keyboard Writer<br/>whispo-rs]
        K -->|Types Text| L[Active Application]
    end
```

### Component Architecture

```mermaid
graph LR
    subgraph "Main Process"
        A[index.ts] --> B[window.ts]
        A --> C[keyboard.ts]
        A --> D[tipc.ts]
        A --> E[menu.ts]
        A --> F[tray.ts]
        
        C --> G[whispo-rs<br/>Rust Binary]
        D --> H[llm.ts]
        D --> I[config.ts]
        D --> J[state.ts]
    end
    
    subgraph "Renderer Process"
        K[App.tsx] --> L[Router]
        L --> M[Pages]
        M --> N[Panel]
        M --> O[Settings]
        M --> P[Setup]
        
        N --> Q[recorder.ts]
        Q --> R[tipc-client.ts]
    end
    
    subgraph "Preload"
        S[preload/index.ts] --> T[IPC Bridge]
    end
    
    T -.-> D
    R -.-> T
```

### Data Flow

```mermaid
sequenceDiagram
    participant User
    participant Keyboard as whispo-rs (Keyboard)
    participant Main as Main Process
    participant Panel as Panel Window
    participant Recorder as Audio Recorder
    participant IPC as IPC Router
    participant STT as STT Provider
    participant LLM as LLM Provider
    participant Writer as whispo-rs (Writer)
    participant App as Active App
    
    User->>Keyboard: Hold Ctrl Key
    Keyboard->>Main: KeyPress Event
    Main->>Panel: Show & Start Recording
    Panel->>Recorder: Start Recording
    Recorder->>Recorder: Capture Audio
    
    User->>Keyboard: Release Ctrl Key
    Keyboard->>Main: KeyRelease Event
    Main->>Panel: Stop Recording
    Panel->>Recorder: Stop Recording
    Recorder->>IPC: Send Audio Buffer
    
    IPC->>STT: Transcribe Audio
    STT->>IPC: Return Transcript
    
    opt Post-Processing Enabled
        IPC->>LLM: Process Transcript
        LLM->>IPC: Return Processed Text
    end
    
    IPC->>Writer: Write Text
    Writer->>App: Type Text
    IPC->>Main: Hide Panel
```

## Project Structure

```
whispo/
├── src/
│   ├── main/              # Electron main process
│   │   ├── index.ts       # Entry point
│   │   ├── window.ts      # Window management
│   │   ├── keyboard.ts    # Keyboard event handling
│   │   ├── llm.ts         # LLM integrations
│   │   ├── tipc.ts        # IPC router
│   │   ├── config.ts      # Configuration store
│   │   ├── state.ts       # Application state
│   │   ├── tray.ts        # System tray
│   │   ├── menu.ts        # Application menu
│   │   └── updater.ts     # Auto-updater
│   │
│   ├── renderer/          # Electron renderer process
│   │   └── src/
│   │       ├── App.tsx    # React app entry
│   │       ├── pages/     # Application pages
│   │       ├── components/# React components
│   │       └── lib/       # Utilities & libraries
│   │
│   ├── preload/           # Preload scripts
│   │   └── index.ts       # IPC bridge
│   │
│   └── shared/            # Shared types & utilities
│       └── types.ts       # TypeScript types
│
├── whispo-rs/             # Rust keyboard listener
│   └── src/
│       └── main.rs        # Keyboard/mouse events
│
├── resources/             # Application resources
│   └── bin/              # Platform binaries
│
└── build/                # Build output
```

## Key Features

### 1. Voice Recording
- **Trigger Methods**:
  - Hold Ctrl key for 800ms to start recording
  - Release Ctrl to stop and transcribe
  - Alternative: Ctrl+/ shortcut for toggle recording
  - Escape key to cancel recording

### 2. AI Integration
- **Speech-to-Text Providers**:
  - OpenAI Whisper API
  - Groq Whisper API
  - Custom API URL support

- **LLM Post-Processing**:
  - OpenAI GPT models
  - Groq LLMs
  - Google Gemini
  - Custom prompts for text transformation

### 3. Window System
- **Main Window**: Application settings and history
- **Panel Window**: Floating recording indicator with audio visualizer
- **Setup Window**: Initial permissions and configuration

### 4. Platform Integration
- **Cross-platform**: macOS (Apple Silicon) and Windows x64
- **System Integration**:
  - Accessibility permissions for text input
  - Microphone permissions for audio recording
  - System tray integration
  - Auto-updater support

### 5. Data Management
- **Local Storage**: All data stored locally
- **Recording History**: Saved recordings with transcripts
- **Configuration**: User preferences and API keys

## Technology Stack

### Frontend
- **Framework**: React 18 with TypeScript
- **Routing**: React Router v6
- **State Management**: React Query (TanStack Query)
- **Styling**: Tailwind CSS with custom components
- **UI Components**: Radix UI primitives
- **Build Tool**: Vite

### Backend
- **Runtime**: Electron 31
- **IPC**: @egoist/tipc for type-safe IPC
- **Window Management**: @egoist/electron-panel-window
- **Configuration**: Electron Store
- **Auto-updater**: electron-updater

### Native Integration
- **Language**: Rust
- **Keyboard Events**: rdev library
- **Text Input**: enigo library
- **Build**: Cargo

### AI Services
- **OpenAI**: Whisper API, GPT models
- **Groq**: Whisper API, LLM models
- **Google**: Gemini API

## Security Considerations

1. **API Keys**: Stored locally in encrypted Electron store
2. **Permissions**: Requires explicit user consent for:
   - Microphone access
   - Accessibility permissions (for text input)
3. **Data Privacy**: All processing can be done locally or with user-chosen APIs
4. **License**: AGPL-3.0 (copyleft license)

## Development Workflow

### Scripts
- `npm run dev`: Start development server with hot reload
- `npm run build`: Build for production
- `npm run build:mac/win/linux`: Platform-specific builds
- `npm run lint`: Run ESLint
- `npm run typecheck`: TypeScript type checking
- `npm run release`: Create a new release

### Build Process
1. TypeScript compilation for Electron code
2. Vite build for React application
3. Rust compilation for keyboard listener
4. Electron Builder for packaging

## Future Considerations

Based on the codebase analysis, potential areas for enhancement:
1. Support for more STT providers
2. Custom keyboard shortcuts configuration
3. Multi-language support
4. Cloud sync for settings/history
5. Plugin system for custom processors
6. Support for more platforms (Linux ARM, etc.)