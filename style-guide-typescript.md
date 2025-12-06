# TypeScript style guide
This document outlines the styling and coding conventions used for TypeScript.

## General
- Use UTF-8 encoding
- Use 4 spaces for indentation
- Always use semicolons
- Always use double quotes instead of single quotes

## File structure
Each file should follow this structure:
1. Imports (grouped: external packages, then local modules, then types)
2. Type definitions (if not in separate file)
3. Code (functions, constants, etc.)

All public types should have their own file when they are substantial. Helper types can be within the file where they are used.

```typescript
import express from "express";
import cors from "cors";
import * as fs from "fs/promises";
import * as path from "path";
import { configLoader } from "./config";
import { logger } from "./utils/logger";
import { FileEntry, ReadFileOptions } from "../types/api";

// Private helper type only used in this file
interface ValidationResult {
    readonly isValid: boolean;
    readonly error: string;
}

export async function readFileAsync(
    projectPath: string,
    filePath: string,
    options?: ReadFileOptions
): Promise<[Optional<string>, Error]> {
    // Implementation
}
```

## Imports
Group imports by type, in order: external packages, then local modules, then types.

```typescript
import express from "express";
import cors from "cors";
import * as fs from "fs/promises";
import * as path from "path";
import { configLoader } from "./config";
import { logger } from "./utils/logger";
import { FileEntry, ReadFileOptions } from "../types/api";
```

## Functions
Use `async/await` syntax for asynchronous operations. Always use proper function declarations for top-level function definitions:

```typescript
export async function readFileAsync(
    projectPath: string,
    filePath: string,
    options?: ReadFileOptions
): Promise<[Optional<string>, Error]> {
    // Implementation
}
```

## Control flow
- Simplify control logic using guard clauses (early returns)
- Always use curly braces for blocks, even single-statement blocks
- Exit early for error conditions

```typescript
export async function processFileAsync(path: string): Promise<[Optional<void>, Error]> {
    if (!path) {
        return [null, "Path is required"];
    }

    if (!await fileExistsAsync(path)) {
        return [null, "File not found"];
    }

    // Main logic here
    return [null, null];
}

// Always use braces
if (condition) {
    return;
}
```

## Error handling
Use tuple-based error handling for expected error cases. Reserve exceptions for truly exceptional situations.

Define common error handling types in your project (typically in a shared types file):
```typescript
// These should be defined once in your project's shared types
type Error = string | null;
type Optional<T> = T | null;
```

Then use these types consistently with the order always being `[value, error]`:
```typescript
// Preferred: Tuple-based error handling
export async function readFileAsync(filePath: string): Promise<[Optional<string>, Error]> {
    if (!filePath) {
        return [null, "File path cannot be empty"];
    }

    if (!await fileExistsAsync(filePath)) {
        return [null, `File not found at "${filePath}"`];
    }

    try {
        const content = await fs.readFile(filePath, "utf-8");
        return [content, null];
    } catch (error) {
        return [null, `Failed to read file: ${error instanceof Error ? error.message : "Unknown error"}`];
    }
}

// Use exceptions only for truly exceptional cases
export function validateConfiguration(config: Configuration): void {
    if (!config) {
        throw new Error("Configuration does not exist. Ensure configuration is loaded before use.");
    }
}

// Destructuring is always consistent: [value, error]
const [content, error] = await readFileAsync(filePath);
if (error !== null) {
    console.error(error);
    return;
}
// Use content safely here
```

Error messages should be descriptive, explaining what went wrong and ideally providing context about why.

## Null checking
Prefer `null` over `undefined` for explicit absence of values. Use `undefined` only for optional parameters and uninitialized state.

Use explicit null checks with clarity:

```typescript
// Good - explicit null check
if (value === null) {
    return;
}

// Good - checking for both null and undefined when needed
if (value == null) {
    return;
}

// For non-null assertions, be explicit
if (value !== null) {
    // Use value safely
}

// Return null for absent values, not undefined
export function findUserById(id: string): User | null {
    const user = users.find(u => u.id === id);
    return user ?? null;
}
```

## Comments
No documentation comments. Use comments only when absolutely needed to explain **why** code is written a certain way, never **what** it does. If the code needs comments to explain what it does, refactor it to be clearer.

## Variable declarations
Use `const` by default, `let` when reassignment is needed. Never use `var`.
Name functions clearly to make the type obvious from context.

## Types and interfaces
Use interfaces for object shapes and DTOs. Use type aliases for unions, intersections, and utility types.
All properties should be `readonly` unless mutation is absolutely necessary for correctness.
Prefer required properties over optional with defaults.

```typescript
// Good - readonly properties by default
interface Configuration {
    readonly apiKey: string;
    readonly timeout: number;
    readonly retries: number;
}

// Only use mutable properties when mutation is necessary
interface MutableState {
    count: number;
    isActive: boolean;
}

// Avoid - optional with defaults handled elsewhere
interface Configuration {
    apiKey?: string;
    timeout?: number;
    retries?: number;
}
```

## Design patterns
Prefer **functional and structural patterns** over OOP:
- Use pure functions where possible
- Use parameter objects when functions need many parameters
- Organize code by feature/domain, not by technical layer
- Avoid classes unless managing stateful resources

```typescript
// Many parameters - use parameter object
interface FileProcessingOptions {
    readonly inputPath: string;
    readonly outputPath: string;
    readonly overwrite: boolean;
    readonly timeout: number;
}

export async function processFileAsync(
    options: FileProcessingOptions
): Promise<[Optional<boolean>, Error]> {
    // Implementation
}

// Stateless logic as pure functions
export function sanitizePath(path: string): string {
    // Implementation
}

export function isPathWithinRoot(path: string, root: string): boolean {
    // Implementation
}
```

