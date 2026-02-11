---
name: tauri-v2-rust-backend
description: Rust backend patterns for feature-based Tauri v2 apps. Covers tauri-specta command definitions, Result error handling, event emission to frontend, service layers, and type contracts. Use when implementing Rust features that coordinate with React frontend via typed IPC.
---

# Tauri v2 Rust Backend Architecture

Backend patterns for building Rust features in Tauri v2 desktop apps with type-safe IPC, proper error handling, and frontend coordination.

## When to Apply

- Implementing Rust commands in `src-tauri/src/features/`
- Creating type-safe IPC contracts with tauri-specta
- Emitting events to React frontend
- Defining error types for frontend handling
- Working on backend while frontend agent handles React side

## Critical Backend Rules

### 1. Command Definition (tauri-specta)

All commands MUST have both attributes for type generation.

```rust
use tauri::command;
use specta::Type;
use serde::{Serialize, Deserialize};

#[derive(Debug, Clone, Serialize, Deserialize, Type)]
pub struct Template {
    pub id: String,
    pub name: String,
    pub levels: Vec<TemplateLevel>,
}

// ✅ RIGHT - Both attributes required
#[command]
#[specta::specta]
pub fn list_templates() -> Result<Vec<Template>, String> {
    // Implementation
    Ok(templates)
}

// ❌ WRONG - Missing specta attribute
#[command]
pub fn list_templates() -> Result<Vec<Template>, String> {
    // Won't generate TypeScript bindings!
}
```

**Command rules:**
- Use `#[command]` for Tauri runtime
- Use `#[specta::specta]` for TypeScript generation
- All input/output types need `#[derive(Type)]`
- Use `Result<T, E>` for error handling
- Keep commands thin - delegate to service layer

---

### 2. Type Definitions (Specta Derives)

All types crossing IPC boundary need specta derives.

```rust
use serde::{Serialize, Deserialize};
use specta::Type;

// ✅ RIGHT - Full derives for IPC types
#[derive(Debug, Clone, Serialize, Deserialize, Type)]
pub struct CaseData {
    pub occurrence: String,
    pub business: Option<String>,
    pub location: Option<String>,
    pub video_start: Option<String>,
    pub video_end: Option<String>,
}

// ✅ RIGHT - Enums with tag for discriminated unions
#[derive(Debug, Clone, Serialize, Deserialize, Type)]
#[serde(tag = "type")]
pub enum ProcessingError {
    PathNotFound { path: String },
    HashMismatch { file: String, expected: String, actual: String },
    PermissionDenied { path: String },
    IoError { message: String },
    Cancelled,
}
```

