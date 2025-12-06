# C# style guide
This document outlines the styling and coding conventions used for C#.

## General
- Use UTF-8 encoding
- Use 4 spaces for indentation
- Use file-scoped namespaces
- Always set access modifiers explicitly
- Use nullable reference types

## File structure
Each file should follow this structure:
1. Namespace declaration (file-scoped)
2. Usings (grouped: System, then external packages, then local)
3. Code (types, classes, etc.)

All public types must have their own file. Private types can be within the file where they are used.
```csharp
namespace MyProject.Services;

using System;
using System.IO;
using System.Threading.Tasks;
using Microsoft.Extensions.Logging;
using MyProject.Configuration;
using MyProject.Utils;
using MyProject.Models;

public class FileService(ILogger<FileService> logger)
{
    // Private helper type only used in this class
    private record ValidationResult
    {
        public required bool IsValid { get; set; }

        public required string Error { get; set; }
    }

    public async Task<(string? content, string? error)> ReadFileAsync(string path)
    {
        // Implementation
    }
}
```

### Access modifiers
Always use explicit access modifiers on all members. Use only `public` or `private`. Avoid `internal` and `protected`.

All classes should use primary constructors.

```csharp
public class FileService(ILogger<FileService> logger, IConfiguration config)
{
    public async Task<(string? content, string? error)> ReadFileAsync(string path)
    {
        logger.LogInformation("Reading file from {Path}", path);
        // Implementation
    }

    private static bool ValidatePath(string path)
    {
        // Helper
    }
}
```

### Methods
Use `async/await` syntax for asynchronous operations. Always include explicit access modifiers.

### Control Flow
- Simplify control logic using guard clauses (early returns)
- Always use curly braces for blocks, even single-statement blocks
- Exit early for error conditions

### Error Handling
Use tuple-based error handling for expected error cases. Reserve exceptions for truly exceptional situations.

```csharp
// Preferred: Tuple-based error handling
public async Task<(string? content, string? error)> ReadFileAsync(string filePath)
{
    if (string.IsNullOrEmpty(filePath))
    {
        return (null, "File path cannot be empty");
    }

    if (!File.Exists(filePath))
    {
        return (null, $"File not found at \"{filePath}\"");
    }

    try
    {
        var content = await File.ReadAllTextAsync(filePath);
        return (content, null);
    }
    catch (Exception ex)
    {
        return (null, $"Failed to read file: {ex.Message}");
    }
}

// Use exceptions only for truly exceptional cases
public void ValidateConfiguration(Configuration config)
{
    if (config is null)
    {
        throw new InvalidOperationException("Configuration does not exist. Ensure configuration is loaded before use.");
    }
}
```

Error messages should be descriptive, explaining what went wrong and ideally providing context about why.

### Null checking
Use `is null` and `is not null` patterns whenever possible:

```csharp
// Good
if (value is null)
{
    return;
}

// Exception: LINQ and lambdas where pattern matching isn't allowed
var items = collection.Where(x => x != null);
```

## Comments
No documentation comments. Use comments only when absolutely needed to explain **why** code is written a certain way, never **what** it does. If the code needs comments to explain what it does, refactor it to be clearer.

## Variable declarations
Use `var` for all local variables and name methods clearly to make the type obvious.
Use the concrete type in variable declarations only if needed for program correctness.

## Types and properties
Use records for DTOs and parameter objects. Use classes for entities with behavior.
All properties should use `{ get; set; }` unless read-only behavior is crucial. Use `required` keyword for mandatory fields instead of providing default values.
All fields and properties must have a blank line between them.

## Design patterns
Prefer **functional and structural patterns** over OOP:
- Use static classes and methods for stateless logic
- Use parameter records when functions need many parameters
- Use pure functions where possible
- Organize code by feature/domain, not by technical layer

Static class names should use passive, namespace-like naming (e.g., `FileProcessing`, `PathValidation`) rather than "doer" names (avoid `FileProcessor`, `PathValidator`).