## Module organization
- Export only what needs to be public
- Use named exports, avoid default exports
- Group related functionality together
- Stateless logic goes in exported functions

```typescript
// Good - named exports of related functionality
export function sanitizePath(path: string): string { }
export function isPathWithinRoot(path: string, root: string): boolean { }
export function resolvePath(base: string, relative: string): string { }

// Avoid
export default class PathValidator { }
```

## Interfacing with servers
Isolate all server communication in dedicated client files within a single directory. This contains the API surface to one place in the codebase.

Create a `clients/` directory with files named `[Entity]Client.ts` for each logical grouping of endpoints:

```
src/
  clients/
    UserClient.ts
    ProjectClient.ts
    FileClient.ts
```

Each client file should:
- Export functions for each endpoint
- Use tuple-based error handling
- Handle request/response transformation
- Contain all type definitions related to that API

```typescript
// clients/UserClient.ts
import { Optional, Error } from "../types/common";

interface UserResponse {
    readonly id: string;
    readonly name: string;
    readonly email: string;
}

interface CreateUserRequest {
    readonly name: string;
    readonly email: string;
}

export async function getUserAsync(userId: string): Promise<[Optional<UserResponse>, Error]> {
    try {
        const response = await fetch(`/api/users/${userId}`);

        if (!response.ok) {
            return [null, `Failed to fetch user: ${response.statusText}`];
        }

        const data = await response.json();
        return [data, null];
    } catch (error) {
        return [null, `Network error: ${error instanceof Error ? error.message : "Unknown error"}`];
    }
}

export async function createUserAsync(request: CreateUserRequest): Promise<[Optional<UserResponse>, Error]> {
    try {
        const response = await fetch("/api/users", {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify(request),
        });

        if (!response.ok) {
            return [null, `Failed to create user: ${response.statusText}`];
        }

        const data = await response.json();
        return [data, null];
    } catch (error) {
        return [null, `Network error: ${error instanceof Error ? error.message : "Unknown error"}`];
    }
}
```

This pattern:
- Makes it easy to find all server interactions
- Enables consistent error handling across all API calls
- Simplifies mocking for tests
- Creates a clear contract between frontend and backend

## Express middleware and routes
Use the async handler pattern for Express routes.
Always add auth middleware, unless authentication must be disabled for the given endpoint.
```typescript
router.post("/files/:project", authMiddleware, asyncHandler(async (req, res) => {
    const [result, error] = await fileService.processFileAsync(req.params.project, req.body);

    if (error !== null) {
        return res.status(400).json({ error });
    }

    return res.json(result);
}));
```

## Naming conventions
- **Files**: use PascalCase for all file names (`FileService.ts`, `PathValidation.ts`)
- **Functions/Variables**: use camelCase (`resolvedPath`, `makeApiRequest`, `processFile`)
- **Types/Interfaces**: use PascalCase, prefix interfaces with `I` (`FileEntry`, `IApiResponse`)
- **Constants**: use PascalCase for exported constants (`ApiKey`, `ServiceUrl`)
- **Module-level constants**: use UPPER_SNAKE_CASE only for true compile-time constants (`MAX_RETRIES`, `DEFAULT_TIMEOUT`)
- **Async functions**: Always suffix with `Async` (`readFileAsync`, `processDataAsync`, `fetchUserAsync`)

## Formatting
- Max line length: Aim for 120 characters, but prioritize readability
- Blank lines: Use to separate logical sections
- Trailing commas: Always use in multi-line objects and arrays
- Use arrow functions for callbacks and simple expressions

```typescript
const config = {
    status: "healthy",
    timestamp: new Date().toISOString(),
    version: "1.0.0",
    environment: config.nodeEnv,
};

// Arrow functions for callbacks
const filtered = items.filter(item => item.isActive);

// Function declarations for top-level functions
export async function processItemsAsync(items: Item[]): Promise<[Optional<void>, Error]> {
    // Implementation
}
```

## Pattern matching
Use discriminated unions and exhaustive type narrowing to document all decision paths explicitly. This is TypeScript's equivalent to pattern matching and should be preferred for state machines, result types, and complex conditional logic.

```typescript
// Discriminated unions for result types
type Result<T> =
    | { readonly type: "success"; readonly data: T }
    | { readonly type: "error"; readonly message: string }
    | { readonly type: "pending" };

export function handleResultAsync<T>(result: Result<T>): void {
    switch (result.type) {
        case "success":
            console.log(result.data);
            break;
        case "error":
            console.error(result.message);
            break;
        case "pending":
            console.log("Waiting...");
            break;
    }
}

// Discriminated unions for state machines
type RequestState =
    | { readonly status: "idle" }
    | { readonly status: "loading"; readonly startedAt: Date }
    | { readonly status: "success"; readonly data: string; readonly completedAt: Date }
    | { readonly status: "failure"; readonly error: string; readonly failedAt: Date };

export function getStatusMessage(state: RequestState): string {
    switch (state.status) {
        case "idle":
            return "Ready to start";
        case "loading":
            return `Loading since ${state.startedAt.toISOString()}`;
        case "success":
            return `Completed: ${state.data}`;
        case "failure":
            return `Failed: ${state.error}`;
    }
}

// Simple switch statements for enums or literals
export function getStatusCode(code: number): string {
    switch (code) {
        case 200:
            return "OK";
        case 404:
            return "Not Found";
        case 500:
            return "Server Error";
        default:
            return "Unknown";
    }
}
```

## Types
- Prefer interfaces for object shapes
- Use type aliases for unions and utility types
- Avoid `any`, use `unknown` if type is truly unknown
- Use discriminated unions for state machines and result types