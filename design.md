# Design Document: Code Obsidian - AI Pair Programming for Solo Learners

## 📋 Document Information

**Project Name:** Code Obsidian  
**Version:** 1.0 - Hackathon MVP  
**Date:** February 14, 2026  
**Track:** [Student Track] AI for Learning & Developer Productivity  
**Team Size:** 2-4 developers  
**Build Time:** 48-72 hours

---

## 🎨 Design Philosophy

**Core Principle:** Build the minimum that proves the innovation  
**Focus:** Multi-agent intelligence + Real-time interaction (our differentiators)  
**Skip:** Over-engineering, premature optimization, non-essential features

### What Makes This Design Special

Instead of a massive microservices architecture (overkill for hackathon), we're building a **smart monolith** with clear agent separation - proving the concept works before scaling.

---

## 🏗️ System Architecture

### High-Level Architecture (Realistic for 48 Hours)

```
┌─────────────────────────────────────────────────────────────┐
│                    Frontend (React)                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ Code Editor  │  │ Chat Panel   │  │ Skill Tree   │      │
│  │  (Monaco)    │  │ (Agent UI)   │  │ (Progress)   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                     WebSocket Connection                     │
└─────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│              Backend (Node.js + Express)                     │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │         Agent Orchestrator (Brain)                     │ │
│  │   Decides which agent to invoke based on context       │ │
│  └────────────────────────────────────────────────────────┘ │
│                     │           │           │                │
│         ┌───────────┼───────────┼───────────┼────────┐      │
│         ▼           ▼           ▼           ▼        ▼      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────┐ ┌──────┐  │
│  │ Teacher  │ │   Code   │ │Debugging │ │Code │ │Mistake│ │
│  │  Agent   │ │  Review  │ │  Agent   │ │Exec │ │ DB   │  │
│  └──────────┘ └──────────┘ └──────────┘ └─────┘ └──────┘  │
│       │             │             │          │        │      │
│       └─────────────┴─────────────┴──────────┴────────┘      │
│                          │                                    │
│                    Claude API                                 │
│              (Anthropic Sonnet 4)                            │
└─────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                   Data Layer                                 │
│  ┌──────────────┐         ┌──────────────┐                  │
│  │ PostgreSQL   │         │    Redis     │                  │
│  │ (User Data,  │         │  (Sessions,  │                  │
│  │  Progress)   │         │   Cache)     │                  │
│  └──────────────┘         └──────────────┘                  │
└─────────────────────────────────────────────────────────────┘
```

---

## 🤖 Multi-Agent System Design (The Innovation Core)

### Agent Architecture

Each agent is a specialized prompt + persona + context manager:

```javascript
// Agent Base Class
class Agent {
  constructor(name, systemPrompt, personality) {
    this.name = name;
    this.systemPrompt = systemPrompt;
    this.personality = personality;
  }

  async respond(userInput, context) {
    const prompt = this.buildPrompt(userInput, context);
    const response = await callClaude(prompt, this.systemPrompt);
    return this.formatResponse(response);
  }

  buildPrompt(userInput, context) {
    // Combines user input + session context + agent-specific instructions
  }
}
```

### Agent 1: Teacher Agent (Socratic Method)

**System Prompt:**
```
You are a patient, encouraging programming teacher who uses the Socratic method.
Your goal: Help students THINK through problems, not give answers.

CRITICAL RULES:
1. NEVER give direct code solutions on first ask
2. ALWAYS respond with guiding questions first
3. Use analogies and real-world examples
4. Celebrate small wins ("Great thinking!")
5. If student is truly stuck after 3 attempts, provide hints

Your personality: Patient, encouraging, curious
Reading level: 8th grade
Tone: Friendly mentor, not robotic tutor

Current student level: {userLevel}
Student's preferred style: {learningStyle}
Recent topics covered: {recentTopics}
```

