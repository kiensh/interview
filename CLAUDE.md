# CLAUDE.md - Project Context for AI Sessions

## Project Overview

This is an **interview preparation repository** containing comprehensive Q&A documents for technical interviews. The content is tailored for **mid-level backend engineers (3+ years experience)**.

## Repository Structure

```
interview/
├── CLAUDE.md              # This file - context for AI sessions
├── README.md              # Main index linking to all topics
├── golang/                # Go language interview questions
│   ├── README.md          # Go section index
│   ├── go-basics.md       # Data types, Variables, Functions, Pointers
│   ├── go-concurrency.md  # Goroutines, Channels, Mutex, WaitGroup
│   ├── go-types-errors.md # Structs, Interfaces, Error handling
│   ├── go-stdlib-testing.md # Standard library, JSON, Testing
│   ├── go-advanced.md     # Modules, Reflection, Generics, Context
│   ├── patterns-performance.md # Design patterns, Profiling
│   ├── api-realtime.md    # REST, gRPC, WebSockets
│   ├── database.md        # SQL, NoSQL, GORM, sqlx
│   ├── messaging.md       # Kafka, RabbitMQ, Event-driven
│   ├── observability.md   # Logging, Metrics, Tracing
│   ├── gin-basics.md      # Gin routing, middleware
│   ├── gin-advanced.md    # Validation, graceful shutdown
│   ├── gin-features.md    # Templates, caching, sessions
│   └── gin-auth-testing.md # Auth, testing, deployment
├── system-design/         # System design interview questions
│   ├── README.md          # System design index
│   └── notification-system.md # Notification system design
├── java/                  # Java interview questions
├── database/              # Database interview questions
├── frontend/              # Frontend interview questions
├── python/                # Python interview questions
└── behavioral/            # Behavioral interview questions
```

## Document Format Conventions

All Q&A files follow this consistent format:

### 1. File Structure
```markdown
# Topic Title

## Table of Contents

### Section Name
- [Q1: Question title](#q1)
- [Q2: Question title](#q2)

---

## Section Name

<a id="q1"></a>
### Q1: Question title
**Answer:**

[Answer content with code examples]

<a id="q2"></a>
### Q2: Next question
**Answer:**

[Answer content]

---

[← Back to Index](README.md)
```

