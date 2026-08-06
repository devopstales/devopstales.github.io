---
title: Kilo Code: Understanding Modes and the Orchestrator
url: https://devopstales.github.io/ai/kilo-code-series-04-modes/
date: 2026-03-28
---


In our previous post, we unlocked massive power by integrating Qwen Code CLI. But with great power comes the need for great management.

<!--more-->

{{< content "/filedir/kilo-code-series.html" >}}

Kilo Code isn't just a single AI assistant; it is a team of specialized personas. In this post, we'll explore the different **Modes** that Kilo uses to break down and execute complex software engineering tasks.

## Why Modes Matter

When you ask a general-purpose AI to "build a REST API," it might jump straight into writing code without considering:

- What's the overall architecture?
- Which database should we use?
- How do we handle authentication?
- What about error handling and logging?

This is where **Modes** come in. Kilo Code separates concerns by using specialized agents for different phases of development:

| Mode | Purpose | When to Use |
|------|---------|-------------|
| **Orchestrator** | Task coordination and planning | Starting new features, complex multi-step tasks |
| **Architect** | System design and architecture | New projects, major refactors, technical decisions |
| **Code** | Implementation and refactoring | Writing code, fixing bugs, adding features |
| **Debug** | Troubleshooting and error resolution | Build failures, runtime errors, test failures |

Think of it like a software development team: the Orchestrator is your project manager, the Architect is your tech lead, the Code mode is your senior developer, and Debug is your dedicated SRE.

---

## Orchestrator Mode: The Conductor

**Orchestrator Mode** is the default mode in Kilo Code. It's designed to handle complex, multi-step tasks by breaking them down and delegating to specialized modes when needed.

### How It Works

When you give the Orchestrator a task, it follows this workflow:

```bash
1. Understand the request
2. Analyze the codebase context
3. Create a step-by-step plan
4. Execute each step (using appropriate modes)
5. Verify the results
6. Report completion
```

### Example: Building a User Authentication System

Let's see the Orchestrator in action:

```bash
User: "Add user authentication to our Express.js app"

Orchestrator Response:
I'll help you add user authentication. Here's my plan:

Phase 1: Architecture (Architect Mode)
- Design the authentication flow
- Choose between JWT vs sessions
- Plan database schema for users

Phase 2: Implementation (Code Mode)
- Install required dependencies (jsonwebtoken, bcrypt)
- Create user model with password hashing
- Build registration endpoint
- Build login endpoint
- Create authentication middleware

Phase 3: Testing (Debug Mode)
- Write unit tests for auth functions
- Test login/logout flow
- Verify token validation

Shall I proceed with this plan?
```

### Orchestrator Configuration

You can customize Orchestrator behavior in your `.kilocode/config.json`:

```json
{
  "orchestrator": {
    "autoPlan": true,
    "requireApproval": false,
    "maxSteps": 10,
    "delegateToSpecialists": true,
    "verboseLogging": false
  }
}
```

**Key Settings:**

| Setting | Description | Default |
|---------|-------------|---------|
| `autoPlan` | Automatically create execution plans | `true` |
| `requireApproval` | Ask before each major step | `false` |
| `maxSteps` | Maximum steps before asking for guidance | `10` |
| `delegateToSpecialists` | Switch to Architect/Code/Debug modes | `true` |

### When to Use Orchestrator Mode

- **Complex features** requiring multiple files and steps
- **New project setup** with multiple components
- **Refactoring projects** that span the codebase
- **Integration tasks** connecting multiple services

**Command:**
```bash
# CLI
kilo-code --mode orchestrator "Migrate our database from MongoDB to PostgreSQL"

# In IDE, use the mode selector or type:
/mode orchestrator
```

---

## Architect Mode: The Designer

**Architect Mode** focuses on high-level system design without writing implementation code. It's perfect for planning before you build.

### Key Capabilities

1. **System Design**: Creates architecture diagrams and component relationships
2. **Technology Selection**: Recommends libraries, frameworks, and tools
3. **API Design**: Designs REST/GraphQL schemas and contracts
4. **Database Modeling**: Plans schemas, indexes, and relationships
5. **Security Review**: Identifies potential security concerns

### Example: Designing a Microservices Architecture