**Example Interaction Flow:**
```javascript
// User asks: "How do I reverse a string in Python?"

teacherAgent.respond(userQuery, sessionContext) 
// Returns:
{
  agent: "Teacher",
  message: "Great question! Before I help, let's think together: 
            What does 'reverse' mean? Can you describe what the 
            output should look like if input is 'hello'?",
  suggestedActions: ["Think aloud", "Try an approach", "Need a hint"],
  metadata: { 
    teachingTechnique: "socratic_questioning",
    expectedNextStep: "user_explains_understanding"
  }
}
```

### Agent 2: Code Review Agent (Real-Time)

**System Prompt:**
```
You are a senior developer doing real-time code review for a junior.
You watch code AS IT'S WRITTEN and provide proactive guidance.

CRITICAL RULES:
1. Intervene BEFORE mistakes compound
2. Catch common anti-patterns early
3. Explain WHY something is problematic, not just WHAT
4. Encourage good practices ("Nice variable naming!")
5. Don't nitpick - focus on learning moments

Intervention triggers:
- Common mistake patterns detected
- Potential logic errors
- Security issues
- Performance concerns
- Style violations (major only)

Your personality: Experienced but not condescending
Tone: "Hey, quick thought before you continue..."
```

**Example Interaction Flow:**
```javascript
// User types: def calculate(a, b, c, d, e):

codeReviewAgent.analyzeCode(currentCode, cursorPosition)
// Returns:
{
  agent: "CodeReview",
  type: "proactive_warning",
  message: "Quick thought: That's a lot of parameters! 
            Have you considered using a dictionary or object? 
            This will make your function easier to use. Want to explore that?",
  severity: "suggestion", // "info", "suggestion", "warning", "error"
  lineNumber: 1,
  suggestedFix: {
    before: "def calculate(a, b, c, d, e):",
    after: "def calculate(config):",
    explanation: "Group related parameters into a single config object"
  }
}
```

### Agent 3: Debugging Agent

**System Prompt:**
```
You are a debugging specialist who teaches debugging METHODOLOGY.
Don't just fix bugs - teach students HOW to debug.

CRITICAL RULES:
1. Follow structured debugging process:
   - Reproduce the error
   - Read the error message
   - Form a hypothesis
   - Test the hypothesis
   - Fix and verify
2. Ask diagnostic questions
3. Teach rubber duck debugging
4. Celebrate when STUDENT finds the bug

Your personality: Detective, methodical problem-solver
Tone: "Let's solve this mystery together!"
```

**Example Interaction Flow:**
```javascript
// User: "My code returns None instead of 5"

debuggingAgent.diagnose(code, expectedOutput, actualOutput)
// Returns:
{
  agent: "Debugger",
  diagnosticQuestions: [
    "Does your function have a return statement?",
    "Can you add print() statements to see what's happening?",
    "What value does your variable have before returning?"
  ],
  debuggingPlan: [
    "Step 1: Check if function has return statement",
    "Step 2: Verify the calculation logic",
    "Step 3: Print intermediate values"
  ],
  commonCause: "missing_return_statement",
  educationalNote: "This is one of the top 5 beginner mistakes! 
                   Let's learn how to catch it early."
}
```

### Agent Orchestrator (Brain)

**Decision Logic:**
```javascript
class AgentOrchestrator {
  selectAgent(event, context) {
    // Event types:
    // - "question_asked"
    // - "code_written"
    // - "code_executed_with_error"
    // - "user_stuck" (detected by inactivity)
    
    if (event.type === "question_asked") {
      if (isConceptQuestion(event.content)) {
        return "Teacher";
      } else if (isDebuggingRequest(event.content)) {
        return "Debugger";
      }
    }
    
    if (event.type === "code_written") {
      const issues = detectIssues(event.code);
      if (issues.length > 0) {
        return "CodeReview";
      }
    }
    
    if (event.type === "code_executed_with_error") {
      return "Debugger";
    }
    
    if (event.type === "user_stuck") {
      // User hasn't typed in 60 seconds
      return "Teacher"; // Gently offer help
    }
    
    return "Teacher"; // Default
  }
  
  async route(event, context) {
    const agentName = this.selectAgent(event, context);
    const agent = this.agents[agentName];
    const response = await agent.respond(event, context);
    
    // Add metadata for UI
    response.agentName = agentName;
    response.agentColor = this.agentColors[agentName];
    response.agentAvatar = this.agentAvatars[agentName];
    
    return response;
  }
}
```