### 2. Code Examples
- Use fenced code blocks with language identifiers (```go, ```java, etc.)
- Use ```json for API request/response bodies containing JSON data
- Include practical, runnable code examples
- Add comments explaining key concepts
- Show both good and bad practices where relevant

### 3. Tables
- Use markdown tables for comparisons
- Common patterns: Feature comparisons, When to use X vs Y

### 4. Anchor Links
- Each question has an anchor: `<a id="q1"></a>`
- Table of contents links to anchors: `[Q1: Title](#q1)`

### 5. ASCII Art Diagrams
When creating system architecture diagrams using box-drawing characters:

**Box Width Consistency:**
- All lines within an enclosed box MUST have the same visual width
- Box-drawing characters (┌ ─ ┐ │ └ ┘ ┬ ┴ ├ ┤ ┼) count as 1 visual character each
- Use a monospace font to verify alignment

**Content Sizing:**
- Inner box content must fit within boundaries
- Standard inner box width: 11 characters (│ + 11 chars + │ = 13 total)
- If text is too long (e.g., "Notification" = 12 chars), abbreviate (e.g., "Notif")
- Words that commonly overflow: "Notification", "ClickHouse", "Email Worker"

**Box Types:**
```
Enclosed boxes (must have consistent width):
┌──────────────────────────────────────┐
│            SECTION TITLE             │
│  ┌─────────┐  ┌─────────┐            │
│  │ Inner 1 │  │ Inner 2 │            │
│  └─────────┘  └─────────┘            │
└──────────────────────────────────────┘

Flow diagrams (arrows may extend beyond boxes - OK):
┌─────────┐       ┌─────────┐
│  Box A  │──MSG─▶│  Box B  │──▶ Target
└─────────┘       └─────────┘
```

**Verification:**
- Use Unicode-aware width calculation (not byte count)
- Each line starting with │, ┌, or └ should have identical width
- Python verification script:

```python
import unicodedata

def visual_width(s):
    return sum(2 if unicodedata.east_asian_width(c) in ('F','W') else 1 for c in s)

# Check all diagram lines have consistent width
with open('your-file.md', 'r') as f:
    lines = f.readlines()

in_code_block = False
current_width = None

for i, line in enumerate(lines):
    line = line.rstrip('\n')
    if line.startswith('```'):
        in_code_block = not in_code_block
        current_width = None
        continue
    if in_code_block:
        if line.startswith('┌'):
            current_width = visual_width(line)
        elif line.startswith(('│', '└', '├')) and current_width:
            w = visual_width(line)
            if w != current_width:
                print(f'Line {i+1}: expected {current_width}, got {w}')
```

## Content Guidelines

### Target Audience
- Mid-level engineers (3+ years experience)
- Backend/full-stack developers
- Preparing for technical interviews

### Question Difficulty
- Focus on practical, real-world scenarios
- Include conceptual understanding AND implementation details
- Cover common interview topics AND production best practices

### Answer Style
- Start with a brief conceptual explanation
- Follow with code examples
- Include comparison tables where helpful
- Mention common pitfalls and best practices

## Technology Coverage

### Go (golang/)
- **Core**: Types, concurrency, interfaces, error handling
- **Advanced**: Generics, reflection, context, modules
- **Web**: REST APIs, gRPC, WebSockets
- **Data**: SQL (GORM, sqlx), NoSQL (Redis, MongoDB)
- **Infrastructure**: Kafka, RabbitMQ, observability
- **Framework**: Gin (comprehensive coverage)

### Java (java/) - Existing
- Core Java, Collections, Concurrency
- Spring Boot, JPA/Hibernate
- Testing, Design patterns

### Database (database/)
- SQL fundamentals, indexing, optimization
- PostgreSQL, MySQL specifics
- NoSQL concepts

### System Design (system-design/)
- Large-scale system architecture
- Notification systems, messaging, caching
- High availability, scalability patterns
- Includes detailed ASCII diagrams (follow diagram conventions above)

## How to Extend This Repository

### Adding a New Topic File
1. Create file in appropriate directory
2. Follow the standard format (see above)
3. Add entry to the directory's README.md
4. Update question counts in README files

### Adding a New Technology Section
1. Create new directory (e.g., `python/`)
2. Create README.md with index
3. Create topic files following conventions
4. Update main README.md

### Updating Existing Content
- Maintain consistent formatting
- Update Table of Contents if adding questions
- Keep code examples up-to-date with latest versions

## Common Tasks

### "Add more questions about X"
1. Find the relevant file
2. Add new questions following the format
3. Update Table of Contents
4. Update question count in README

### "Create questions for new technology Y"
1. Create new directory
2. Plan file structure (see golang/ as reference)
3. Create README.md index first
4. Create topic files with Q&A content

### "Update code examples to latest version"
1. Check current examples for deprecated APIs
2. Update with modern patterns
3. Add notes about version requirements if relevant

## Notes for AI Sessions

1. **Consistency is key** - Follow existing format exactly
2. **Code quality matters** - Examples should be production-ready
3. **Practical focus** - Real interview questions, not academic
4. **Cross-reference** - Link between related topics when helpful
5. **Keep updated** - Note version-specific information

## File Naming Conventions

- Lowercase with hyphens: `go-basics.md`, `gin-basics.md`
- Descriptive but concise
- Prefixes for grouping:
  - `go-*.md` - Go language core topics
  - `gin-*.md` - Gin framework topics
  - No prefix - Backend/development topics (database, messaging, etc.)

## Estimated Content

| Directory | Files | Questions |
|-----------|-------|-----------|
| golang/ | 14 + README | ~155 |
| system-design/ | 1 + README | ~20 |
| java/ | 9 + README | varies |
| database/ | 5 + README | varies |
| frontend/ | 5 + README | varies |
| python/ | 5 + README | varies |
| behavioral/ | 1 + README | varies |

---

*Last updated: January 2026*