```bash
User: "Design a microservices architecture for our e-commerce platform"

Architect Response:
## Proposed Architecture

### Services
1. **API Gateway** (Kong/Express)
   - Route requests to appropriate services
   - Handle authentication and rate limiting

2. **Product Service** (Node.js + PostgreSQL)
   - Product catalog management
   - Search and filtering
   - Inventory tracking

3. **Order Service** (Node.js + PostgreSQL)
   - Order creation and management
   - Order status tracking
   - Integration with payment service

4. **Payment Service** (Python + Stripe API)
   - Payment processing
   - Refund handling
   - Webhook management

5. **User Service** (Node.js + Redis)
   - User authentication (JWT)
   - Session management
   - Profile management

### Communication Pattern
- Synchronous: REST APIs for user-facing operations
- Asynchronous: RabbitMQ for order processing events

### Data Flow
[Client] → [API Gateway] → [Services] → [Databases]
                      ↓
                 [RabbitMQ]
                      ↓
            [Analytics Service]

### Technology Recommendations
- Container orchestration: Kubernetes
- Service mesh: Istio (for production)
- Monitoring: Prometheus + Grafana
- Logging: ELK Stack

Shall I create detailed API specifications for each service?
```

### Architect Mode Output Formats

Architect mode can generate various output formats:

```bash
# Generate architecture diagram (Mermaid format)
/architect --output mermaid "Design a CI/CD pipeline"

# Generate API specification (OpenAPI/Swagger)
/architect --output openapi "Design REST API for blog posts"

# Generate database schema (SQL)
/architect --output sql "Design schema for social media app"

# Generate decision document (Markdown)
/architect --output markdown "Choose between Redis and Memcached"
```

### Architect Configuration

```json
{
  "architect": {
    "defaultOutputFormat": "markdown",
    "includeDiagrams": true,
    "considerSecurity": true,
    "considerScalability": true,
    "maxServices": 10
  }
}
```

### When to Use Architect Mode

- **New project planning** before writing any code
- **Major refactoring** decisions
- **Technology evaluation** and selection
- **System design reviews**
- **API contract design**

**Command:**

```bash
# CLI
kilo-code --mode architect "Design the database schema for a task management app"

# In IDE
/mode architect
/design our new feature flag system
```

---

## Code Mode: The Builder

**Code Mode** is where the actual implementation happens. This mode focuses on writing, modifying, and refactoring code with precision.

### Key Capabilities

1. **Multi-file Editing**: Edits multiple files in a single operation
2. **Code Generation**: Creates new files and functions from specifications
3. **Refactoring**: Improves code structure without changing behavior
4. **Test Generation**: Creates unit and integration tests
5. **Documentation**: Adds comments and generates documentation

### Example: Implementing a Feature

```bash
User: "Implement the user registration endpoint we designed"

Code Mode Response:
I'll implement the user registration endpoint based on the architecture.

## Files to Create/Modify

### 1. models/User.js (NEW)
```javascript
const mongoose = require('mongoose');
const bcrypt = require('bcrypt');

const userSchema = new mongoose.Schema({
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,
    trim: true
  },
  password: {
    type: String,
    required: true,
    minlength: 8
  },
  name: {
    type: String,
    required: true,
    trim: true
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 12);
  next();
});

// Compare password method
userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### 2. routes/auth.js (NEW)

```javascript
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const jwt = require('jsonwebtoken');

router.post('/register', async (req, res) => {
  try {
    const { email, password, name } = req.body;
    
    // Check if user exists
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ error: 'Email already registered' });
    }
    
    // Create user
    const user = new User({ email, password, name });
    await user.save();
    
    // Generate JWT
    const token = jwt.sign(
      { userId: user._id },
      process.env.JWT_SECRET,
      { expiresIn: '7d' }
    );
    
    res.status(201).json({
      message: 'User registered successfully',
      token,
      user: {
        id: user._id,
        email: user.email,
        name: user.name
      }
    });
  } catch (error) {
    res.status(500).json({ error: 'Server error' });
  }
});

module.exports = router;
```

### 3. app.js (MODIFY)

```javascript
// Add this import
const authRoutes = require('./routes/auth');

// Add this route registration
app.use('/api/auth', authRoutes);
```

## Tests Generated

I've also created `tests/auth.test.js` with:
- Registration success test
- Duplicate email test
- Password hashing verification
- JWT token validation

Ready to apply these changes?

```bash

### Code Mode Features

**Smart Diff Preview:**
Before applying changes, Code Mode shows a diff:

```diff
--- a/app.js
+++ b/app.js
@@ -5,6 +5,8 @@ const express = require('express');
 const cors = require('cors');
 const mongoose = require('mongoose');
 