**Type rules:**
- Always include `Serialize`, `Deserialize`, `Type`
- Add `Debug`, `Clone` for convenience
- Use `#[serde(tag = "type")]` for enums (frontend gets discriminated unions)
- Avoid `i64`/`u64` (JavaScript can't handle 64-bit integers) - use `f64` or `i32`
- Document fields with `///` (becomes JSDoc in TypeScript)

---

### 3. Result Pattern (Error Handling)

Commands return `Result<T, E>` for type-safe error handling on frontend.

```rust
// ✅ RIGHT - Typed error enum
#[command]
#[specta::specta]
pub fn save_template(template: Template) -> Result<(), TemplateError> {
    // Validate
    if template.name.is_empty() {
        return Err(TemplateError::ValidationError { 
            message: "Name cannot be empty".to_string() 
        });
    }
    
    // Save
    save_to_disk(&template)
        .map_err(|e| TemplateError::IoError { 
            message: e.to_string() 
        })?;
    
    Ok(())
}

// ✅ GOOD - String error (simpler, less frontend ceremony)
#[command]
#[specta::specta]
pub fn delete_template(id: String) -> Result<(), String> {
    if !template_exists(&id) {
        return Err(format!("Template {} not found", id));
    }
    
    std::fs::remove_file(template_path(&id))
        .map_err(|e| e.to_string())?;
    
    Ok(())
}
```

**Error patterns:**
- **Typed enum**: When frontend needs to match on error type
- **String**: For simple errors, user-facing messages
- **Never panic in commands** - always return Result
- **Frontend gets:** `{ status: 'ok', data: T }` or `{ status: 'error', error: E }`

---

### 4. Event Emission (Backend → Frontend)

Emit events for real-time updates to React frontend.

```rust
use tauri::{AppHandle, Manager};

// Real-time progress updates
#[derive(Clone, Serialize, Type)]
pub struct ProgressPayload {
    pub percent: f32,
    pub current_file: String,
    pub speed_mbps: f32,
}

#[command]
#[specta::specta]
pub async fn process_files(
    files: Vec<String>,
    app: AppHandle,
) -> Result<ProcessResult, ProcessingError> {
    for (i, file) in files.iter().enumerate() {
        // Calculate progress
        let percent = (i as f32 / files.len() as f32) * 100.0;
        
        // Emit to frontend
        app.emit("operation:progress", ProgressPayload {
            percent,
            current_file: file.clone(),
            speed_mbps: calculate_speed(),
        })?;
        
        // Process file...
        process_single_file(file)?;
    }
    
    Ok(ProcessResult { files_processed: files.len() })
}
```

**Event patterns:**
- Use kebab-case: `feature:event-name` (e.g., `processing:progress`)
- Payload must `#[derive(Serialize, Type)]`
- Emit frequently for high-frequency updates (progress bars)
- Frontend subscribes with `TauriEvents.listen('event-name', handler)`

---

### 5. Feature Organization

Features are organized as modules mirroring frontend structure.

```
src-tauri/src/features/
├── mod.rs                    # Feature registry
├── preferences/
│   ├── commands/
│   │   └── mod.rs            # Public commands
│   ├── services/
│   │   ├── mod.rs
│   │   └── storage.rs        # Business logic
│   └── types/
│       └── mod.rs            # Feature types
├── processing/
│   ├── commands/mod.rs
│   ├── services/
│   │   ├── mod.rs
│   │   ├── hasher.rs
│   │   └── file_ops.rs
│   └── types/mod.rs
└── templates/
    ├── commands/mod.rs
    ├── services/
    │   ├── mod.rs
    │   ├── resolver.rs
    │   └── validator.rs
    └── types/mod.rs
```

**Organization rules:**
- `commands/` - Thin wrappers, validate input, call services
- `services/` - Business logic, file I/O, heavy lifting
- `types/` - Feature-specific types with specta derives
- Keep commands thin - most logic in services

---

### 6. Command Registration

Register commands in `bindings.rs` for tauri-specta.

```rust
// src-tauri/src/bindings.rs

use specta_typescript::Typescript;
use tauri_specta::{collect_commands, Builder};

// Import feature commands
use crate::features::{
    preferences::commands as prefs_commands,
    templates::commands as template_commands,
    processing::commands as processing_commands,
};

pub fn register_commands() -> Builder {
    Builder::<tauri::Wry>::new()
        .commands(collect_commands![
            // Preferences
            prefs_commands::load_preferences,
            prefs_commands::save_preferences,
            
            // Templates
            template_commands::list_templates,
            template_commands::save_template,
            template_commands::delete_template,
            
            // Processing
            processing_commands::process_files,
            processing_commands::cancel_operation,
        ])
}

#[cfg(debug_assertions)]
#[test]
fn export_bindings() {
    let builder = register_commands();
    
    builder
        .export(
            Typescript::default()
                .bigint(specta_typescript::BigIntExportBehavior::Number),
            "../src/lib/bindings.ts",
        )
        .expect("Failed to export bindings");
}
```

**Registration rules:**
- All commands MUST be registered here
- Run test to regenerate: `cargo test export_bindings -- --ignored`
- Or use `npm run tauri dev` (auto-generates on start)
- Bindings generate to `src/lib/bindings.ts` (frontend)

---

### 7. Service Layer Pattern

Commands delegate to services for business logic.

```rust
// features/templates/commands/mod.rs

use crate::features::templates::{services, types::*};

#[command]
#[specta::specta]
pub fn list_templates() -> Result<Vec<Template>, TemplateError> {
    services::list_all_templates()
}

#[command]
#[specta::specta]
pub fn save_template(template: Template) -> Result<(), TemplateError> {
    // Validate in command
    services::validator::validate_template(&template)?;
    
    // Business logic in service
    services::storage::save_template(template)
}

// features/templates/services/storage.rs

use super::types::*;
use std::fs;

pub fn save_template(template: Template) -> Result<(), TemplateError> {
    let path = get_template_path(&template.id);
    let json = serde_json::to_string_pretty(&template)
        .map_err(|e| TemplateError::IoError { 
            message: e.to_string() 
        })?;
    
    // Atomic write
    let temp_path = format!("{}.tmp", path);
    fs::write(&temp_path, json)
        .map_err(|e| TemplateError::IoError { 
            message: e.to_string() 
        })?;
    
    fs::rename(&temp_path, &path)
        .map_err(|e| TemplateError::IoError { 
            message: e.to_string() 
        })?;
    
    Ok(())
}
```

**Service responsibilities:**
- File I/O, database access
- Business logic, validation (complex rules)
- Data transformations
- Error handling
- Keep services testable (no Tauri dependencies)

---

### 8. Async Commands

Use async when doing I/O, network, or long operations.

```rust
#[command]
#[specta::specta]
pub async fn process_large_file(
    path: String,
    app: AppHandle,
) -> Result<ProcessResult, ProcessingError> {
    // Open file
    let file = tokio::fs::File::open(&path).await
        .map_err(|e| ProcessingError::IoError { 
            message: e.to_string() 
        })?;
    
    // Stream processing
    let mut reader = tokio::io::BufReader::new(file);
    let mut total = 0u64;
    
    loop {
        let chunk = read_chunk(&mut reader).await?;
        if chunk.is_empty() { break; }
        
        total += chunk.len() as u64;
        
        // Emit progress
        app.emit("processing:progress", ProgressPayload {
            bytes_processed: total,
            ..Default::default()
        })?;
    }
    
    Ok(ProcessResult { bytes: total })
}
```

**Async rules:**
- Use `async fn` for I/O-bound operations
- Use `tokio::fs` for async file operations
- Use `tokio::spawn` for concurrent tasks
- Emit progress events during long operations
- Handle cancellation (check cancel flag)

---

### 9. Error Type Best Practices

```rust
// ✅ GOOD - Descriptive variants with context
#[derive(Debug, Clone, Serialize, Deserialize, Type)]
#[serde(tag = "type")]
pub enum ProcessingError {
    PathNotFound { path: String },
    HashMismatch { 
        file: String, 
        expected: String, 
        actual: String 
    },
    PermissionDenied { path: String },
    IoError { message: String },
    Cancelled,
}

// Frontend can match on type:
// if (error.type === 'HashMismatch') {
//   console.log(`Hash failed: ${error.file}`)
// }

// ✅ GOOD - String for simple errors
pub fn simple_operation() -> Result<Data, String> {
    validate_input()
        .map_err(|e| format!("Validation failed: {}", e))?;
    
    Ok(data)
}
```

**Error guidelines:**
- Use typed enums when frontend needs to handle different error cases
- Use String for simple user-facing error messages
- Include context in error variants (file paths, values)
- Use `#[serde(tag = "type")]` for discriminated unions

---

## Complete Feature Example

**Types:**
```rust
// features/templates/types/mod.rs

#[derive(Debug, Clone, Serialize, Deserialize, Type)]
pub struct Template {
    pub id: String,
    pub name: String,
    pub levels: Vec<TemplateLevel>,
}

#[derive(Debug, Clone, Serialize, Deserialize, Type)]
pub struct TemplateLevel {
    pub pattern: String,
    pub fallback: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize, Type)]
#[serde(tag = "type")]
pub enum TemplateError {
    NotFound { id: String },
    ValidationError { message: String },
    IoError { message: String },
}
```

**Service:**
```rust
// features/templates/services/storage.rs

pub fn list_all_templates() -> Result<Vec<Template>, TemplateError> {
    let dir = get_templates_dir();
    
    let entries = std::fs::read_dir(dir)
        .map_err(|e| TemplateError::IoError { 
            message: e.to_string() 
        })?;
    
    let mut templates = Vec::new();
    
    for entry in entries {
        let entry = entry.map_err(|e| TemplateError::IoError { 
            message: e.to_string() 
        })?;
        
        let content = std::fs::read_to_string(entry.path())
            .map_err(|e| TemplateError::IoError { 
                message: e.to_string() 
            })?;
        
        let template: Template = serde_json::from_str(&content)
            .map_err(|e| TemplateError::IoError { 
                message: e.to_string() 
            })?;
        
        templates.push(template);
    }
    
    Ok(templates)
}
```

**Commands:**
```rust
// features/templates/commands/mod.rs

use crate::features::templates::{services, types::*};

#[command]
#[specta::specta]
pub fn list_templates() -> Result<Vec<Template>, TemplateError> {
    services::storage::list_all_templates()
}

#[command]
#[specta::specta]
pub fn save_template(template: Template) -> Result<(), TemplateError> {
    services::validator::validate_template(&template)?;
    services::storage::save_template(template)
}

#[command]
#[specta::specta]
pub fn delete_template(id: String) -> Result<(), TemplateError> {
    let path = services::storage::get_template_path(&id);
    
    if !path.exists() {
        return Err(TemplateError::NotFound { id });
    }
    
    std::fs::remove_file(path)
        .map_err(|e| TemplateError::IoError { 
            message: e.to_string() 
        })?;
    
    Ok(())
}
```

---

## Common Backend Mistakes

### ❌ Mistake 1: Missing specta attribute
```rust
// WRONG - No TypeScript bindings generated
#[command]
pub fn my_command() -> String { ... }

// RIGHT
#[command]
#[specta::specta]
pub fn my_command() -> String { ... }
```

### ❌ Mistake 2: Missing Type derive
```rust
// WRONG - Can't cross IPC boundary
#[derive(Serialize, Deserialize)]
pub struct MyData { ... }

// RIGHT
#[derive(Serialize, Deserialize, Type)]
pub struct MyData { ... }
```

### ❌ Mistake 3: Panic in command
```rust
// WRONG - Crashes app
#[command]
#[specta::specta]
pub fn load_file(path: String) -> String {
    std::fs::read_to_string(path).unwrap() // DON'T PANIC!
}

// RIGHT - Return Result
#[command]
#[specta::specta]
pub fn load_file(path: String) -> Result<String, String> {
    std::fs::read_to_string(path)
        .map_err(|e| e.to_string())
}
```

### ❌ Mistake 4: Using i64/u64
```rust
// WRONG - JavaScript can't handle 64-bit integers
#[derive(Serialize, Deserialize, Type)]
pub struct Stats {
    pub timestamp: i64, // specta will error
}

// RIGHT - Use f64 for timestamps
#[derive(Serialize, Deserialize, Type)]
pub struct Stats {
    pub timestamp: f64,
}
```

### ❌ Mistake 5: Heavy logic in command
```rust
// WRONG - Command does too much
#[command]
#[specta::specta]
pub fn process_data(data: Vec<Data>) -> Result<Output, String> {
    // 200 lines of business logic here...
}

// RIGHT - Delegate to service
#[command]
#[specta::specta]
pub fn process_data(data: Vec<Data>) -> Result<Output, String> {
    services::processor::process(data)
}
```

### ❌ Mistake 6: Not registering command
```rust
// WRONG - Command defined but not registered in bindings.rs
#[command]
#[specta::specta]
pub fn new_command() -> String { ... }

// RIGHT - Add to bindings.rs collect_commands![]
```

---

## Frontend Integration

**While you work on Rust features, here's what the frontend agent is doing:**

### Frontend Agent's Responsibilities

1. **Creating services** that wrap your commands
2. **Creating hooks** with TanStack Query for caching
3. **Building components** that consume typed data
4. **Listening to events** you emit
5. **Handling Result types** you return

### Integration Flow

**After you regenerate bindings:**
```bash
cargo test export_bindings -- --ignored
# OR
npm run tauri dev  # Auto-generates
```

**Frontend gets:**
- `src/lib/bindings.ts` with typed commands
- All your types as TypeScript interfaces
- `commands.yourCommand()` ready to call
- Result types: `{ status: 'ok', data: T } | { status: 'error', error: E }`

**Example frontend service (they write this):**
```typescript
// features/templates/services/templateService.ts
import { commands, unwrapResult } from '@/lib/tauri-bindings'

export async function listTemplates(): Promise<Template[]> {
  return unwrapResult(await commands.listTemplates())
}
```

**Example frontend hook (they write this):**
```typescript
// features/templates/hooks/useTemplates.ts
export function useTemplates() {
  return useQuery({
    queryKey: ['templates'],
    queryFn: listTemplates,
  })
}
```

### Coordination Points

**Event contracts:**
- You emit: `app.emit("processing:progress", payload)`
- They listen: `TauriEvents.listen('processing:progress', handler)`
- Coordinate event names and payload types upfront

**Error handling:**
- You return: `Result<T, ProcessingError>`
- They match: `if (result.status === 'error') { /* handle */ }`
- Use typed enums when they need to match on error type
- Use String for simple user messages

**Type contracts:**
- Your Rust types generate TypeScript interfaces
- Changes to Rust types require frontend updates
- Breaking changes need coordination
- Document types with `///` (becomes JSDoc)

### Shared Architectural Principles

**Feature parity:** Backend `src-tauri/src/features/my_feature/` mirrors frontend `src/features/my-feature/`

**Type flow:** Rust → tauri-specta → TypeScript (types flow one direction)

**Result pattern:** You return `Result<T, E>`, they handle both cases

**Event naming:** Use kebab-case with feature prefix: `processing:progress`, `template:saved`

**Parallel work:** After IPC contract agreed, work in parallel independently

---

## Binding Generation

**When to regenerate bindings:**
- After adding/changing commands
- After modifying types
- After changing error enums
- Before frontend agent needs new commands

**How to regenerate:**
```bash
# Option 1: Run test
cargo test export_bindings -- --ignored --nocapture

# Option 2: Start dev server (auto-generates)
npm run tauri dev

# Option 3: Explicit command
npm run rust:bindings
```

**Bindings output:**
- File: `src/lib/bindings.ts`
- Auto-generated (has `@ts-nocheck` header)
- NEVER edit manually
- Commit to version control

---

## Testing Commands

```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_save_template() {
        let template = Template {
            id: "test-123".to_string(),
            name: "Test Template".to_string(),
            levels: vec![],
        };
        
        let result = save_template(template);
        assert!(result.is_ok());
    }
    
    #[test]
    fn test_validation_error() {
        let invalid = Template {
            id: "".to_string(),
            name: "".to_string(),
            levels: vec![],
        };
        
        let result = save_template(invalid);
        assert!(result.is_err());
    }
}
```

**Run tests:**
```bash
cargo test
cargo test --package app --lib features::templates
```

---

## Quality Checks

```bash
cargo fmt              # Format code
cargo clippy           # Lint
cargo test             # Run tests
cargo build --release  # Check builds
```

**From project root:**
```bash
npm run rust:fmt       # Format
npm run rust:clippy    # Lint
npm run rust:test      # Test
npm run check:all      # Full quality gate
```

---

## Key References

- **Full architecture:** `docs/developer/architecture-guide.md`
- **tauri-specta docs:** https://github.com/specta-rs/tauri-specta
- **Example feature:** `src-tauri/src/features/preferences/`

---

## Backend Implementation Checklist

- ✅ Commands have both `#[command]` and `#[specta::specta]`
- ✅ All IPC types have `#[derive(Serialize, Deserialize, Type)]`
- ✅ Error enums use `#[serde(tag = "type")]`
- ✅ Commands return `Result<T, E>` (no panics)
- ✅ Commands registered in `bindings.rs`
- ✅ Bindings regenerated after changes
- ✅ Heavy logic in services (commands are thin)
- ✅ Events emitted for real-time updates
- ✅ Tests written for commands
- ✅ `cargo fmt` and `cargo clippy` pass