---

## 💻 Frontend Architecture

### Tech Stack
- **Framework:** React 18 + TypeScript
- **State Management:** Zustand (lightweight, perfect for hackathon)
- **Editor:** Monaco Editor (VS Code's editor)
- **WebSocket:** Socket.io-client
- **UI Library:** Tailwind CSS + shadcn/ui
- **Charts:** Recharts (for skill tree visualization)

### Component Structure

```
src/
├── App.tsx
├── components/
│   ├── CodeEditor/
│   │   ├── Editor.tsx           # Monaco editor wrapper
│   │   ├── OutputPanel.tsx      # Code execution results
│   │   └── EditorToolbar.tsx    # Run, Explain, Reset buttons
│   ├── AgentChat/
│   │   ├── ChatPanel.tsx        # Main chat interface
│   │   ├── AgentMessage.tsx     # Individual message with agent indicator
│   │   ├── TypingIndicator.tsx  # Shows which agent is "thinking"
│   │   └── QuickActions.tsx     # Suggested next steps
│   ├── SkillTree/
│   │   ├── SkillTree.tsx        # Visual progress tree
│   │   ├── SkillNode.tsx        # Individual skill node
│   │   └── ProgressBar.tsx      # Overall progress
│   └── Layout/
│       ├── Header.tsx
│       ├── Sidebar.tsx
│       └── MainLayout.tsx
├── hooks/
│   ├── useAgent.ts              # Hook to interact with agents
│   ├── useCodeExecution.ts      # Hook for running code
│   └── useSession.ts            # Session management
├── services/
│   ├── websocket.ts             # WebSocket connection
│   ├── api.ts                   # HTTP requests
│   └── codeAnalysis.ts          # Client-side code parsing
└── stores/
    ├── editorStore.ts           # Code editor state
    ├── agentStore.ts            # Agent conversation state
    └── userStore.ts             # User progress state
```

### Key Component: Live Code Editor

```typescript
// components/CodeEditor/Editor.tsx
import Editor from '@monaco-editor/react';
import { useEffect, useRef } from 'react';
import { useAgent } from '@/hooks/useAgent';

export function CodeEditor() {
  const editorRef = useRef(null);
  const { analyzeCode, getProactiveFeedback } = useAgent();
  const [code, setCode] = useState('');
  const [lastTypingTime, setLastTypingTime] = useState(Date.now());

  useEffect(() => {
    // Trigger code analysis on pause (debounced)
    const timer = setTimeout(() => {
      if (Date.now() - lastTypingTime > 2000) {
        analyzeCode(code);
      }
    }, 2000);

    return () => clearTimeout(timer);
  }, [code, lastTypingTime]);

  const handleEditorChange = (value: string) => {
    setCode(value);
    setLastTypingTime(Date.now());
    
    // Real-time mistake detection
    const mistakes = detectCommonMistakes(value);
    if (mistakes.length > 0) {
      getProactiveFeedback(mistakes[0]);
    }
  };

  return (
    <div className="editor-container">
      <Editor
        height="600px"
        language="python"
        value={code}
        onChange={handleEditorChange}
        theme="vs-dark"
        options={{
          minimap: { enabled: false },
          fontSize: 14,
          lineNumbers: 'on',
          automaticLayout: true,
          suggestOnTriggerCharacters: false, // Disable autocomplete to not interfere
        }}
      />
    </div>
  );
}
```

### Key Component: Agent Chat Panel

```typescript
// components/AgentChat/ChatPanel.tsx
import { useEffect, useRef } from 'react';
import { AgentMessage } from './AgentMessage';
import { useAgentStore } from '@/stores/agentStore';

export function ChatPanel() {
  const messages = useAgentStore((state) => state.messages);
  const sendMessage = useAgentStore((state) => state.sendMessage);
  const messagesEndRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [messages]);

  const agentColors = {
    Teacher: 'bg-blue-100 border-blue-500',
    CodeReview: 'bg-green-100 border-green-500',
    Debugger: 'bg-purple-100 border-purple-500',
  };

  return (
    <div className="chat-panel h-full flex flex-col">
      <div className="messages flex-1 overflow-y-auto p-4 space-y-4">
        {messages.map((msg) => (
          <AgentMessage
            key={msg.id}
            agent={msg.agent}
            content={msg.content}
            timestamp={msg.timestamp}
            color={agentColors[msg.agent]}
          />
        ))}
        <div ref={messagesEndRef} />
      </div>

      <div className="input-area p-4 border-t">
        <input
          type="text"
          placeholder="Ask a question or explain your thinking..."
          className="w-full p-2 border rounded"
          onKeyPress={(e) => {
            if (e.key === 'Enter') {
              sendMessage(e.currentTarget.value);
              e.currentTarget.value = '';
            }
          }}
        />
      </div>
    </div>
  );
}
```

---

## 🔧 Backend Architecture

### Tech Stack
- **Runtime:** Node.js 18+
- **Framework:** Express.js
- **WebSocket:** Socket.io
- **Database:** PostgreSQL (via Prisma ORM)
- **Cache:** Redis
- **Code Execution:** Docker containers (Python runtime)
- **AI:** Anthropic Claude API

### API Structure

```
Backend/
├── server.js                # Entry point
├── routes/
│   ├── auth.js             # User authentication
│   ├── session.js          # Session management
│   └── progress.js         # Progress tracking
├── agents/
│   ├── Agent.js            # Base agent class
│   ├── TeacherAgent.js     # Teacher implementation
│   ├── CodeReviewAgent.js  # Code review implementation
│   ├── DebuggingAgent.js   # Debugging implementation
│   └── Orchestrator.js     # Agent routing logic
├── services/
│   ├── claudeService.js    # Claude API wrapper
│   ├── codeExecutor.js     # Docker-based code execution
│   ├── mistakeDetector.js  # Pattern matching for common errors
│   └── sessionManager.js   # Session state management
├── websocket/
│   └── handlers.js         # WebSocket event handlers
├── database/
│   ├── schema.prisma       # Database schema
│   └── migrations/         # Database migrations
└── utils/
    ├── promptBuilder.js    # Builds context-rich prompts
    └── responseParser.js   # Parses agent responses
```

### Core Service: Claude Integration

```javascript
// services/claudeService.js
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
});

class ClaudeService {
  async chat(systemPrompt, userMessage, context = {}) {
    const contextualPrompt = this.buildContextualPrompt(userMessage, context);

    const response = await client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 1000,
      system: systemPrompt,
      messages: [
        {
          role: 'user',
          content: contextualPrompt,
        },
      ],
    });

    return this.parseResponse(response);
  }

  buildContextualPrompt(userMessage, context) {
    return `
      Student Level: ${context.userLevel || 'beginner'}
      Learning Style: ${context.learningStyle || 'guided'}
      Current Code:
      \`\`\`python
      ${context.currentCode || 'No code yet'}
      \`\`\`
      
      Recent Topics: ${context.recentTopics?.join(', ') || 'None'}
      Mistakes Made Today: ${context.todaysMistakes?.length || 0}
      
      User Query: ${userMessage}
    `;
  }

  parseResponse(response) {
    return {
      text: response.content[0].text,
      usage: {
        inputTokens: response.usage.input_tokens,
        outputTokens: response.usage.output_tokens,
      },
    };
  }

  async streamChat(systemPrompt, userMessage, context, onChunk) {
    const stream = await client.messages.stream({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 1000,
      system: systemPrompt,
      messages: [
        {
          role: 'user',
          content: this.buildContextualPrompt(userMessage, context),
        },
      ],
    });

    stream.on('text', (text) => {
      onChunk(text);
    });

    return stream.finalMessage();
  }
}