+const authRoutes = require('./routes/auth');
+
 const app = express();
 
 app.use(cors());
@@ -15,6 +17,8 @@ app.use(express.json());
 // Routes
 app.use('/api/users', userRoutes);
 
+app.use('/api/auth', authRoutes);
+
 // Error handling
 app.use((err, req, res, next) => {
   console.error(err.stack);
```

**Apply Options:**
- `y` - Apply all changes
- `n` - Reject all changes
- `d` - Show detailed diff
- `s` - Apply selected changes only

### Code Mode Configuration

```json
{
  "code": {
    "autoFormat": true,
    "generateTests": true,
    "addComments": true,
    "preferFunctional": false,
    "maxFilesPerRequest": 10,
    "backupBeforeModify": true
  }
}
```

### When to Use Code Mode

- **Implementing features** from specifications
- **Refactoring existing code**
- **Adding tests** to existing functionality
- **Fixing bugs** with known solutions
- **Writing documentation**

**Command:**

```bash
# CLI
kilo-code --mode code "Implement the password reset functionality"

# In IDE
/mode code
/create the user profile component
```

---

## Debug Mode: The Detective

**Debug Mode** specializes in finding and fixing bugs. It analyzes error messages, stack traces, and test failures to identify root causes.

### Key Capabilities

1. **Error Analysis**: Parses and explains error messages
2. **Log Analysis**: Reviews application logs for anomalies
3. **Test Failure Investigation**: Diagnoses why tests fail
4. **Performance Profiling**: Identifies bottlenecks
5. **Security Scanning**: Finds potential vulnerabilities

### Example: Debugging a Production Issue

`````bash
User: "Our API is returning 500 errors on the /checkout endpoint"

Debug Mode Response:
I'll investigate the /checkout endpoint issue.

## Step 1: Analyze Error Logs

Found these errors in the last hour:

```
Error: Cannot read properties of undefined (reading 'price')
    at calculateTotal (/app/services/order.js:45:23)
    at checkoutHandler (/app/routes/checkout.js:28:18)
    at process.processTicksAndRejections (node:internal/process/task_queues:95:5)
```

## Step 2: Examine the Code

**File: /app/services/order.js:45**
```javascript
function calculateTotal(items) {
  let total = 0;
  for (const item of items) {
    total += item.price * item.quantity;  // Line 45 - error here
  }
  return total;
}
```

## Root Cause

The `items` array contains products where `price` is sometimes `undefined`.
This happens when products are loaded without their pricing information.

## Step 3: Check Related Code

**File: /app/routes/checkout.js**
```javascript
async function checkoutHandler(req, res) {
  const cart = await getCart(req.user.id);
  const items = await getProducts(cart.itemIds);  // Missing price join
  const total = calculateTotal(items);
  // ...
}
```

## Proposed Fix

```diff
--- a/app/services/order.js
+++ b/app/services/order.js
@@ -1,8 +1,12 @@
 function calculateTotal(items) {
   let total = 0;
   for (const item of items) {
-    total += item.price * item.quantity;
+    if (!item.price) {
+      throw new Error(`Product ${item.id} has no price`);
+    }
+    total += item.price * item.quantity;
   }
   return total;
 }
```

```diff
--- a/app/routes/checkout.js
+++ b/app/routes/checkout.js
@@ -5,7 +5,7 @@ async function checkoutHandler(req, res) {
   const cart = await getCart(req.user.id);
-  const items = await getProducts(cart.itemIds);
+  const items = await getProducts(cart.itemIds, { include: 'pricing' });
   const total = calculateTotal(items);
   // ...
 }
```

## Verification

I'll run the checkout tests to verify the fix:

```bash
$ npm test -- checkout
✓ checkout should calculate total correctly
✓ checkout should handle missing prices
✓ checkout should create order

All tests passed!
```

Apply these fixes?
`````

### Debug Mode Tools

Debug mode can use various diagnostic tools:

```bash
# Run with specific debugger
/debug --tool lldb "Investigate the segmentation fault"

# Analyze logs
/debug --tool logs "Find errors in the last hour"

# Profile performance
/debug --tool profiler "Find slow database queries"

# Check memory leaks
/debug --tool heap "Analyze memory usage"
```

### Debug Mode Configuration

```json
{
  "debug": {
    "autoRunTests": true,
    "collectLogs": true,
    "maxLogLines": 500,
    "enableProfiling": false,
    "breakOnFirstError": false
  }
}
```

### When to Use Debug Mode

- **Build failures** and compilation errors
- **Runtime errors** and crashes
- **Test failures** in CI/CD
- **Performance issues** and slow queries
- **Memory leaks** and resource exhaustion

**Command:**

```bash
# CLI
kilo-code --mode debug "The tests are failing with TypeError"

# In IDE
/mode debug
/fix the build error
```

---

## Switching Between Modes

### In Kilo Code IDE

**Method 1: Mode Selector**
1. Look for the mode dropdown in the chat panel
2. Select your desired mode
3. Continue the conversation

**Method 2: Slash Commands**

```bash
/mode orchestrator
/mode architect
/mode code
/mode debug
```

**Method 3: Contextual Switching**
The Orchestrator automatically switches modes based on task needs:

```bash
User: "Build a complete authentication system"
Orchestrator: "I'll plan this (Architect), then implement (Code), then test (Debug)"
```

### In Kilo Code CLI

```bash
# Set mode for session
kilo-code --mode architect

# One-time mode switch
kilo-code "/mode code Implement the login function"

# Mode with specific options
kilo-code --mode debug --tool logs "Find the error"
```

### Mode Persistence

Modes persist within a session but reset between sessions. To set a default:

```json
// .kilocode/config.json
{
  "defaultMode": "orchestrator",
  "modeByTask": {
    "design": "architect",
    "implement": "code",
    "fix": "debug",
    "plan": "orchestrator"
  }
}
```

---

## Advanced: Custom Modes

You can create custom modes for your team's specific needs:

```json
// .kilocode/modes/security-review.json
{
  "name": "security-review",
  "baseMode": "architect",
  "systemPrompt": "You are a security expert reviewing code for vulnerabilities.",
  "tools": ["code-scanner", "dependency-checker", "secret-detector"],
  "restrictions": {
    "cannotModifyFiles": true,
    "mustCiteOWASP": true
  }
}
```

**Using Custom Mode:**

```bash
kilo-code --mode security-review "Audit our authentication code"
```

---

## Mode Comparison Table

| Feature | Orchestrator | Architect | Code | Debug |
|---------|-------------|-----------|------|-------|
| **Planning** | ✓✓✓ | ✓✓✓ | ✓ | ✓ |
| **Design** | ✓ | ✓✓✓ | - | - |
| **Implementation** | ✓ | - | ✓✓✓ | ✓ |
| **Testing** | ✓ | - | ✓ | ✓✓✓ |
| **File Modification** | ✓ | - | ✓✓✓ | ✓ |
| **Tool Access** | All | Limited | All | Diagnostic |
| **Best For** | Complex tasks | Design decisions | Writing code | Fixing bugs |

---

## Best Practices

### 1. Start with Orchestrator for Complex Tasks

```bash
Good: "Build a complete blog platform with user authentication"
(The Orchestrator will plan and delegate appropriately)
```

### 2. Use Architect for Major Decisions

```bash
Good: "Should we use MongoDB or PostgreSQL for our analytics platform?"
(The Architect will provide a detailed comparison)
```

### 3. Use Code for Implementation

```bash
Good: "Implement the blog post CRUD endpoints based on our design"
(Code mode will write the actual implementation)
```

### 4. Use Debug for Problems

```bash
Good: "The login endpoint returns 401 for valid credentials"
(Debug mode will investigate and fix)
```

### 5. Don't Fight the Modes

```bash
Bad: Asking Code mode to design architecture
Bad: Asking Architect mode to write implementation
```

---

## Real-World Example: Full Feature Development

Here's how modes work together for a complete feature:

```bash
Session: Add password reset functionality

1. [Orchestrator] Receives request, creates plan
   "I'll design (Architect), implement (Code), and test (Debug)"

2. [Architect] Designs the flow
   - Email template design
   - Token generation strategy
   - API endpoint design
   - Security considerations

3. [Code] Implements the design
   - Creates reset token model
   - Builds email service integration
   - Implements /forgot-password endpoint
   - Implements /reset-password endpoint

4. [Debug] Tests and verifies
   - Runs unit tests
   - Tests email delivery (mocked)
   - Verifies token expiration
   - Checks for security issues

5. [Orchestrator] Reports completion
   "Password reset feature complete. All tests passing."
```

