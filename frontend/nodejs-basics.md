# Node.js Basics Interview Questions

## Table of Contents

### Core Concepts
- [Q1: What is Node.js and how does it work?](#q1)
- [Q2: Explain the Node.js event loop](#q2)
- [Q3: What is the difference between process.nextTick() and setImmediate()?](#q3)
- [Q4: How does Node.js handle modules (CommonJS vs ESM)?](#q4)

### Express.js
- [Q5: What is Express.js and how do you create a basic server?](#q5)
- [Q6: What is middleware in Express?](#q6)
- [Q7: How do you handle routing in Express?](#q7)
- [Q8: How do you handle errors in Express?](#q8)

### Async Patterns
- [Q9: How do you handle asynchronous operations in Node.js?](#q9)
- [Q10: What are Streams in Node.js?](#q10)
- [Q11: Explain the Buffer class](#q11)

### Advanced Topics
- [Q12: How does Node.js handle child processes?](#q12)
- [Q13: What is clustering in Node.js?](#q13)
- [Q14: How do you handle environment variables?](#q14)
- [Q15: What are common security practices in Node.js?](#q15)

---

## Core Concepts

<a id="q1"></a>
### Q1: What is Node.js and how does it work?
**Answer:**

Node.js is a JavaScript runtime built on Chrome's V8 engine, designed for building scalable network applications.

**Key characteristics:**
| Feature | Description |
|---------|-------------|
| Single-threaded | Main thread handles JavaScript execution |
| Event-driven | Uses events and callbacks for async operations |
| Non-blocking I/O | Operations don't block the main thread |
| Cross-platform | Runs on Windows, Linux, macOS |

```javascript
// Basic Node.js server
const http = require('http');

const server = http.createServer((req, res) => {
    res.writeHead(200, { 'Content-Type': 'text/plain' });
    res.end('Hello, World!');
});

server.listen(3000, () => {
    console.log('Server running on port 3000');
});

// Node.js architecture:
// 1. V8 Engine - Executes JavaScript
// 2. libuv - Handles async I/O, event loop
// 3. Node.js bindings - Connect JS to C++ libraries
// 4. Node.js API - Built-in modules

// Global objects
console.log(__dirname);  // Current directory path
console.log(__filename); // Current file path
console.log(process.env.NODE_ENV); // Environment variables
console.log(process.argv); // Command line arguments

// Process object
process.exit(0); // Exit with success code
process.exit(1); // Exit with error code

process.on('uncaughtException', (err) => {
    console.error('Uncaught exception:', err);
    process.exit(1);
});

process.on('unhandledRejection', (reason, promise) => {
    console.error('Unhandled rejection:', reason);
});

// Check Node.js version
console.log(process.version); // v18.x.x

// Memory usage
console.log(process.memoryUsage());
// { rss, heapTotal, heapUsed, external, arrayBuffers }
```

<a id="q2"></a>
### Q2: Explain the Node.js event loop
**Answer:**

The event loop is what allows Node.js to perform non-blocking I/O operations.

```
   ┌───────────────────────────┐
┌─>│           timers          │  setTimeout, setInterval
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  I/O callbacks deferred
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  Internal use only
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           poll            │  Retrieve new I/O events
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           check           │  setImmediate callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │  socket.on('close', ...)
   └───────────────────────────┘
```

```javascript
// Event loop phases demonstration

console.log('Start'); // 1

setTimeout(() => {
    console.log('setTimeout 1'); // 5
}, 0);

setImmediate(() => {
    console.log('setImmediate'); // 6
});

Promise.resolve().then(() => {
    console.log('Promise 1'); // 3
}).then(() => {
    console.log('Promise 2'); // 4
});

process.nextTick(() => {
    console.log('nextTick'); // 2
});

console.log('End'); // 1 (continued)

// Output order:
// Start
// End
// nextTick
// Promise 1
// Promise 2
// setTimeout 1
// setImmediate

// Priority:
// 1. Synchronous code
// 2. process.nextTick (microtask queue)
// 3. Promises (microtask queue)
// 4. setTimeout/setInterval (timers phase)
// 5. setImmediate (check phase)

// I/O example
const fs = require('fs');

fs.readFile('file.txt', () => {
    setTimeout(() => console.log('timeout'), 0);
    setImmediate(() => console.log('immediate'));
});
// Inside I/O callback, setImmediate always executes first
// Output: immediate, timeout
```

<a id="q3"></a>
### Q3: What is the difference between process.nextTick() and setImmediate()?
**Answer:**

| Feature | process.nextTick() | setImmediate() |
|---------|-------------------|----------------|
| Execution | Before any I/O | After I/O events |
| Phase | After current operation | Check phase |
| Priority | Higher (runs first) | Lower |
| Recursive | Can starve I/O | Won't starve I/O |

```javascript
// process.nextTick - executes immediately after current operation
process.nextTick(() => {
    console.log('nextTick');
});
console.log('current operation');
// Output: current operation, nextTick

// setImmediate - executes in next iteration of event loop
setImmediate(() => {
    console.log('immediate');
});
console.log('current operation');
// Output: current operation, immediate

// In I/O callbacks, setImmediate executes before setTimeout(0)
const fs = require('fs');

fs.readFile('file.txt', () => {
    setImmediate(() => console.log('immediate'));
    setTimeout(() => console.log('timeout'), 0);
});
// Output: immediate, timeout (setImmediate first in I/O callback)

// Outside I/O, order is non-deterministic
setImmediate(() => console.log('immediate'));
setTimeout(() => console.log('timeout'), 0);
// Could be either order

// Recursive nextTick can starve I/O
function recursiveNextTick() {
    process.nextTick(recursiveNextTick); // BAD - I/O never executes
}

// Safe recursive pattern
function recursiveImmediate() {
    setImmediate(recursiveImmediate); // OK - I/O can still execute
}

// Use cases:
// process.nextTick:
// - Ensure callback runs after current code but before I/O
// - Allow users to assign event handlers after constructor
// - Emit events after constructor completes

class EventEmitterExample extends EventEmitter {
    constructor() {
        super();
        // Emit after constructor completes
        process.nextTick(() => {
            this.emit('ready');
        });
    }
}

const emitter = new EventEmitterExample();
emitter.on('ready', () => console.log('Ready!'));

// setImmediate:
// - Schedule code to run after I/O events
// - Break up CPU-intensive operations
function processLargeArray(arr, callback) {
    const chunk = arr.splice(0, 1000);
    // Process chunk...
    
    if (arr.length > 0) {
        setImmediate(() => processLargeArray(arr, callback));
    } else {
        callback();
    }
}
```

<a id="q4"></a>
### Q4: How does Node.js handle modules (CommonJS vs ESM)?
**Answer:**

| Feature | CommonJS (CJS) | ES Modules (ESM) |
|---------|----------------|------------------|
| Syntax | `require()` / `module.exports` | `import` / `export` |
| Loading | Synchronous | Asynchronous |
| File extension | `.js`, `.cjs` | `.mjs` or `"type": "module"` |
| Top-level await | No | Yes |
| Dynamic import | `require()` | `import()` |

```javascript
// CommonJS (traditional Node.js)

// Exporting
// math.js
const add = (a, b) => a + b;
const subtract = (a, b) => a - b;

module.exports = { add, subtract };
// Or:
module.exports.add = add;
// Or:
exports.add = add; // exports is alias for module.exports

// Importing
const math = require('./math');
const { add, subtract } = require('./math');

// ES Modules

// Exporting
// math.mjs (or .js with "type": "module" in package.json)
export const add = (a, b) => a + b;
export const subtract = (a, b) => a - b;
export default class Calculator {}

// Importing
import Calculator, { add, subtract } from './math.mjs';
import * as math from './math.mjs';

// Enable ESM in package.json
{
    "type": "module"
}

// Or use .mjs extension

// Dynamic import (works in both)
async function loadModule() {
    const module = await import('./dynamic-module.js');
    module.doSomething();
}

// Top-level await (ESM only)
const data = await fetchData();

// Differences in behavior

// __dirname and __filename in ESM
import { fileURLToPath } from 'url';
import { dirname } from 'path';

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);

// Importing JSON
// CommonJS
const config = require('./config.json');

// ESM (Node.js 17.5+)
import config from './config.json' assert { type: 'json' };

// Or use createRequire
import { createRequire } from 'module';
const require = createRequire(import.meta.url);
const config = require('./config.json');

// Circular dependencies
// CommonJS - returns incomplete exports
// ESM - works with live bindings

// Interoperability
// ESM can import CJS
import cjsModule from './cjs-module.cjs';

// CJS can import ESM with dynamic import
async function main() {
    const esmModule = await import('./esm-module.mjs');
}
```

---

## Express.js

<a id="q5"></a>
### Q5: What is Express.js and how do you create a basic server?
**Answer:**

Express.js is a minimal, flexible Node.js web application framework.

```javascript
const express = require('express');
const app = express();

// Middleware for parsing JSON
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Basic route
app.get('/', (req, res) => {
    res.send('Hello, World!');
});

// Route with parameter
app.get('/users/:id', (req, res) => {
    const { id } = req.params;
    res.json({ userId: id });
});

// Route with query string
app.get('/search', (req, res) => {
    const { q, page } = req.query;
    res.json({ query: q, page });
});

// POST request
app.post('/users', (req, res) => {
    const { name, email } = req.body;
    // Create user...
    res.status(201).json({ id: 1, name, email });
});

// Multiple handlers
app.get('/example',
    (req, res, next) => {
        console.log('First handler');
        next();
    },
    (req, res) => {
        res.send('Second handler');
    }
);

// Start server
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
});

// Response methods
app.get('/responses', (req, res) => {
    // res.send('text');           // Send text
    // res.json({ key: 'value' }); // Send JSON
    // res.status(404).send('Not found');
    // res.redirect('/other');
    // res.render('template', { data }); // Render view
    // res.sendFile('/path/to/file');
    // res.download('/path/to/file');
    // res.set('Header', 'value');
    // res.cookie('name', 'value');
});

// Request object
app.get('/request', (req, res) => {
    req.params;   // URL parameters
    req.query;    // Query string
    req.body;     // Request body
    req.headers;  // HTTP headers
    req.cookies;  // Cookies (with cookie-parser)
    req.ip;       // Client IP
    req.method;   // HTTP method
    req.path;     // URL path
    req.protocol; // http or https
    req.secure;   // Is HTTPS?
    req.hostname; // Host name
});
```

<a id="q6"></a>
### Q6: What is middleware in Express?
**Answer:**

Middleware functions have access to request, response, and next middleware in the cycle.

```javascript
const express = require('express');
const app = express();

// Middleware signature
// (req, res, next) => { ... }

// Application-level middleware
app.use((req, res, next) => {
    console.log(`${req.method} ${req.path}`);
    next(); // Pass to next middleware
});

// Middleware with path
app.use('/api', (req, res, next) => {
    console.log('API request');
    next();
});

// Router-level middleware
const router = express.Router();
router.use((req, res, next) => {
    console.log('Router middleware');
    next();
});

// Built-in middleware
app.use(express.json()); // Parse JSON body
app.use(express.urlencoded({ extended: true })); // Parse URL-encoded
app.use(express.static('public')); // Serve static files

// Third-party middleware
const cors = require('cors');
const helmet = require('helmet');
const morgan = require('morgan');

app.use(cors()); // Enable CORS
app.use(helmet()); // Security headers
app.use(morgan('dev')); // Request logging

// Custom middleware examples

// Authentication middleware
const authenticate = (req, res, next) => {
    const token = req.headers.authorization?.split(' ')[1];
    
    if (!token) {
        return res.status(401).json({ error: 'No token provided' });
    }
    
    try {
        const decoded = jwt.verify(token, process.env.JWT_SECRET);
        req.user = decoded;
        next();
    } catch (error) {
        res.status(401).json({ error: 'Invalid token' });
    }
};

app.get('/protected', authenticate, (req, res) => {
    res.json({ user: req.user });
});

// Request timing middleware
const timing = (req, res, next) => {
    const start = Date.now();
    
    res.on('finish', () => {
        const duration = Date.now() - start;
        console.log(`${req.method} ${req.path} - ${duration}ms`);
    });
    
    next();
};

// Validation middleware
const validateUser = (req, res, next) => {
    const { name, email } = req.body;
    
    if (!name || !email) {
        return res.status(400).json({ error: 'Name and email required' });
    }
    
    if (!email.includes('@')) {
        return res.status(400).json({ error: 'Invalid email' });
    }
    
    next();
};

app.post('/users', validateUser, (req, res) => {
    // Create user...
});

// Error-handling middleware (4 parameters)
app.use((err, req, res, next) => {
    console.error(err.stack);
    res.status(500).json({ error: 'Something went wrong' });
});
```

<a id="q7"></a>
### Q7: How do you handle routing in Express?
**Answer:**

```javascript
const express = require('express');
const app = express();

// Basic routes
app.get('/', (req, res) => res.send('GET /'));
app.post('/', (req, res) => res.send('POST /'));
app.put('/', (req, res) => res.send('PUT /'));
app.delete('/', (req, res) => res.send('DELETE /'));
app.patch('/', (req, res) => res.send('PATCH /'));

// All HTTP methods
app.all('/all', (req, res) => res.send(`${req.method} /all`));

// Route parameters
app.get('/users/:id', (req, res) => {
    res.json({ id: req.params.id });
});

// Multiple parameters
app.get('/users/:userId/posts/:postId', (req, res) => {
    const { userId, postId } = req.params;
    res.json({ userId, postId });
});

// Optional parameter
app.get('/users/:id?', (req, res) => {
    const id = req.params.id || 'all';
    res.send(`User: ${id}`);
});

// Regex in route
app.get('/user/:id(\\d+)', (req, res) => {
    // Only matches numeric id
    res.send(`User ID: ${req.params.id}`);
});

// Route handlers
app.route('/articles')
    .get((req, res) => res.send('Get all articles'))
    .post((req, res) => res.send('Create article'))
    .put((req, res) => res.send('Update article'));

// Express Router (modular routing)
// routes/users.js
const router = express.Router();

router.get('/', (req, res) => {
    res.json({ users: [] });
});

router.get('/:id', (req, res) => {
    res.json({ user: { id: req.params.id } });
});

router.post('/', (req, res) => {
    res.status(201).json({ user: req.body });
});

router.put('/:id', (req, res) => {
    res.json({ user: { id: req.params.id, ...req.body } });
});

router.delete('/:id', (req, res) => {
    res.status(204).send();
});

module.exports = router;

// app.js
const usersRouter = require('./routes/users');
const postsRouter = require('./routes/posts');

app.use('/api/users', usersRouter);
app.use('/api/posts', postsRouter);

// Nested routers
const commentsRouter = express.Router({ mergeParams: true });

commentsRouter.get('/', (req, res) => {
    // Access parent params with mergeParams: true
    res.json({ postId: req.params.postId, comments: [] });
});

postsRouter.use('/:postId/comments', commentsRouter);
// GET /api/posts/:postId/comments

// 404 handler (after all routes)
app.use((req, res) => {
    res.status(404).json({ error: 'Not found' });
});
```

<a id="q8"></a>
### Q8: How do you handle errors in Express?
**Answer:**

```javascript
const express = require('express');
const app = express();

// Synchronous errors are caught automatically
app.get('/sync-error', (req, res) => {
    throw new Error('Sync error'); // Caught by error handler
});

// Async errors need to be passed to next()
app.get('/async-error', async (req, res, next) => {
    try {
        await someAsyncOperation();
    } catch (error) {
        next(error); // Pass to error handler
    }
});

// Async wrapper to avoid try-catch
const asyncHandler = (fn) => (req, res, next) => {
    Promise.resolve(fn(req, res, next)).catch(next);
};

app.get('/async', asyncHandler(async (req, res) => {
    const data = await fetchData();
    res.json(data);
}));

// Or use express-async-handler package
const asyncHandler = require('express-async-handler');

// Custom error class
class AppError extends Error {
    constructor(message, statusCode) {
        super(message);
        this.statusCode = statusCode;
        this.isOperational = true;
        
        Error.captureStackTrace(this, this.constructor);
    }
}

class NotFoundError extends AppError {
    constructor(message = 'Resource not found') {
        super(message, 404);
    }
}

class ValidationError extends AppError {
    constructor(message = 'Validation failed') {
        super(message, 400);
    }
}

// Using custom errors
app.get('/users/:id', asyncHandler(async (req, res) => {
    const user = await User.findById(req.params.id);
    
    if (!user) {
        throw new NotFoundError('User not found');
    }
    
    res.json(user);
}));

// Error handling middleware (must have 4 parameters)
app.use((err, req, res, next) => {
    // Log error
    console.error(err.stack);
    
    // Operational errors (expected)
    if (err.isOperational) {
        return res.status(err.statusCode).json({
            status: 'error',
            message: err.message
        });
    }
    
    // Programming errors (unexpected)
    res.status(500).json({
        status: 'error',
        message: 'Internal server error'
    });
});

// Environment-specific error handling
const errorHandler = (err, req, res, next) => {
    err.statusCode = err.statusCode || 500;
    
    if (process.env.NODE_ENV === 'development') {
        res.status(err.statusCode).json({
            status: 'error',
            message: err.message,
            stack: err.stack,
            error: err
        });
    } else {
        // Production: don't leak error details
        res.status(err.statusCode).json({
            status: 'error',
            message: err.isOperational ? err.message : 'Something went wrong'
        });
    }
};

// Handling specific error types
app.use((err, req, res, next) => {
    // Mongoose validation error
    if (err.name === 'ValidationError') {
        const errors = Object.values(err.errors).map(e => e.message);
        return res.status(400).json({ errors });
    }
    
    // Mongoose duplicate key error
    if (err.code === 11000) {
        return res.status(400).json({ error: 'Duplicate field value' });
    }
    
    // JWT error
    if (err.name === 'JsonWebTokenError') {
        return res.status(401).json({ error: 'Invalid token' });
    }
    
    next(err);
});
```

---

## Async Patterns

<a id="q9"></a>
### Q9: How do you handle asynchronous operations in Node.js?
**Answer:**

```javascript
// 1. Callbacks (traditional)
const fs = require('fs');

fs.readFile('file.txt', 'utf8', (err, data) => {
    if (err) {
        console.error(err);
        return;
    }
    console.log(data);
});

// 2. Promises
const fsPromises = require('fs').promises;

fsPromises.readFile('file.txt', 'utf8')
    .then(data => console.log(data))
    .catch(err => console.error(err));

// 3. Async/await (recommended)
async function readFile() {
    try {
        const data = await fsPromises.readFile('file.txt', 'utf8');
        console.log(data);
    } catch (err) {
        console.error(err);
    }
}

// Promisifying callbacks
const { promisify } = require('util');
const readFilePromise = promisify(fs.readFile);

// Or use fs.promises directly
const { readFile, writeFile } = require('fs').promises;

// Parallel operations
async function parallel() {
    const [users, posts, comments] = await Promise.all([
        fetchUsers(),
        fetchPosts(),
        fetchComments()
    ]);
    return { users, posts, comments };
}

// Sequential operations
async function sequential() {
    const user = await getUser(id);
    const posts = await getPosts(user.id);
    const comments = await getComments(posts[0].id);
    return { user, posts, comments };
}

// Handling multiple promises with different outcomes
async function fetchAll() {
    const results = await Promise.allSettled([
        fetchUsers(),
        fetchPosts(),
        fetchComments()
    ]);
    
    return results.map(result => {
        if (result.status === 'fulfilled') {
            return result.value;
        }
        console.error(result.reason);
        return null;
    });
}

// Race condition - first to resolve
async function fetchWithTimeout(url, timeout = 5000) {
    return Promise.race([
        fetch(url),
        new Promise((_, reject) =>
            setTimeout(() => reject(new Error('Timeout')), timeout)
        )
    ]);
}

// Async iteration
async function* asyncGenerator() {
    yield await fetchData(1);
    yield await fetchData(2);
    yield await fetchData(3);
}

async function processData() {
    for await (const data of asyncGenerator()) {
        console.log(data);
    }
}

// Concurrent execution with limit
async function processWithLimit(items, limit, fn) {
    const results = [];
    const executing = [];
    
    for (const item of items) {
        const promise = fn(item).then(result => {
            executing.splice(executing.indexOf(promise), 1);
            return result;
        });
        
        results.push(promise);
        executing.push(promise);
        
        if (executing.length >= limit) {
            await Promise.race(executing);
        }
    }
    
    return Promise.all(results);
}

// Using p-limit library
const pLimit = require('p-limit');
const limit = pLimit(5); // Max 5 concurrent

const results = await Promise.all(
    urls.map(url => limit(() => fetch(url)))
);
```

<a id="q10"></a>
### Q10: What are Streams in Node.js?
**Answer:**

Streams handle data piece by piece, useful for large datasets.

| Stream Type | Description | Example |
|-------------|-------------|---------|
| Readable | Read data from source | fs.createReadStream |
| Writable | Write data to destination | fs.createWriteStream |
| Duplex | Both read and write | net.Socket |
| Transform | Modify data while streaming | zlib.createGzip |

```javascript
const fs = require('fs');
const zlib = require('zlib');
const { pipeline } = require('stream/promises');

// Readable stream
const readable = fs.createReadStream('large-file.txt', {
    encoding: 'utf8',
    highWaterMark: 64 * 1024 // 64KB chunks
});

readable.on('data', (chunk) => {
    console.log(`Received ${chunk.length} bytes`);
});

readable.on('end', () => {
    console.log('Finished reading');
});

readable.on('error', (err) => {
    console.error('Error:', err);
});

// Writable stream
const writable = fs.createWriteStream('output.txt');

writable.write('Hello\n');
writable.write('World\n');
writable.end(); // Signal no more data

writable.on('finish', () => {
    console.log('Finished writing');
});

// Pipe - connect streams
const readStream = fs.createReadStream('input.txt');
const writeStream = fs.createWriteStream('output.txt');

readStream.pipe(writeStream);

// Pipe with compression
fs.createReadStream('file.txt')
    .pipe(zlib.createGzip())
    .pipe(fs.createWriteStream('file.txt.gz'));

// Pipeline (handles errors better)
async function compress() {
    await pipeline(
        fs.createReadStream('file.txt'),
        zlib.createGzip(),
        fs.createWriteStream('file.txt.gz')
    );
    console.log('Compression complete');
}

// Transform stream
const { Transform } = require('stream');

const upperCase = new Transform({
    transform(chunk, encoding, callback) {
        this.push(chunk.toString().toUpperCase());
        callback();
    }
});

fs.createReadStream('input.txt')
    .pipe(upperCase)
    .pipe(fs.createWriteStream('output.txt'));

// Custom readable stream
const { Readable } = require('stream');

class Counter extends Readable {
    constructor(max) {
        super();
        this.max = max;
        this.current = 0;
    }
    
    _read() {
        this.current++;
        if (this.current <= this.max) {
            this.push(String(this.current));
        } else {
            this.push(null); // Signal end
        }
    }
}

const counter = new Counter(5);
counter.pipe(process.stdout);

// Stream with async iterator
async function processStream(stream) {
    for await (const chunk of stream) {
        console.log(chunk);
    }
}

// HTTP response streaming
const http = require('http');

http.createServer((req, res) => {
    const fileStream = fs.createReadStream('large-video.mp4');
    fileStream.pipe(res);
}).listen(3000);
```

<a id="q11"></a>
### Q11: Explain the Buffer class
**Answer:**

Buffers handle binary data directly in memory, outside V8 heap.

```javascript
// Creating buffers
const buf1 = Buffer.alloc(10); // 10 bytes, filled with 0
const buf2 = Buffer.alloc(10, 1); // 10 bytes, filled with 1
const buf3 = Buffer.allocUnsafe(10); // Uninitialized (faster but may contain old data)
const buf4 = Buffer.from([1, 2, 3, 4]);
const buf5 = Buffer.from('Hello', 'utf8');
const buf6 = Buffer.from('48656c6c6f', 'hex');

// Buffer size
console.log(buf5.length); // 5

// Reading from buffer
console.log(buf5.toString()); // 'Hello'
console.log(buf5.toString('hex')); // '48656c6c6f'
console.log(buf5.toString('base64')); // 'SGVsbG8='
console.log(buf5[0]); // 72 (ASCII for 'H')

// Writing to buffer
const buf = Buffer.alloc(10);
buf.write('Hi');
buf[2] = 33; // '!'
console.log(buf.toString()); // 'Hi!'

// Buffer methods
const buf7 = Buffer.from('Hello');
const buf8 = Buffer.from('World');

// Concatenate
const combined = Buffer.concat([buf7, Buffer.from(' '), buf8]);
console.log(combined.toString()); // 'Hello World'

// Compare
console.log(buf7.compare(buf8)); // -1 (buf7 < buf8)
console.log(buf7.equals(Buffer.from('Hello'))); // true

// Copy
const target = Buffer.alloc(10);
buf7.copy(target);
console.log(target.toString()); // 'Hello'

// Slice (shares memory!)
const slice = buf7.slice(0, 2);
console.log(slice.toString()); // 'He'

// Subarray (same as slice, shares memory)
const sub = buf7.subarray(0, 2);

// Iterate
for (const byte of buf7) {
    console.log(byte);
}

// Fill
buf.fill(0); // Zero out buffer

// indexOf
const index = buf7.indexOf('l');
console.log(index); // 2

// JSON serialization
const json = JSON.stringify(buf7);
// {"type":"Buffer","data":[72,101,108,108,111]}

const restored = Buffer.from(JSON.parse(json).data);

// Working with binary data
const fs = require('fs');

// Read file as buffer
const fileBuffer = fs.readFileSync('image.png');
console.log(fileBuffer.length); // File size in bytes

// Convert buffer to ArrayBuffer
const arrayBuffer = buf7.buffer.slice(
    buf7.byteOffset,
    buf7.byteOffset + buf7.byteLength
);

// Buffer pool (small allocations reuse memory)
// Buffers < 4KB use shared pool
const small = Buffer.from('Hi'); // Uses pool
const large = Buffer.alloc(10000); // Own memory
```

---

## Advanced Topics

<a id="q12"></a>
### Q12: How does Node.js handle child processes?
**Answer:**

```javascript
const { spawn, exec, execFile, fork } = require('child_process');

// spawn - stream-based, good for long-running processes
const ls = spawn('ls', ['-la']);

ls.stdout.on('data', (data) => {
    console.log(`stdout: ${data}`);
});

ls.stderr.on('data', (data) => {
    console.error(`stderr: ${data}`);
});

ls.on('close', (code) => {
    console.log(`Process exited with code ${code}`);
});

// exec - buffers output, good for short commands
exec('ls -la', (error, stdout, stderr) => {
    if (error) {
        console.error(`Error: ${error.message}`);
        return;
    }
    console.log(stdout);
});

// exec with Promise
const { promisify } = require('util');
const execPromise = promisify(exec);

async function runCommand() {
    const { stdout } = await execPromise('ls -la');
    console.log(stdout);
}

// execFile - similar to exec but doesn't spawn shell
execFile('node', ['--version'], (error, stdout) => {
    console.log(stdout);
});

// fork - special case for Node.js scripts, enables IPC
// parent.js
const child = fork('child.js');

child.on('message', (message) => {
    console.log('From child:', message);
});

child.send({ hello: 'world' });

// child.js
process.on('message', (message) => {
    console.log('From parent:', message);
    process.send({ response: 'received' });
});

// Worker threads (for CPU-intensive tasks)
const { Worker, isMainThread, parentPort } = require('worker_threads');

if (isMainThread) {
    const worker = new Worker(__filename);
    
    worker.on('message', (result) => {
        console.log('Result:', result);
    });
    
    worker.postMessage({ num: 1000000 });
} else {
    parentPort.on('message', (data) => {
        // CPU-intensive calculation
        let sum = 0;
        for (let i = 0; i < data.num; i++) {
            sum += i;
        }
        parentPort.postMessage(sum);
    });
}

// Worker with separate file
const worker = new Worker('./worker.js', {
    workerData: { value: 100 }
});

// worker.js
const { workerData, parentPort } = require('worker_threads');
const result = heavyComputation(workerData.value);
parentPort.postMessage(result);
```

<a id="q13"></a>
### Q13: What is clustering in Node.js?
**Answer:**

Clustering allows running multiple instances of Node.js to handle load across CPU cores.

```javascript
const cluster = require('cluster');
const http = require('http');
const numCPUs = require('os').cpus().length;

if (cluster.isPrimary) {
    console.log(`Primary ${process.pid} is running`);
    
    // Fork workers
    for (let i = 0; i < numCPUs; i++) {
        cluster.fork();
    }
    
    cluster.on('exit', (worker, code, signal) => {
        console.log(`Worker ${worker.process.pid} died`);
        // Restart worker
        cluster.fork();
    });
    
    // Listen for messages from workers
    cluster.on('message', (worker, message) => {
        console.log(`Message from worker ${worker.id}:`, message);
    });
} else {
    // Workers share TCP connection
    http.createServer((req, res) => {
        res.writeHead(200);
        res.end(`Hello from worker ${process.pid}`);
    }).listen(8000);
    
    console.log(`Worker ${process.pid} started`);
}

// Using PM2 (process manager) - easier approach
// pm2 start app.js -i max  // max = number of CPUs

// Sticky sessions for WebSocket (workers need to handle same client)
const cluster = require('cluster');
const http = require('http');
const { setupPrimary } = require('@socket.io/sticky');

if (cluster.isPrimary) {
    const httpServer = http.createServer();
    
    setupPrimary(httpServer, {
        loadBalancingMethod: 'least-connection'
    });
    
    httpServer.listen(3000);
    
    for (let i = 0; i < numCPUs; i++) {
        cluster.fork();
    }
} else {
    const httpServer = http.createServer();
    const io = require('socket.io')(httpServer);
    
    // Socket.io setup...
}

// Graceful shutdown
process.on('SIGTERM', () => {
    console.log('SIGTERM received, shutting down gracefully');
    
    server.close(() => {
        console.log('HTTP server closed');
        process.exit(0);
    });
    
    // Force close after timeout
    setTimeout(() => {
        console.error('Forcing shutdown');
        process.exit(1);
    }, 30000);
});
```

<a id="q14"></a>
### Q14: How do you handle environment variables?
**Answer:**

```javascript
// Accessing environment variables
console.log(process.env.NODE_ENV);
console.log(process.env.DATABASE_URL);

// Setting in different environments

// Command line
// NODE_ENV=production node app.js
// DATABASE_URL=postgres://... node app.js

// Package.json scripts
{
    "scripts": {
        "start": "NODE_ENV=production node app.js",
        "dev": "NODE_ENV=development node app.js"
    }
}

// .env file with dotenv
// npm install dotenv

// .env
// NODE_ENV=development
// DATABASE_URL=postgres://localhost:5432/mydb
// API_KEY=secret123

// app.js
require('dotenv').config();
// Or for specific path:
require('dotenv').config({ path: '.env.local' });

console.log(process.env.DATABASE_URL);

// Environment-specific .env files
// .env          - default
// .env.local    - local overrides
// .env.development
// .env.production

// Load based on environment
const envFile = `.env.${process.env.NODE_ENV || 'development'}`;
require('dotenv').config({ path: envFile });

// Type-safe config with validation
const config = {
    port: parseInt(process.env.PORT || '3000', 10),
    nodeEnv: process.env.NODE_ENV || 'development',
    database: {
        url: process.env.DATABASE_URL,
        pool: parseInt(process.env.DB_POOL_SIZE || '10', 10)
    },
    jwt: {
        secret: process.env.JWT_SECRET,
        expiresIn: process.env.JWT_EXPIRES_IN || '1d'
    }
};

// Validation
function validateEnv() {
    const required = ['DATABASE_URL', 'JWT_SECRET'];
    const missing = required.filter(key => !process.env[key]);
    
    if (missing.length > 0) {
        throw new Error(`Missing environment variables: ${missing.join(', ')}`);
    }
}

validateEnv();

// Using envalid library
const envalid = require('envalid');

const env = envalid.cleanEnv(process.env, {
    NODE_ENV: envalid.str({ choices: ['development', 'production', 'test'] }),
    PORT: envalid.port({ default: 3000 }),
    DATABASE_URL: envalid.url(),
    JWT_SECRET: envalid.str()
});

console.log(env.PORT); // Type-safe, validated

// Don't commit .env files!
// .gitignore:
// .env
// .env.local
// .env.*.local
```

<a id="q15"></a>
### Q15: What are common security practices in Node.js?
**Answer:**

```javascript
const express = require('express');
const helmet = require('helmet');
const rateLimit = require('express-rate-limit');
const mongoSanitize = require('express-mongo-sanitize');
const xss = require('xss-clean');
const hpp = require('hpp');

const app = express();

// 1. Set security headers
app.use(helmet());

// 2. Rate limiting
const limiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 100, // 100 requests per window
    message: 'Too many requests'
});
app.use('/api', limiter);

// 3. Body parser limits
app.use(express.json({ limit: '10kb' }));

// 4. Data sanitization against NoSQL injection
app.use(mongoSanitize());

// 5. Data sanitization against XSS
app.use(xss());

// 6. Prevent parameter pollution
app.use(hpp({
    whitelist: ['duration', 'price'] // Allow duplicates for these
}));

// 7. CORS configuration
const cors = require('cors');
app.use(cors({
    origin: ['https://trusted-site.com'],
    credentials: true
}));

// 8. Input validation
const { body, validationResult } = require('express-validator');

app.post('/users',
    body('email').isEmail().normalizeEmail(),
    body('password').isLength({ min: 8 }),
    (req, res) => {
        const errors = validationResult(req);
        if (!errors.isEmpty()) {
            return res.status(400).json({ errors: errors.array() });
        }
        // Process request...
    }
);

// 9. Secure password hashing
const bcrypt = require('bcrypt');

async function hashPassword(password) {
    const salt = await bcrypt.genSalt(12);
    return bcrypt.hash(password, salt);
}

async function verifyPassword(password, hash) {
    return bcrypt.compare(password, hash);
}

// 10. JWT best practices
const jwt = require('jsonwebtoken');

function generateToken(user) {
    return jwt.sign(
        { id: user.id, email: user.email },
        process.env.JWT_SECRET,
        {
            expiresIn: '1h',
            audience: 'myapp',
            issuer: 'myapp'
        }
    );
}

// 11. SQL injection prevention (parameterized queries)
// Bad
// db.query(`SELECT * FROM users WHERE id = ${userId}`);

// Good
// db.query('SELECT * FROM users WHERE id = $1', [userId]);

// 12. Secure cookie settings
app.use(session({
    secret: process.env.SESSION_SECRET,
    cookie: {
        secure: true,       // HTTPS only
        httpOnly: true,     // No JavaScript access
        sameSite: 'strict', // CSRF protection
        maxAge: 3600000     // 1 hour
    }
}));

// 13. Content Security Policy
app.use(helmet.contentSecurityPolicy({
    directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'", "'unsafe-inline'"],
        styleSrc: ["'self'", "'unsafe-inline'"],
        imgSrc: ["'self'", 'data:', 'https:']
    }
}));

// 14. Error handling - don't leak info
app.use((err, req, res, next) => {
    if (process.env.NODE_ENV === 'production') {
        res.status(500).json({ error: 'Internal server error' });
    } else {
        res.status(500).json({ error: err.message, stack: err.stack });
    }
});

// 15. Regular dependency updates
// npm audit
// npm audit fix
```

---

[← TypeScript Basics](typescript-basics.md) | [Back to Frontend Index](README.md) | [React Basics →](react-basics.md)