export default new ClaudeService();
```

### Core Service: Mistake Detection

```javascript
// services/mistakeDetector.js
const COMMON_MISTAKES = [
  {
    id: 'shadow_builtin',
    pattern: /^\s*(list|dict|str|int|float|set|tuple)\s*=/m,
    message: "⚠️ You're shadowing a Python built-in type! This will cause issues.",
    severity: 'warning',
    suggestion: 'Use a different variable name like my_list, numbers, etc.',
  },
  {
    id: 'missing_colon',
    pattern: /^\s*(if|elif|else|for|while|def|class)\s+[^:]*$/m,
    message: "❌ Missing colon at the end of this line!",
    severity: 'error',
    suggestion: 'Add a : at the end of the line',
  },
  {
    id: 'indentation_tabs',
    pattern: /^\t/m,
    message: "⚠️ Mixing tabs and spaces can cause IndentationError",
    severity: 'warning',
    suggestion: 'Use 4 spaces for indentation (PEP 8)',
  },
  {
    id: 'mutable_default_arg',
    pattern: /def\s+\w+\([^)]*=\s*\[\s*\]/,
    message: "🚨 Mutable default argument detected! This is a classic Python gotcha.",
    severity: 'warning',
    suggestion: 'Use None as default and create list inside function',
  },
  // ... 50+ more patterns
];