```csharp
// Stateless logic - static class with namespace-like naming
public static class PathValidation
{
    public static string SanitizePath(string path)
    {
        // Implementation
    }

    public static bool IsPathWithinRoot(string path, string root)
    {
        // Implementation
    }
}

// Many parameters - use parameter record
public record FileProcessingOptions
{
    public required string InputPath { get; set; }

    public required string OutputPath { get; set; }

    public bool Overwrite { get; set; }

    public int Timeout { get; set; }
}

public static class FileProcessing
{
    public static async Task<(bool success, string? error)> ProcessAsync(
        FileProcessingOptions options)
    {
        // Implementation
    }
}
```

### Module organization
- Everything is either `public` or `private`. Use `public` where you would use `internal`.
- Public interfaces for stateful services
- Stateless logic goes in static classes that accept stateful service interfaces as parameters

```csharp
// Public interface for stateful service
public interface IFileService
{
    Task<(string? content, string? error)> ReadFileAsync(string path);
}

// Public implementation with primary constructor
public class FileService(ILogger<FileService> logger) : IFileService
{
    public async Task<(string? content, string? error)> ReadFileAsync(string path)
    {
        var (isValid, error) = PathValidation.ValidatePath(path);
        if (!isValid)
        {
            return (null, error);
        }

        // Implementation
    }
}

// Stateless logic that accepts stateful services
public static class FileOperations
{
    public static async Task<(bool success, string? error)> CopyFileAsync(
        string source,
        string destination,
        IFileService fileService)
    {
        // Implementation uses fileService parameter
    }
}
```

## Controllers
Always use traditional controller pattern with primary constructors.
Apply `[Authorize]` attribute at controller level, then use `[AllowAnonymous]` on specific endpoints that don't require authorization.
Use explicit return types instead of `IActionResult`.
```csharp
[ApiController]
[Route("api/[controller]")]
[Authorize]
public class FilesController(IFileService fileService, ILogger<FilesController> logger) : ControllerBase
{
    [HttpPost("{project}")]
    public async Task<ActionResult<ProcessFileResponse>> ProcessFile(string project, [FromBody] FileRequest request)
    {
        var (result, error) = await fileService.ProcessAsync(project, request);
        if (error is not null)
        {
            return BadRequest(new { error });
        }

        return Ok(result);
    }

    [HttpGet("health")]
    [AllowAnonymous]
    public ActionResult<HealthResponse> Health()
    {
        return Ok(new HealthResponse { Status = "healthy" });
    }
}
```

## Naming conventions
- **Files**: PascalCase matching the primary type (`FileService.cs`, `PathValidation.cs`)
- **Methods/Properties**: PascalCase (`ResolvedPath`, `MakeApiRequest`)
- **Local variables/parameters**: camelCase (`resolvedPath`, `apiRequest`)
- **Private fields**: Avoid by using functional/structural patterns, but use camelCase if needed (`logger`, `configuration`)
- **Interfaces**: PascalCase with `I` prefix (`IFileService`, `IApiClient`)
- **Types/Classes**: PascalCase (`FileEntry`, `ApiResponse`)
- **Static classes**: Passive namespace-like naming (`FileProcessing`, `PathValidation` - not `FileProcessor`, `PathValidator`)
- **Constants**: PascalCase (`ApiKey`, `ServiceUrl`)
- **Async methods**: Suffix with `Async` (`ReadFileAsync`, `ProcessDataAsync`)

## Formatting
- Max line length: Aim for 120 characters, but prioritize readability
- Blank lines: Use to separate logical sections and between all properties/fields
- Trailing commas: Always use in multi-line initializers
- Use expression-bodied members for simple properties and methods

## Pattern matching
Use pattern matching where possible to document all decision paths explicitly.
```csharp
public string GetStatus(int code) => code switch
{
    200 => "OK",
    404 => "Not Found",
    500 => "Server Error",
    _ => "Unknown",
};
```

## Required properties
Use `required` keyword instead of nullable types with default values:
```csharp
public record Configuration
{
    public required string ApiKey { get; set; }

    public required int Timeout { get; set; }
}
```