export function detectMistakes(code) {
  const mistakes = [];

  for (const mistake of COMMON_MISTAKES) {
    if (mistake.pattern.test(code)) {
      const match = code.match(mistake.pattern);
      mistakes.push({
        ...mistake,
        line: code.substr(0, match.index).split('\n').length,
        snippet: match[0],
      });
    }
  }

  return mistakes;
}
```

### WebSocket Handler

```javascript
// websocket/handlers.js
import { Server } from 'socket.io';
import { AgentOrchestrator } from '../agents/Orchestrator.js';
import { SessionManager } from '../services/sessionManager.js';

export function setupWebSocket(server) {
  const io = new Server(server, {
    cors: { origin: process.env.FRONTEND_URL },
  });

  io.on('connection', (socket) => {
    console.log('User connected:', socket.id);

    const session = SessionManager.getOrCreate(socket.id);
    const orchestrator = new AgentOrchestrator(session);

    // User asks a question
    socket.on('user_message', async (message) => {
      const response = await orchestrator.route({
        type: 'question_asked',
        content: message,
      }, session.getContext());

      socket.emit('agent_response', response);
    });

    // User writes code
    socket.on('code_update', async (code) => {
      session.updateCode(code);
      
      // Check for common mistakes
      const mistakes = detectMistakes(code);
      if (mistakes.length > 0) {
        const response = await orchestrator.route({
          type: 'code_written',
          code,
          mistakes,
        }, session.getContext());

        socket.emit('agent_response', response);
      }
    });

    // User runs code
    socket.on('execute_code', async (code) => {
      const result = await executeCode(code);

      if (result.error) {
        const response = await orchestrator.route({
          type: 'code_executed_with_error',
          code,
          error: result.error,
        }, session.getContext());

        socket.emit('agent_response', response);
      }

      socket.emit('execution_result', result);
    });

    // User inactive (might be stuck)
    socket.on('user_inactive', async () => {
      const response = await orchestrator.route({
        type: 'user_stuck',
      }, session.getContext());

      socket.emit('agent_response', response);
    });

    socket.on('disconnect', () => {
      SessionManager.cleanup(socket.id);
      console.log('User disconnected:', socket.id);
    });
  });
}
```

---

## 💾 Database Schema

```prisma
// database/schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model User {
  id            String   @id @default(uuid())
  email         String   @unique
  username      String   @unique
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
  
  // Learning data
  sessions      Session[]
  progress      Progress[]
  achievements  Achievement[]
}

model Session {
  id            String   @id @default(uuid())
  userId        String
  user          User     @relation(fields: [userId], references: [id])
  
  startedAt     DateTime @default(now())
  endedAt       DateTime?
  
  // Session data
  code          String   @default("")
  language      String   @default("python")
  messages      Json     // Array of {agent, content, timestamp}
  context       Json     // Current session context
  
  // Metrics
  linesWritten  Int      @default(0)
  mistakesCaught Int     @default(0)
  conceptsCovered String[] // Array of concept IDs
}

model Progress {
  id            String   @id @default(uuid())
  userId        String
  user          User     @relation(fields: [userId], references: [id])
  
  // Skill tracking
  skill         String   // e.g., "loops", "functions", "recursion"
  level         Int      @default(0) // 0-100
  lastPracticed DateTime @default(now())
  
  // Metadata
  timesAttempted Int     @default(0)
  timesSucceeded Int     @default(0)
  
  @@unique([userId, skill])
}

model Achievement {
  id            String   @id @default(uuid())
  userId        String
  user          User     @relation(fields: [userId], references: [id])
  
  type          String   // "first_code", "10_functions", "debugged_solo"
  unlockedAt    DateTime @default(now())
}

model CommonMistake {
  id            String   @id @default(uuid())
  pattern       String   // Regex pattern
  message       String   // Warning message
  suggestion    String   // How to fix
  severity      String   // "info", "warning", "error"
  category      String   // "syntax", "logic", "style", "performance"
  frequency     Int      @default(0) // How often it's triggered
}
```

---

## 🎯 Implementation Roadmap (48-72 Hours)

### Day 1: Core Foundation (0-24 hours)

**Hours 0-4: Setup & Scaffold**
- [ ] Initialize repos (frontend + backend)
- [ ] Set up development environment
- [ ] Configure Claude API
- [ ] Basic project structure

**Hours 4-12: Agent System**
- [ ] Implement Agent base class
- [ ] Create Teacher Agent with basic prompts
- [ ] Create Code Review Agent
- [ ] Create Debugging Agent
- [ ] Build Orchestrator routing logic

**Hours 12-20: Basic UI**
- [ ] Monaco editor integration
- [ ] Chat panel component
- [ ] WebSocket connection
- [ ] Basic styling with Tailwind

**Hours 20-24: Integration**
- [ ] Connect frontend to backend
- [ ] Test agent responses
- [ ] Fix critical bugs

### Day 2: Features & Polish (24-48 hours)

**Hours 24-32: Real-Time Features**
- [ ] Implement code analysis on typing pause
- [ ] Add mistake detection patterns (50+)
- [ ] Real-time proactive warnings
- [ ] Code execution service

**Hours 32-40: Learning Features**
- [ ] Session memory implementation
- [ ] Basic progress tracking
- [ ] Skill tree visualization (3 skills)
- [ ] Concept explainer (20 concepts)

**Hours 40-48: UX Polish**
- [ ] Agent avatars and colors
- [ ] Typing indicators
- [ ] Smooth transitions
- [ ] Responsive design
- [ ] Error handling

### Day 3: Demo Prep (48-72 hours)

**Hours 48-56: Testing**
- [ ] End-to-end testing
- [ ] Performance optimization
- [ ] Bug fixes
- [ ] Load testing

**Hours 56-64: Demo Content**
- [ ] Prepare demo script
- [ ] Create sample scenarios
- [ ] Record demo video
- [ ] Prepare slides

**Hours 64-72: Final Polish**
- [ ] README with screenshots
- [ ] Architecture documentation
- [ ] Deploy to production
- [ ] Practice demo presentation

---

## 🚀 Deployment Strategy

### Hosting
- **Frontend:** Vercel (free tier, instant deploy from GitHub)
- **Backend:** Railway or Render (free tier, supports WebSocket)
- **Database:** Railway PostgreSQL or Supabase (free tier)
- **Redis:** Upstash (free tier, 10k commands/day)

### Environment Variables
```env
# Backend
ANTHROPIC_API_KEY=sk-ant-...
DATABASE_URL=postgresql://...
REDIS_URL=redis://...
FRONTEND_URL=https://devmentor.vercel.app
PORT=3000

# Frontend
NEXT_PUBLIC_WS_URL=wss://devmentor-api.railway.app
NEXT_PUBLIC_API_URL=https://devmentor-api.railway.app
```

### CI/CD
```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Deploy to Vercel
        run: vercel --prod --token ${{ secrets.VERCEL_TOKEN }}
```

---

## 📊 Success Metrics (Demo Day)

### Technical Metrics
- ✅ Agent response time < 2 seconds
- ✅ WebSocket connection stable
- ✅ Zero crashes during demo
- ✅ All three agents clearly distinguishable
- ✅ Real-time code analysis works

### Demo Impact
- ✅ Judges understand concept in < 1 minute
- ✅ "Wow" moment achieved (proactive mistake catching)
- ✅ Judges try it themselves
- ✅ Questions are about scaling, not understanding
- ✅ Requests for GitHub link

---

## 🎨 UI/UX Mockups

### Main Interface Layout

```
┌─────────────────────────────────────────────────────────────────┐
│ DevMentor                                    Sarah | Progress    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────┐  ┌───────────────────────────┐   │
│  │                          │  │  💬 Agent Chat            │   │
│  │   Code Editor            │  │  ─────────────────────── │   │
│  │   (Monaco)               │  │                           │   │
│  │                          │  │  🧑‍🏫 Teacher: "Great!      │   │
│  │  def factorial(n):       │  │  Before we code, can     │   │
│  │      if n < 0:           │  │  you explain what a      │   │
│  │          return None     │  │  factorial is?"          │   │
│  │      if n == 0:          │  │                           │   │
│  │          return 1        │  │  You: "It's like 5*4*3   │   │
│  │      return n * factorial│  │  *2*1 for 5!"            │   │
│  │                          │  │                           │   │
│  │                          │  │  🧑‍🏫 Teacher: "Perfect!   │   │
│  │  ▶ Run  💡 Explain       │  │  Now let's code it..."   │   │
│  └──────────────────────────┘  │                           │   │
│                                 │  📝 Quick Actions:        │   │
│  ┌──────────────────────────┐  │  • "Continue coding"     │   │
│  │  Output                  │  │  • "Explain my code"     │   │
│  │  ────────────────────    │  │  • "I'm stuck"           │   │
│  │  120                     │  └───────────────────────────┘   │
│  │                          │                                   │
│  └──────────────────────────┘                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🏆 Winning the Hackathon: The Pitch

### 30-Second Elevator Pitch
"DevMentor is an AI pair programmer that teaches like a senior developer. Instead of giving you answers, it asks YOU questions. Instead of explaining after you're stuck, it catches mistakes BEFORE you make them. It's the mentor every solo learner wishes they had - and it's powered by a multi-agent AI system that adapts to your learning style in real-time."

### 2-Minute Demo Script
1. **Show the problem** (15 sec): "Learning to code alone is hard. No one to ask questions at midnight."
2. **Introduce DevMentor** (15 sec): "DevMentor is your AI pair programmer with 3 specialized agents."
3. **Live coding demo** (60 sec):
   - Start writing a function
   - Teacher asks guiding questions
   - Code Review catches mistake proactively
   - User fixes it
   - Debugger helps when code fails
4. **Show progress tracking** (15 sec): "And it tracks your growth over time"
5. **Impact statement** (15 sec): "This democratizes mentorship for millions of solo learners"

### Key Differentiators to Emphasize
1. **Multi-agent system** (judges love agentic AI in 2025)
2. **Proactive, not reactive** (unique approach)
3. **Teaches thinking, not just coding** (educational impact)
4. **Real-time pair programming** (technical innovation)
5. **Buildable and scalable** (not just a concept)

---

**Document Status:** Ready for Implementation  
**Last Updated:** February 14, 2026  
**Next Step:** Start coding! 🚀
