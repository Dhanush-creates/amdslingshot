# Design Document: AURA AI

## Overview

AURA AI is a 3D Generative AI workspace that bridges the gap between conceptual thinking and software implementation through an interactive visual interface. The system architecture consists of six primary layers: User Layer, Frontend Layer (React + Three.js), AI Orchestrator Layer, Backend Layer (FastAPI), Database Layer (PostgreSQL + Redis), and Infrastructure Layer (AMD-optimized). The design emphasizes real-time bidirectional synchronization between visual 3D representations and executable code, leveraging AMD hardware acceleration for optimal performance.

The core innovation lies in the Sync Engine, which maintains consistency between the 3D Mind Map and generated code through event-driven architecture and operational transformation algorithms. Users interact with a force-directed 3D graph where nodes represent architectural components (APIs, data models, UI components, services) and edges represent relationships (dependencies, data flow, composition). All interactions trigger AI-powered code generation while maintaining architectural integrity.

## Architecture

### System Layers

```mermaid
graph TB
    subgraph "User Layer"
        U[User Browser]
    end
    
    subgraph "Frontend Layer"
        R[React Application]
        T[Three.js 3D Engine]
        WS[WebSocket Client]
        SC[State Management - Zustand]
    end
    
    subgraph "AI Orchestrator Layer"
        AO[AI Orchestrator Service]
        LG[LangGraph Workflow Engine]
        PM[Prompt Manager]
        MC[Model Cache]
    end
    
    subgraph "Backend Layer"
        API[FastAPI Server]
        SE[Sync Engine]
        CG[Code Generator]
        ED[Error Detector]
        VH[Version History Manager]
    end
    
    subgraph "Database Layer"
        PG[(PostgreSQL)]
        RD[(Redis Cache)]
        S3[(Object Storage)]
    end
    
    subgraph "Infrastructure Layer"
        AMD[AMD Ryzen AI NPU]
        GPU[AMD Radeon GPU]
        LB[Load Balancer]
        MQ[Message Queue - RabbitMQ]
    end
    
    U --> R
    R --> T
    R --> WS
    R --> SC
    WS --> API
    API --> AO
    API --> SE
    API --> CG
    API --> ED
    API --> VH
    AO --> LG
    AO --> PM
    AO --> MC
    API --> PG
    API --> RD
    API --> S3
    API --> MQ
    AO --> AMD
    T --> GPU
    LB --> API
```

### Data Flow

1. **User Input Flow**: User enters natural language → Frontend sends to Backend API → AI Orchestrator processes with LangChain → Structured intent extracted → Sync Engine creates node specification → Mind Map updates → Code Generator produces code → Response streams back to Frontend

2. **Visual Manipulation Flow**: User drags/modifies node → Frontend captures event → WebSocket sends delta to Backend → Sync Engine validates change → Code Generator updates affected code → Error Detector validates → Changes broadcast to all connected clients → UI updates

3. **Code Edit Flow**: User edits code directly → Frontend sends code delta → Sync Engine parses AST → Node properties extracted → Mind Map updates → Validation runs → Conflicts resolved or flagged

4. **Collaboration Flow**: User A modifies node → Backend broadcasts via WebSocket → User B receives update → Operational Transformation applied → UI updates with conflict resolution if needed

## Components and Interfaces

### Frontend Components

#### 1. Three.js 3D Mind Map Engine

**Responsibilities:**
- Render 3D node graph using WebGL
- Handle user interactions (pan, zoom, rotate, select, drag)
- Implement force-directed layout algorithm
- Manage node animations and transitions
- Optimize rendering performance for 100+ nodes

**Key Classes:**
- `MindMapScene`: Main Three.js scene manager
- `NodeRenderer`: Renders individual node geometries with custom shaders
- `EdgeRenderer`: Renders connections using line geometries or tubes
- `InteractionController`: Handles raycasting and user input
- `LayoutEngine`: Implements force-directed graph layout (d3-force-3d)
- `CameraController`: Manages camera movement and focus

**Interfaces:**
```typescript
interface Node3D {
  id: string;
  type: NodeType; // 'api' | 'data_model' | 'ui_component' | 'service' | 'function'
  position: Vector3;
  properties: Record<string, any>;
  status: 'valid' | 'error' | 'warning' | 'generating';
  metadata: NodeMetadata;
}

interface Edge3D {
  id: string;
  source: string;
  target: string;
  type: EdgeType; // 'dependency' | 'data_flow' | 'composition' | 'inheritance'
  properties: Record<string, any>;
}

interface MindMapState {
  nodes: Map<string, Node3D>;
  edges: Map<string, Edge3D>;
  selectedNodes: Set<string>;
  camera: CameraState;
  layout: LayoutConfig;
}
```

**AMD GPU Optimization:**
- Use instanced rendering for similar node types
- Implement frustum culling to skip off-screen nodes
- Leverage AMD FidelityFX for post-processing effects
- Use compute shaders for physics calculations on AMD RDNA architecture

#### 2. React Application Shell

**Responsibilities:**
- Manage application routing and navigation
- Coordinate between 3D view and code editor
- Handle authentication and session management
- Provide UI controls and panels

**Key Components:**
- `App`: Root component with routing
- `Dashboard`: Main workspace layout
- `CodeEditor`: Monaco-based code editor with syntax highlighting
- `PropertyPanel`: Node property editor
- `TemplateGallery`: Template selection interface
- `CollaborationBar`: Real-time user presence indicators
- `APIKeyManager`: Secure credential management UI

#### 3. State Management (Zustand)

**Stores:**
- `projectStore`: Current project state, nodes, edges
- `uiStore`: UI state, panels, modals, selections
- `authStore`: User authentication state
- `collaborationStore`: Connected users, locks, cursors
- `historyStore`: Undo/redo stack

**Synchronization Strategy:**
- WebSocket updates trigger store mutations
- Optimistic updates for local changes
- Conflict resolution using operational transformation
- Periodic full state reconciliation

### Backend Components

#### 1. FastAPI Server

**Responsibilities:**
- Expose REST API for CRUD operations
- Handle WebSocket connections for real-time updates
- Authenticate and authorize requests
- Route requests to appropriate services

**Key Endpoints:**
```python
# Project Management
POST   /api/projects                    # Create new project
GET    /api/projects/{id}               # Get project details
PUT    /api/projects/{id}               # Update project
DELETE /api/projects/{id}               # Delete project
GET    /api/projects/{id}/export        # Export project as ZIP

# Node Operations
POST   /api/projects/{id}/nodes         # Create node
PUT    /api/projects/{id}/nodes/{node_id}  # Update node
DELETE /api/projects/{id}/nodes/{node_id}  # Delete node

# Code Generation
POST   /api/generate/code               # Generate code from description
POST   /api/generate/node               # Generate node from code

# Collaboration
WS     /ws/projects/{id}                # WebSocket for real-time updates

# Templates
GET    /api/templates                   # List templates
POST   /api/templates                   # Create template
GET    /api/templates/{id}              # Get template details

# API Key Management
POST   /api/keys                        # Store encrypted API key
GET    /api/keys                        # List keys (masked)
DELETE /api/keys/{id}                   # Delete key
```

**Middleware:**
- Authentication (JWT)
- Rate limiting (100 req/min per user)
- CORS configuration
- Request logging
- Error handling

#### 2. Sync Engine

**Responsibilities:**
- Maintain bidirectional consistency between nodes and code
- Parse code to extract node properties
- Generate code from node specifications
- Detect and resolve conflicts
- Validate architectural constraints

**Core Algorithms:**

**Code-to-Node Parsing:**
```python
class CodeParser:
    def parse_to_nodes(self, code: str, language: str) -> List[NodeSpec]:
        # Use tree-sitter for language-agnostic parsing
        tree = self.parser.parse(code)
        nodes = []
        
        # Extract functions, classes, API routes, etc.
        for node in tree.root_node.children:
            if node.type in ['function_definition', 'class_definition']:
                nodes.append(self._extract_node_spec(node))
        
        return nodes
    
    def _extract_node_spec(self, ast_node) -> NodeSpec:
        # Extract name, parameters, return type, docstring
        # Infer node type from context
        # Extract dependencies from imports and calls
        pass
```

**Node-to-Code Generation:**
```python
class CodeGenerator:
    def generate_code(self, node: NodeSpec, language: str) -> str:
        # Use language-specific templates
        template = self.template_manager.get_template(node.type, language)
        
        # Fill template with node properties
        code = template.render(
            name=node.name,
            parameters=node.parameters,
            return_type=node.return_type,
            body=self._generate_body(node)
        )
        
        return code
```

**Operational Transformation for Conflict Resolution:**
```python
class OperationalTransform:
    def transform(self, op1: Operation, op2: Operation) -> Tuple[Operation, Operation]:
        # Transform concurrent operations to maintain consistency
        # Handle insert, delete, update operations
        # Preserve user intent while resolving conflicts
        pass
```

**Interfaces:**
```python
class NodeSpec(BaseModel):
    id: str
    type: NodeType
    name: str
    properties: Dict[str, Any]
    dependencies: List[str]
    code_location: CodeLocation
    
class CodeLocation(BaseModel):
    file_path: str
    start_line: int
    end_line: int
    
class SyncOperation(BaseModel):
    type: Literal['node_update', 'code_update', 'node_create', 'node_delete']
    node_id: str
    changes: Dict[str, Any]
    timestamp: datetime
    user_id: str
```

#### 3. AI Orchestrator

**Responsibilities:**
- Coordinate multiple AI model calls
- Manage prompt templates and context
- Implement retry logic and fallbacks
- Cache AI responses
- Optimize for AMD NPU acceleration

**Architecture:**
```python
class AIOrchestrator:
    def __init__(self):
        self.llm_client = self._init_llm_client()
        self.prompt_manager = PromptManager()
        self.cache = RedisCache()
        self.amd_optimizer = AMDNPUOptimizer()
    
    async def process_user_input(self, input_text: str, context: ProjectContext) -> IntentResult:
        # Extract intent using LangChain
        prompt = self.prompt_manager.get_prompt('intent_extraction', context)
        
        # Check cache
        cache_key = self._generate_cache_key(input_text, context)
        if cached := await self.cache.get(cache_key):
            return cached
        
        # Run inference with AMD NPU if available
        if self.amd_optimizer.is_available():
            result = await self.amd_optimizer.run_inference(prompt, input_text)
        else:
            result = await self.llm_client.generate(prompt, input_text)
        
        # Cache result
        await self.cache.set(cache_key, result, ttl=3600)
        
        return result
    
    async def generate_code(self, node_spec: NodeSpec, language: str) -> str:
        # Generate code using AI with language-specific prompts
        prompt = self.prompt_manager.get_prompt('code_generation', {
            'node_type': node_spec.type,
            'language': language,
            'spec': node_spec.dict()
        })
        
        code = await self.llm_client.generate(prompt)
        
        # Validate and format
        formatted_code = self._format_code(code, language)
        
        return formatted_code
```

**LangGraph Workflow:**
```python
from langgraph.graph import StateGraph, END

class CodeGenerationWorkflow:
    def __init__(self):
        self.graph = StateGraph()
        self._build_graph()
    
    def _build_graph(self):
        # Define workflow nodes
        self.graph.add_node("parse_intent", self.parse_intent)
        self.graph.add_node("validate_spec", self.validate_spec)
        self.graph.add_node("generate_code", self.generate_code)
        self.graph.add_node("validate_code", self.validate_code)
        self.graph.add_node("handle_error", self.handle_error)
        
        # Define edges
        self.graph.add_edge("parse_intent", "validate_spec")
        self.graph.add_conditional_edges(
            "validate_spec",
            self.should_generate,
            {True: "generate_code", False: "handle_error"}
        )
        self.graph.add_edge("generate_code", "validate_code")
        self.graph.add_conditional_edges(
            "validate_code",
            self.is_valid,
            {True: END, False: "handle_error"}
        )
```

**AMD NPU Optimization:**
```python
class AMDNPUOptimizer:
    def __init__(self):
        self.npu_available = self._detect_amd_npu()
        if self.npu_available:
            self.runtime = self._init_ryzen_ai_runtime()
    
    def _detect_amd_npu(self) -> bool:
        # Detect AMD Ryzen AI NPU using platform detection
        # Check for AMD IPU/NPU device
        pass
    
    async def run_inference(self, prompt: str, input_text: str) -> str:
        # Optimize model for AMD NPU
        # Use quantized models (INT8) for faster inference
        # Batch multiple requests when possible
        # Leverage AMD's Ryzen AI Software for model optimization
        pass
```

#### 4. Error Detector

**Responsibilities:**
- Validate generated code syntax
- Detect logical errors and anti-patterns
- Check dependency compatibility
- Identify security vulnerabilities
- Provide fix suggestions

**Implementation:**
```python
class ErrorDetector:
    def __init__(self):
        self.syntax_validators = {
            'python': PythonValidator(),
            'typescript': TypeScriptValidator(),
            'javascript': JavaScriptValidator()
        }
        self.linters = {
            'python': Pylint(),
            'typescript': ESLint(),
        }
        self.security_scanner = SecurityScanner()
    
    async def validate_code(self, code: str, language: str) -> ValidationResult:
        errors = []
        warnings = []
        
        # Syntax validation
        syntax_errors = await self.syntax_validators[language].validate(code)
        errors.extend(syntax_errors)
        
        # Linting
        lint_issues = await self.linters[language].lint(code)
        warnings.extend(lint_issues)
        
        # Security scanning
        security_issues = await self.security_scanner.scan(code)
        errors.extend([i for i in security_issues if i.severity == 'high'])
        warnings.extend([i for i in security_issues if i.severity == 'medium'])
        
        return ValidationResult(
            is_valid=len(errors) == 0,
            errors=errors,
            warnings=warnings,
            suggestions=self._generate_suggestions(errors, warnings)
        )
    
    def _generate_suggestions(self, errors: List[Error], warnings: List[Warning]) -> List[Suggestion]:
        # Use AI to generate fix suggestions
        suggestions = []
        for error in errors:
            suggestion = self.ai_orchestrator.generate_fix(error)
            suggestions.append(suggestion)
        return suggestions
```

#### 5. Version History Manager

**Responsibilities:**
- Create automatic snapshots every 5 minutes
- Store version diffs efficiently
- Enable rollback to previous versions
- Provide version comparison

**Implementation:**
```python
class VersionHistoryManager:
    def __init__(self, db: Database, storage: ObjectStorage):
        self.db = db
        self.storage = storage
    
    async def create_snapshot(self, project_id: str) -> Version:
        # Get current project state
        project = await self.db.get_project(project_id)
        
        # Calculate diff from previous version
        previous_version = await self.get_latest_version(project_id)
        diff = self._calculate_diff(previous_version, project)
        
        # Store diff in object storage
        diff_key = f"versions/{project_id}/{uuid4()}.diff"
        await self.storage.put(diff_key, diff)
        
        # Create version record
        version = Version(
            project_id=project_id,
            diff_key=diff_key,
            timestamp=datetime.utcnow(),
            snapshot_type='auto'
        )
        
        await self.db.save_version(version)
        return version
    
    async def rollback(self, project_id: str, version_id: str) -> Project:
        # Get target version and all versions after it
        target_version = await self.db.get_version(version_id)
        versions_to_revert = await self.db.get_versions_after(project_id, target_version.timestamp)
        
        # Apply diffs in reverse
        current_state = await self.db.get_project(project_id)
        for version in reversed(versions_to_revert):
            diff = await self.storage.get(version.diff_key)
            current_state = self._apply_reverse_diff(current_state, diff)
        
        # Save rolled-back state
        await self.db.update_project(project_id, current_state)
        
        return current_state
```

### Database Layer

#### PostgreSQL Schema

```sql
-- Users table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_login TIMESTAMP
);

-- Projects table
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id UUID REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    language VARCHAR(50) NOT NULL,
    visibility VARCHAR(20) DEFAULT 'private', -- 'private', 'public', 'shared'
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    view_count INTEGER DEFAULT 0
);

-- Nodes table
CREATE TABLE nodes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
    type VARCHAR(50) NOT NULL, -- 'api', 'data_model', 'ui_component', 'service', 'function'
    name VARCHAR(255) NOT NULL,
    properties JSONB NOT NULL,
    position JSONB NOT NULL, -- {x, y, z}
    status VARCHAR(20) DEFAULT 'valid', -- 'valid', 'error', 'warning', 'generating'
    code_location JSONB, -- {file_path, start_line, end_line}
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Edges table
CREATE TABLE edges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
    source_node_id UUID REFERENCES nodes(id) ON DELETE CASCADE,
    target_node_id UUID REFERENCES nodes(id) ON DELETE CASCADE,
    type VARCHAR(50) NOT NULL, -- 'dependency', 'data_flow', 'composition', 'inheritance'
    properties JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Code files table
CREATE TABLE code_files (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
    file_path VARCHAR(500) NOT NULL,
    content TEXT NOT NULL,
    language VARCHAR(50) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(project_id, file_path)
);

-- Versions table
CREATE TABLE versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
    diff_key VARCHAR(500) NOT NULL, -- S3 key for diff storage
    snapshot_type VARCHAR(20) NOT NULL, -- 'auto', 'manual', 'checkpoint'
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by UUID REFERENCES users(id)
);

-- API keys table (encrypted)
CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    provider VARCHAR(50) NOT NULL, -- 'openai', 'anthropic', 'custom'
    encrypted_key TEXT NOT NULL,
    key_suffix VARCHAR(10) NOT NULL, -- Last 4 chars for display
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_used TIMESTAMP
);

-- Templates table
CREATE TABLE templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    category VARCHAR(100),
    preview_image_url VARCHAR(500),
    project_snapshot JSONB NOT NULL, -- Complete project structure
    created_by UUID REFERENCES users(id),
    is_official BOOLEAN DEFAULT FALSE,
    usage_count INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Collaboration sessions table
CREATE TABLE collaboration_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    connected_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_activity TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    cursor_position JSONB
);

-- Node locks table (for collaboration)
CREATE TABLE node_locks (
    node_id UUID REFERENCES nodes(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    locked_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (node_id)
);

-- Indexes for performance
CREATE INDEX idx_projects_owner ON projects(owner_id);
CREATE INDEX idx_nodes_project ON nodes(project_id);
CREATE INDEX idx_edges_project ON edges(project_id);
CREATE INDEX idx_edges_source ON edges(source_node_id);
CREATE INDEX idx_edges_target ON edges(target_node_id);
CREATE INDEX idx_code_files_project ON code_files(project_id);
CREATE INDEX idx_versions_project ON versions(project_id);
CREATE INDEX idx_collaboration_project ON collaboration_sessions(project_id);
CREATE INDEX idx_templates_category ON templates(category);
```

#### Redis Cache Structure

```
# Project cache (TTL: 5 minutes)
project:{project_id} -> JSON serialized project state

# Node cache (TTL: 5 minutes)
node:{node_id} -> JSON serialized node data

# AI response cache (TTL: 1 hour)
ai:intent:{hash} -> JSON serialized intent result
ai:code:{hash} -> Generated code string

# User session cache (TTL: 24 hours)
session:{session_id} -> User session data

# Rate limiting (TTL: 1 minute)
ratelimit:{user_id}:{endpoint} -> Request count

# Collaboration presence (TTL: 30 seconds)
presence:{project_id} -> Set of connected user IDs

# Node locks (TTL: 5 minutes)
lock:{node_id} -> User ID holding lock
```

## Data Models

### Core Domain Models

```typescript
// Frontend TypeScript Models

enum NodeType {
  API = 'api',
  DATA_MODEL = 'data_model',
  UI_COMPONENT = 'ui_component',
  SERVICE = 'service',
  FUNCTION = 'function',
  DATABASE = 'database',
  EXTERNAL_API = 'external_api'
}

enum EdgeType {
  DEPENDENCY = 'dependency',
  DATA_FLOW = 'data_flow',
  COMPOSITION = 'composition',
  INHERITANCE = 'inheritance',
  API_CALL = 'api_call'
}

interface Node {
  id: string;
  projectId: string;
  type: NodeType;
  name: string;
  properties: NodeProperties;
  position: Vector3;
  status: 'valid' | 'error' | 'warning' | 'generating';
  codeLocation?: CodeLocation;
  metadata: NodeMetadata;
  createdAt: Date;
  updatedAt: Date;
}

interface NodeProperties {
  // Common properties
  description?: string;
  tags?: string[];
  
  // API-specific
  httpMethod?: 'GET' | 'POST' | 'PUT' | 'DELETE' | 'PATCH';
  path?: string;
  requestSchema?: JSONSchema;
  responseSchema?: JSONSchema;
  
  // Data model-specific
  fields?: Field[];
  relationships?: Relationship[];
  
  // UI component-specific
  componentType?: 'page' | 'component' | 'layout';
  props?: PropDefinition[];
  state?: StateDefinition[];
  
  // Function-specific
  parameters?: Parameter[];
  returnType?: string;
  isAsync?: boolean;
}

interface Field {
  name: string;
  type: string;
  required: boolean;
  defaultValue?: any;
  validation?: ValidationRule[];
}

interface Relationship {
  type: 'one-to-one' | 'one-to-many' | 'many-to-many';
  targetModel: string;
  foreignKey?: string;
}

interface Edge {
  id: string;
  projectId: string;
  sourceNodeId: string;
  targetNodeId: string;
  type: EdgeType;
  properties: EdgeProperties;
  createdAt: Date;
}

interface EdgeProperties {
  label?: string;
  weight?: number;
  dataSchema?: JSONSchema;
}

interface Project {
  id: string;
  ownerId: string;
  name: string;
  description?: string;
  language: 'python' | 'typescript' | 'javascript';
  visibility: 'private' | 'public' | 'shared';
  nodes: Node[];
  edges: Edge[];
  codeFiles: CodeFile[];
  createdAt: Date;
  updatedAt: Date;
}

interface CodeFile {
  id: string;
  projectId: string;
  filePath: string;
  content: string;
  language: string;
  createdAt: Date;
  updatedAt: Date;
}

interface Version {
  id: string;
  projectId: string;
  diffKey: string;
  snapshotType: 'auto' | 'manual' | 'checkpoint';
  createdAt: Date;
  createdBy?: string;
}

interface Template {
  id: string;
  name: string;
  description: string;
  category: string;
  previewImageUrl: string;
  projectSnapshot: ProjectSnapshot;
  createdBy?: string;
  isOfficial: boolean;
  usageCount: number;
  createdAt: Date;
}

interface CollaborationSession {
  id: string;
  projectId: string;
  userId: string;
  username: string;
  cursorPosition?: Vector3;
  selectedNodes: string[];
  connectedAt: Date;
  lastActivity: Date;
}
```

```python
# Backend Python Models

from pydantic import BaseModel, Field
from typing import Optional, List, Dict, Any, Literal
from datetime import datetime
from enum import Enum

class NodeType(str, Enum):
    API = "api"
    DATA_MODEL = "data_model"
    UI_COMPONENT = "ui_component"
    SERVICE = "service"
    FUNCTION = "function"
    DATABASE = "database"
    EXTERNAL_API = "external_api"

class EdgeType(str, Enum):
    DEPENDENCY = "dependency"
    DATA_FLOW = "data_flow"
    COMPOSITION = "composition"
    INHERITANCE = "inheritance"
    API_CALL = "api_call"

class Vector3(BaseModel):
    x: float
    y: float
    z: float

class CodeLocation(BaseModel):
    file_path: str
    start_line: int
    end_line: int

class NodeProperties(BaseModel):
    description: Optional[str] = None
    tags: List[str] = Field(default_factory=list)
    
    # API-specific
    http_method: Optional[Literal["GET", "POST", "PUT", "DELETE", "PATCH"]] = None
    path: Optional[str] = None
    request_schema: Optional[Dict[str, Any]] = None
    response_schema: Optional[Dict[str, Any]] = None
    
    # Data model-specific
    fields: Optional[List[Dict[str, Any]]] = None
    relationships: Optional[List[Dict[str, Any]]] = None
    
    # UI component-specific
    component_type: Optional[Literal["page", "component", "layout"]] = None
    props: Optional[List[Dict[str, Any]]] = None
    state: Optional[List[Dict[str, Any]]] = None
    
    # Function-specific
    parameters: Optional[List[Dict[str, Any]]] = None
    return_type: Optional[str] = None
    is_async: bool = False

class Node(BaseModel):
    id: str
    project_id: str
    type: NodeType
    name: str
    properties: NodeProperties
    position: Vector3
    status: Literal["valid", "error", "warning", "generating"] = "valid"
    code_location: Optional[CodeLocation] = None
    created_at: datetime
    updated_at: datetime

class Edge(BaseModel):
    id: str
    project_id: str
    source_node_id: str
    target_node_id: str
    type: EdgeType
    properties: Dict[str, Any] = Field(default_factory=dict)
    created_at: datetime

class CodeFile(BaseModel):
    id: str
    project_id: str
    file_path: str
    content: str
    language: str
    created_at: datetime
    updated_at: datetime

class Project(BaseModel):
    id: str
    owner_id: str
    name: str
    description: Optional[str] = None
    language: Literal["python", "typescript", "javascript"]
    visibility: Literal["private", "public", "shared"] = "private"
    created_at: datetime
    updated_at: datetime
    view_count: int = 0

class ValidationError(BaseModel):
    line: int
    column: int
    message: str
    severity: Literal["error", "warning"]
    code: str
    suggestion: Optional[str] = None

class ValidationResult(BaseModel):
    is_valid: bool
    errors: List[ValidationError]
    warnings: List[ValidationError]
    suggestions: List[str]

class IntentResult(BaseModel):
    intent_type: Literal["create_node", "modify_node", "create_connection", "generate_code"]
    entities: Dict[str, Any]
    confidence: float
    clarification_needed: bool
    clarification_questions: List[str] = Field(default_factory=list)
```


## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property Reflection

After analyzing all acceptance criteria, I've identified several areas where properties can be consolidated:

1. **Performance properties** (parsing time, generation time, sync time, rendering FPS) can be grouped by operation type
2. **Round-trip properties** (node ↔ code synchronization) are already well-defined in 4.6
3. **UI interaction properties** (hover, click, drag) share common patterns
4. **Security properties** (encryption, masking, no exposure) for API keys can be combined
5. **Export completeness properties** (README, dependencies, structure) can be unified
6. **Collaboration broadcast properties** share timing and consistency requirements

The following properties represent the unique, non-redundant correctness guarantees:

### Core Synchronization Properties

**Property 1: Bidirectional Sync Round-Trip Consistency**
*For any* valid node configuration, modifying the node properties then immediately reverting those changes should restore the original code state exactly, and conversely, modifying code then reverting should restore the original node state.
**Validates: Requirements 4.6**

**Property 2: Node-to-Code Generation Consistency**
*For any* node created in the Mind_Map, the Code_Generator should produce syntactically valid code in the selected language that, when parsed back, produces a node with equivalent properties.
**Validates: Requirements 3.1, 3.4, 4.1**

**Property 3: Code-to-Node Parsing Consistency**
*For any* valid code file in a supported language, parsing should extract nodes whose code generation produces functionally equivalent code.
**Validates: Requirements 4.2**

**Property 4: Connection Integrity**
*For any* two connected nodes, the generated integration code should maintain proper dependency ordering and data flow such that the source node's outputs match the target node's expected inputs.
**Validates: Requirements 3.3, 4.4**

### Input Processing Properties

**Property 5: Input Processing Performance**
*For any* user input description up to 1000 characters, the User_Input_Processor should parse and extract actionable components within 2 seconds.
**Validates: Requirements 1.1, 1.4**

**Property 6: Entity Extraction Completeness**
*For any* processed input, all explicitly mentioned entities (components, relationships, data models, UI elements) should be identified and categorized correctly.
**Validates: Requirements 1.5**

**Property 7: Ambiguity Detection**
*For any* input containing ambiguous terms (multiple possible interpretations), the User_Input_Processor should request clarification rather than making assumptions.
**Validates: Requirements 1.2**

### 3D Visualization Properties

**Property 8: Node Layout Non-Overlap**
*For any* set of nodes in the Mind_Map, the force-directed layout algorithm should position them such that no two node bounding volumes intersect.
**Validates: Requirements 2.2**

**Property 9: Real-Time Position Updates**
*For any* node drag operation, the node's position should update in real-time and all connected edges should maintain valid connections to the node's new position.
**Validates: Requirements 2.5**

**Property 10: Rendering Performance**
*For any* Mind_Map with up to 100 nodes, the rendering engine should maintain at least 30 frames per second during all interactions.
**Validates: Requirements 2.6**

**Property 11: Interaction Responsiveness**
*For any* node interaction (hover, click, drag), the Mind_Map should respond with appropriate visual feedback within 100 milliseconds.
**Validates: Requirements 2.3, 2.4**

### Code Generation Properties

**Property 12: Language-Specific Code Quality**
*For any* generated code in a supported language (Python, JavaScript, TypeScript), the code should pass language-specific linters and follow established style guidelines.
**Validates: Requirements 3.2, 15.3**

**Property 13: Dependency Correctness**
*For any* generated code file, all imported modules and dependencies should be valid, available, and correctly added to the project's dependency manifest.
**Validates: Requirements 19.1, 19.4**

**Property 14: Test Generation Completeness**
*For any* generated implementation code, the Code_Generator should produce corresponding test files with at least one test case per public function or method.
**Validates: Requirements 15.6**

### Error Detection Properties

**Property 15: Syntax Error Detection Performance**
*For any* code containing syntax errors, the Error_Detector should identify all errors within 2 seconds of the code change.
**Validates: Requirements 6.1**

**Property 16: Error Visualization Consistency**
*For any* code error detected, the corresponding node in the Mind_Map should be highlighted with error status, and hovering should display the error message and suggestions.
**Validates: Requirements 6.2, 6.3**

**Property 17: Dependency Validation**
*For any* project graph, the Error_Detector should identify and flag all missing dependencies and circular dependency chains.
**Validates: Requirements 6.6**

**Property 18: Error Resolution Feedback**
*For any* node with errors, when all errors are resolved, the node status should update to valid (green) within 1 second.
**Validates: Requirements 6.5**

### Security Properties

**Property 19: API Key Encryption**
*For any* API key stored by the API_Key_Manager, the stored value should be encrypted using AES-256 and the plaintext key should never appear in logs, displays, or error messages.
**Validates: Requirements 7.1, 7.2, 7.5**

**Property 20: API Key Masking**
*For any* request to view stored API keys, only the last 4 characters should be displayed, with the rest masked.
**Validates: Requirements 7.3**

**Property 21: API Key Deletion Completeness**
*For any* API key deletion request, the key should be permanently removed from all storage locations within 1 second and should not be recoverable.
**Validates: Requirements 7.6**

### Performance Optimization Properties

**Property 22: AMD NPU Utilization**
*For any* AI inference task running on a system with AMD Ryzen AI NPU, the AMD_Optimizer should utilize the NPU and achieve at least 30% latency reduction compared to CPU-only execution.
**Validates: Requirements 8.1, 8.5**

**Property 23: Hardware Fallback Graceful Degradation**
*For any* system without AMD hardware, the AURA_System should detect the absence and fall back to CPU-based processing without errors or functionality loss.
**Validates: Requirements 8.4**

**Property 24: Batch Processing Optimization**
*For any* sequence of multiple AI requests within a 1-second window, the AMD_Optimizer should batch them into a single inference call when possible.
**Validates: Requirements 8.6**

### Export Properties

**Property 25: Export Completeness**
*For any* project export, the generated ZIP archive should contain all code files, a README with setup instructions, a dependency manifest (package.json or requirements.txt), and preserve all comments and documentation.
**Validates: Requirements 9.2, 9.3, 9.4, 9.5**

**Property 26: Export Performance**
*For any* project export request, the complete ZIP archive should be generated within 10 seconds regardless of project size up to 500 nodes.
**Validates: Requirements 9.1**

### Collaboration Properties

**Property 27: Real-Time Broadcast Latency**
*For any* user action (node selection, property modification) in a collaborative session, the action should be broadcast to all other connected users within 500 milliseconds.
**Validates: Requirements 10.2**

**Property 28: Node Locking Consistency**
*For any* node being edited by a user, the node should be locked for other users, display an editing indicator, and prevent concurrent modifications until the lock is released.
**Validates: Requirements 10.3**

**Property 29: Presence Indicator Accuracy**
*For any* collaborative session, all connected users should see accurate avatars and cursor positions for all other users, and disconnected users should have their indicators removed within 2 seconds.
**Validates: Requirements 10.1, 10.4, 10.5**

**Property 30: Concurrent User Capacity**
*For any* project with up to 10 concurrent users, the system should maintain all performance guarantees (response time, FPS, broadcast latency) without degradation.
**Validates: Requirements 10.6**

### Template Properties

**Property 31: Template Instantiation Performance**
*For any* template selection, the complete Mind_Map and code should be instantiated and ready for editing within 5 seconds.
**Validates: Requirements 11.3**

**Property 32: Template Customization Immediacy**
*For any* instantiated template, all components should be immediately editable without requiring additional loading or processing.
**Validates: Requirements 11.6**

**Property 33: Custom Template Creation**
*For any* user project, the user should be able to save it as a custom template that preserves the complete structure, code, and configuration.
**Validates: Requirements 11.5**

### Version Control Properties

**Property 34: Automatic Snapshot Frequency**
*For any* project with active changes, the system should create automatic version snapshots at 5-minute intervals.
**Validates: Requirements 12.1**

**Property 35: Rollback Safety**
*For any* rollback operation, the system should create a snapshot of the current state before reverting, ensuring no data loss.
**Validates: Requirements 12.6**

**Property 36: Version Visualization Accuracy**
*For any* previous version selected from history, the Mind_Map should render exactly as it appeared at that timestamp, with all nodes, edges, and properties matching the historical state.
**Validates: Requirements 12.3**

**Property 37: Version Retention Policy**
*For any* version created, it should be retained and accessible for at least 30 days from creation.
**Validates: Requirements 12.5**

### Sharing and Community Properties

**Property 38: Share Link Uniqueness**
*For any* published project, the generated shareable link should be unique, persistent, and provide read-only access to the project.
**Validates: Requirements 13.1, 13.2**

**Property 39: Project Cloning Completeness**
*For any* shared project that a user clones, the cloned copy should contain all nodes, edges, code, and configuration from the original, and should be fully editable in the user's workspace.
**Validates: Requirements 13.3**

**Property 40: Search Result Relevance**
*For any* search query in the community projects interface, all returned results should match the search terms in either tags, keywords, name, or description.
**Validates: Requirements 13.5**

### Data Model and API Properties

**Property 41: Data Model Visualization Completeness**
*For any* data model node, the visualization should display all fields with their types, constraints, and all relationships with appropriate visual indicators.
**Validates: Requirements 16.1, 16.2**

**Property 42: Migration Script Correctness**
*For any* data model node, the generated migration script should create a database schema that matches the node's field definitions, relationships, indexes, and constraints.
**Validates: Requirements 16.3, 16.6**

**Property 43: Breaking Change Detection**
*For any* modification to a data model node, if the change would require data migration (field deletion, type change, constraint addition), the system should detect and warn the user.
**Validates: Requirements 16.4**

**Property 44: API Endpoint Code Completeness**
*For any* API endpoint node, the generated code should include route handler, request validation, response serialization, and error handling.
**Validates: Requirements 17.2**

**Property 45: CRUD Generation from Connections**
*For any* API endpoint node connected to a data model node, the Code_Generator should automatically generate complete CRUD operations (Create, Read, Update, Delete) for that model.
**Validates: Requirements 17.6**

**Property 46: OpenAPI Documentation Generation**
*For any* set of API endpoint nodes in a project, the system should generate valid OpenAPI 3.0 specification documenting all endpoints, parameters, request bodies, and responses.
**Validates: Requirements 17.5**

### UI Component Properties

**Property 47: UI Component Preview Performance**
*For any* generated UI component, a live preview should render within 3 seconds of generation.
**Validates: Requirements 18.2**

**Property 48: Real-Time Preview Updates**
*For any* UI component property modification, the live preview should update within 500 milliseconds to reflect the change.
**Validates: Requirements 18.4**

**Property 49: Accessibility Compliance**
*For any* generated HTML UI component, the code should include appropriate ARIA attributes, semantic HTML elements, and keyboard navigation support.
**Validates: Requirements 18.5**

**Property 50: Form Validation Completeness**
*For any* generated form component, the code should include client-side validation logic, error message display, and error handling for all input fields.
**Validates: Requirements 18.6**

### Dependency Management Properties

**Property 51: Dependency Conflict Detection**
*For any* project with multiple dependencies, when a conflict is detected (incompatible version requirements), the system should identify the conflict and suggest compatible version ranges.
**Validates: Requirements 19.2**

**Property 52: Security Vulnerability Scanning**
*For any* dependency added to a project, the system should check for known security vulnerabilities and warn the user if any are found.
**Validates: Requirements 19.3**

**Property 53: Lock File Generation**
*For any* project with dependencies, the system should generate and maintain a lock file (package-lock.json or poetry.lock) ensuring reproducible builds.
**Validates: Requirements 19.5**

### Multi-Tenancy Properties

**Property 54: Data Isolation**
*For any* two different users, their project data, nodes, code files, and API keys should be completely isolated with no cross-user data access possible.
**Validates: Requirements 20.1**

**Property 55: Resource Quota Enforcement**
*For any* user with defined resource quotas (storage, API calls, projects), the system should prevent resource consumption beyond the quota and display clear messages when limits are reached.
**Validates: Requirements 20.3, 20.4**

**Property 56: Horizontal Scalability**
*For any* deployment with multiple server instances, the system should distribute load evenly and maintain all functionality and performance guarantees regardless of which instance handles a request.
**Validates: Requirements 20.6**


## Error Handling

### Error Categories

#### 1. User Input Errors
- **Invalid Descriptions**: When user input cannot be parsed into actionable components
  - Response: Display clarification questions
  - Recovery: Allow user to refine input
  
- **Ambiguous Intent**: When multiple interpretations are possible
  - Response: Present options for user selection
  - Recovery: User selects intended interpretation

#### 2. Code Generation Errors
- **Syntax Errors**: Generated code fails language validation
  - Response: Retry generation with corrected prompt
  - Recovery: Up to 3 retry attempts, then flag for manual review
  
- **Dependency Conflicts**: Required packages have incompatible versions
  - Response: Suggest compatible version ranges
  - Recovery: User selects version or system auto-resolves

- **Template Errors**: Code template rendering fails
  - Response: Log error with context, use fallback template
  - Recovery: Generate minimal valid code structure

#### 3. Synchronization Errors
- **Parse Failures**: Code cannot be parsed to extract nodes
  - Response: Mark affected nodes as "out of sync"
  - Recovery: User can manually trigger re-sync or edit code

- **Concurrent Modification Conflicts**: Multiple users edit same node
  - Response: Use operational transformation to merge changes
  - Recovery: If merge fails, prompt users to resolve manually

- **State Inconsistency**: Node state doesn't match code state
  - Response: Trigger full reconciliation
  - Recovery: Use code as source of truth, update nodes

#### 4. Performance Errors
- **Timeout Errors**: Operations exceed time limits
  - Response: Cancel operation, notify user
  - Recovery: Suggest breaking into smaller operations

- **Memory Errors**: System runs out of memory
  - Response: Gracefully degrade (reduce node count, simplify rendering)
  - Recovery: Prompt user to reduce project complexity

- **Rate Limit Errors**: External API rate limits exceeded
  - Response: Queue requests, implement exponential backoff
  - Recovery: Process queued requests when rate limit resets

#### 5. Collaboration Errors
- **Connection Loss**: WebSocket disconnects
  - Response: Attempt automatic reconnection with exponential backoff
  - Recovery: Restore session state on reconnection

- **Lock Timeout**: User holds lock too long
  - Response: Auto-release lock after 5 minutes of inactivity
  - Recovery: Notify user that lock was released

- **Broadcast Failure**: Update fails to reach all users
  - Response: Retry broadcast up to 3 times
  - Recovery: Force full state sync for affected users

#### 6. Storage Errors
- **Database Errors**: PostgreSQL connection or query failures
  - Response: Retry with exponential backoff
  - Recovery: Use cached data if available, queue writes

- **Cache Errors**: Redis unavailable
  - Response: Continue without cache (degraded performance)
  - Recovery: Reconnect to Redis when available

- **Object Storage Errors**: S3/storage service unavailable
  - Response: Queue uploads, retry later
  - Recovery: Process queue when service recovers

### Error Handling Patterns

```python
# Retry with exponential backoff
async def retry_with_backoff(func, max_attempts=3, base_delay=1):
    for attempt in range(max_attempts):
        try:
            return await func()
        except RetryableError as e:
            if attempt == max_attempts - 1:
                raise
            delay = base_delay * (2 ** attempt)
            await asyncio.sleep(delay)

# Circuit breaker for external services
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.last_failure_time = None
        self.state = "closed"  # closed, open, half-open
    
    async def call(self, func):
        if self.state == "open":
            if time.time() - self.last_failure_time > self.timeout:
                self.state = "half-open"
            else:
                raise CircuitBreakerOpenError()
        
        try:
            result = await func()
            if self.state == "half-open":
                self.state = "closed"
                self.failure_count = 0
            return result
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.failure_threshold:
                self.state = "open"
            raise

# Graceful degradation
class GracefulDegradation:
    def __init__(self):
        self.features = {
            "ai_generation": True,
            "real_time_sync": True,
            "3d_rendering": True,
            "collaboration": True
        }
    
    def disable_feature(self, feature: str):
        self.features[feature] = False
        logger.warning(f"Feature {feature} disabled due to errors")
    
    def is_enabled(self, feature: str) -> bool:
        return self.features.get(feature, False)
```

### Error Logging and Monitoring

```python
# Structured error logging
import structlog

logger = structlog.get_logger()

def log_error(error: Exception, context: Dict[str, Any]):
    logger.error(
        "operation_failed",
        error_type=type(error).__name__,
        error_message=str(error),
        stack_trace=traceback.format_exc(),
        user_id=context.get("user_id"),
        project_id=context.get("project_id"),
        operation=context.get("operation"),
        timestamp=datetime.utcnow().isoformat()
    )

# Error metrics
class ErrorMetrics:
    def __init__(self, metrics_client):
        self.client = metrics_client
    
    def record_error(self, error_type: str, operation: str):
        self.client.increment(
            "errors.count",
            tags=[f"type:{error_type}", f"operation:{operation}"]
        )
    
    def record_recovery(self, error_type: str, recovery_method: str):
        self.client.increment(
            "errors.recovered",
            tags=[f"type:{error_type}", f"method:{recovery_method}"]
        )
```

## Testing Strategy

### Dual Testing Approach

AURA AI requires both unit testing and property-based testing to ensure comprehensive coverage. Unit tests validate specific examples and edge cases, while property tests verify universal correctness guarantees across all inputs. Together, they provide confidence in system correctness.

### Unit Testing

Unit tests focus on:
- Specific examples demonstrating correct behavior
- Edge cases (empty inputs, maximum sizes, boundary conditions)
- Error conditions and exception handling
- Integration points between components
- Mock external dependencies (AI APIs, databases)

**Testing Framework**: 
- Backend: pytest (Python)
- Frontend: Jest + React Testing Library (TypeScript)

**Example Unit Tests**:
```python
# Backend unit test example
def test_node_creation_with_valid_properties():
    node = Node(
        id="test-123",
        project_id="proj-456",
        type=NodeType.API,
        name="GetUsers",
        properties=NodeProperties(
            http_method="GET",
            path="/api/users"
        ),
        position=Vector3(x=0, y=0, z=0),
        status="valid"
    )
    assert node.type == NodeType.API
    assert node.properties.http_method == "GET"

def test_code_generator_handles_empty_node_name():
    generator = CodeGenerator()
    with pytest.raises(ValidationError):
        generator.generate_code(
            NodeSpec(name="", type=NodeType.FUNCTION),
            language="python"
        )
```

```typescript
// Frontend unit test example
describe('MindMapScene', () => {
  it('should create a node at specified position', () => {
    const scene = new MindMapScene();
    const node = scene.createNode({
      id: 'test-1',
      type: NodeType.API,
      position: new Vector3(10, 20, 30)
    });
    
    expect(node.position.x).toBe(10);
    expect(node.position.y).toBe(20);
    expect(node.position.z).toBe(30);
  });
  
  it('should handle node deletion with connected edges', () => {
    const scene = new MindMapScene();
    const node1 = scene.createNode({ id: 'n1', type: NodeType.API });
    const node2 = scene.createNode({ id: 'n2', type: NodeType.SERVICE });
    scene.createEdge({ source: 'n1', target: 'n2' });
    
    scene.deleteNode('n1');
    
    expect(scene.getNode('n1')).toBeUndefined();
    expect(scene.getEdges()).toHaveLength(0);
  });
});
```

### Property-Based Testing

Property tests verify universal correctness properties across randomly generated inputs. Each property test runs a minimum of 100 iterations with different random inputs.

**Testing Framework**:
- Backend: Hypothesis (Python)
- Frontend: fast-check (TypeScript)

**Configuration**:
- Minimum 100 iterations per property test
- Each test tagged with: `Feature: aura-ai, Property {N}: {property_text}`
- Shrinking enabled to find minimal failing examples

**Example Property Tests**:

```python
# Backend property test example
from hypothesis import given, strategies as st

@given(
    node_properties=st.builds(NodeProperties),
    language=st.sampled_from(['python', 'typescript', 'javascript'])
)
def test_property_2_node_to_code_generation_consistency(node_properties, language):
    """
    Feature: aura-ai, Property 2: Node-to-Code Generation Consistency
    For any node created in the Mind_Map, the Code_Generator should produce 
    syntactically valid code that, when parsed back, produces equivalent properties.
    """
    generator = CodeGenerator()
    parser = CodeParser()
    
    # Create node
    node = Node(
        id=str(uuid4()),
        project_id=str(uuid4()),
        type=NodeType.FUNCTION,
        name="test_function",
        properties=node_properties,
        position=Vector3(x=0, y=0, z=0),
        status="valid"
    )
    
    # Generate code
    code = generator.generate_code(node, language)
    
    # Verify syntax validity
    assert parser.is_valid_syntax(code, language)
    
    # Parse back to node
    parsed_nodes = parser.parse_to_nodes(code, language)
    
    # Verify equivalence
    assert len(parsed_nodes) > 0
    parsed_node = parsed_nodes[0]
    assert parsed_node.name == node.name
    assert parsed_node.type == node.type

@given(
    nodes=st.lists(
        st.builds(Node, 
                  position=st.builds(Vector3,
                                   x=st.floats(min_value=-100, max_value=100),
                                   y=st.floats(min_value=-100, max_value=100),
                                   z=st.floats(min_value=-100, max_value=100))),
        min_size=2,
        max_size=50
    )
)
def test_property_8_node_layout_non_overlap(nodes):
    """
    Feature: aura-ai, Property 8: Node Layout Non-Overlap
    For any set of nodes, the layout algorithm should position them 
    such that no two node bounding volumes intersect.
    """
    layout_engine = LayoutEngine()
    
    # Apply layout
    positioned_nodes = layout_engine.apply_force_directed_layout(nodes)
    
    # Check for overlaps
    for i, node1 in enumerate(positioned_nodes):
        for node2 in positioned_nodes[i+1:]:
            distance = calculate_distance(node1.position, node2.position)
            min_distance = node1.radius + node2.radius
            assert distance >= min_distance, f"Nodes {node1.id} and {node2.id} overlap"

@given(
    api_key=st.text(min_size=20, max_size=100)
)
def test_property_19_api_key_encryption(api_key):
    """
    Feature: aura-ai, Property 19: API Key Encryption
    For any API key stored, the stored value should be encrypted using AES-256 
    and the plaintext should never appear in logs or displays.
    """
    manager = APIKeyManager()
    
    # Store key
    key_id = manager.store_key(api_key, provider="openai")
    
    # Retrieve stored key
    stored_key = manager.get_stored_key(key_id)
    
    # Verify encryption
    assert stored_key != api_key, "Key should be encrypted"
    assert manager.is_encrypted_with_aes256(stored_key)
    
    # Verify no plaintext in logs
    logs = get_recent_logs()
    assert api_key not in logs, "Plaintext key found in logs"
    
    # Verify masked display
    display_value = manager.get_display_value(key_id)
    assert len(display_value) <= 4 or display_value.count('*') > 0
```

```typescript
// Frontend property test example
import fc from 'fast-check';

describe('Property Tests', () => {
  it('Property 9: Real-Time Position Updates', () => {
    fc.assert(
      fc.property(
        fc.record({
          nodeId: fc.uuid(),
          initialPosition: fc.record({
            x: fc.float({ min: -100, max: 100 }),
            y: fc.float({ min: -100, max: 100 }),
            z: fc.float({ min: -100, max: 100 })
          }),
          dragDelta: fc.record({
            x: fc.float({ min: -50, max: 50 }),
            y: fc.float({ min: -50, max: 50 }),
            z: fc.float({ min: -50, max: 50 })
          })
        }),
        ({ nodeId, initialPosition, dragDelta }) => {
          /**
           * Feature: aura-ai, Property 9: Real-Time Position Updates
           * For any node drag operation, the position should update in real-time
           * and connected edges should maintain valid connections.
           */
          const scene = new MindMapScene();
          const node = scene.createNode({
            id: nodeId,
            type: NodeType.API,
            position: new Vector3(initialPosition.x, initialPosition.y, initialPosition.z)
          });
          
          const connectedNode = scene.createNode({
            id: 'connected',
            type: NodeType.SERVICE,
            position: new Vector3(0, 0, 0)
          });
          
          const edge = scene.createEdge({
            source: nodeId,
            target: 'connected',
            type: EdgeType.DEPENDENCY
          });
          
          // Drag node
          scene.dragNode(nodeId, dragDelta);
          
          // Verify position updated
          const updatedNode = scene.getNode(nodeId);
          expect(updatedNode.position.x).toBeCloseTo(initialPosition.x + dragDelta.x);
          expect(updatedNode.position.y).toBeCloseTo(initialPosition.y + dragDelta.y);
          expect(updatedNode.position.z).toBeCloseTo(initialPosition.z + dragDelta.z);
          
          // Verify edge still connected
          const updatedEdge = scene.getEdge(edge.id);
          expect(updatedEdge.sourceNodeId).toBe(nodeId);
          expect(updatedEdge.targetNodeId).toBe('connected');
        }
      ),
      { numRuns: 100 }
    );
  });
  
  it('Property 1: Bidirectional Sync Round-Trip Consistency', () => {
    fc.assert(
      fc.property(
        fc.record({
          nodeName: fc.string({ minLength: 1, maxLength: 50 }),
          nodeType: fc.constantFrom(NodeType.API, NodeType.FUNCTION, NodeType.SERVICE),
          properties: fc.object()
        }),
        async ({ nodeName, nodeType, properties }) => {
          /**
           * Feature: aura-ai, Property 1: Bidirectional Sync Round-Trip Consistency
           * For any valid node, modifying then reverting should restore original state.
           */
          const syncEngine = new SyncEngine();
          
          // Create initial node
          const originalNode = {
            id: uuid(),
            name: nodeName,
            type: nodeType,
            properties: properties
          };
          
          // Generate code from node
          const code = await syncEngine.nodeToCode(originalNode);
          
          // Parse code back to node
          const parsedNode = await syncEngine.codeToNode(code);
          
          // Verify equivalence
          expect(parsedNode.name).toBe(originalNode.name);
          expect(parsedNode.type).toBe(originalNode.type);
          
          // Modify and revert
          const modifiedNode = { ...parsedNode, name: 'modified' };
          const modifiedCode = await syncEngine.nodeToCode(modifiedNode);
          const revertedCode = code;
          const revertedNode = await syncEngine.codeToNode(revertedCode);
          
          // Verify original state restored
          expect(revertedNode.name).toBe(originalNode.name);
        }
      ),
      { numRuns: 100 }
    );
  });
});
```

### Integration Testing

Integration tests verify interactions between components:
- Frontend ↔ Backend API communication
- Backend ↔ Database operations
- Backend ↔ AI services
- WebSocket real-time updates
- End-to-end user workflows

**Example Integration Test**:
```python
@pytest.mark.integration
async def test_full_node_creation_workflow():
    """Test complete workflow from user input to code generation"""
    async with TestClient(app) as client:
        # Create project
        response = await client.post("/api/projects", json={
            "name": "Test Project",
            "language": "python"
        })
        project_id = response.json()["id"]
        
        # Submit user input
        response = await client.post(f"/api/projects/{project_id}/generate", json={
            "description": "Create a REST API endpoint to get all users"
        })
        
        assert response.status_code == 200
        result = response.json()
        
        # Verify node created
        assert "node_id" in result
        node_id = result["node_id"]
        
        # Verify code generated
        response = await client.get(f"/api/projects/{project_id}/nodes/{node_id}/code")
        code = response.json()["code"]
        
        assert "def get_users" in code or "async def get_users" in code
        assert "@app.get" in code or "@router.get" in code
```

### Performance Testing

Performance tests verify system meets timing requirements:
- Load testing with multiple concurrent users
- Stress testing with maximum node counts
- Latency measurements for all operations
- AMD hardware optimization validation

**Tools**: Locust (load testing), pytest-benchmark (microbenchmarks)

### Test Coverage Goals

- Unit test coverage: Minimum 80% code coverage
- Property test coverage: All 56 correctness properties implemented
- Integration test coverage: All critical user workflows
- Performance test coverage: All timing requirements validated

### Continuous Testing

- Run unit tests on every commit
- Run property tests on every pull request
- Run integration tests before deployment
- Run performance tests weekly and before major releases


## Infrastructure and Deployment

### Development Environment

**Local Development Setup**:
```yaml
# docker-compose.yml
version: '3.8'

services:
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    volumes:
      - ./frontend:/app
      - /app/node_modules
    environment:
      - REACT_APP_API_URL=http://localhost:8000
      - REACT_APP_WS_URL=ws://localhost:8000/ws
    depends_on:
      - backend

  backend:
    build: ./backend
    ports:
      - "8000:8000"
    volumes:
      - ./backend:/app
    environment:
      - DATABASE_URL=postgresql://aura:aura@postgres:5432/aura_db
      - REDIS_URL=redis://redis:6379
      - AMD_NPU_ENABLED=true
    depends_on:
      - postgres
      - redis
    devices:
      - /dev/dri:/dev/dri  # AMD GPU access

  postgres:
    image: postgres:15
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_USER=aura
      - POSTGRES_PASSWORD=aura
      - POSTGRES_DB=aura_db
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      - RABBITMQ_DEFAULT_USER=aura
      - RABBITMQ_DEFAULT_PASS=aura

volumes:
  postgres_data:
  redis_data:
```

### Production Architecture

```mermaid
graph TB
    subgraph "Load Balancer Layer"
        LB[Nginx Load Balancer]
    end
    
    subgraph "Application Layer"
        FE1[Frontend Instance 1]
        FE2[Frontend Instance 2]
        BE1[Backend Instance 1]
        BE2[Backend Instance 2]
        BE3[Backend Instance 3]
    end
    
    subgraph "Cache Layer"
        RC1[Redis Primary]
        RC2[Redis Replica]
    end
    
    subgraph "Database Layer"
        PG1[PostgreSQL Primary]
        PG2[PostgreSQL Replica 1]
        PG3[PostgreSQL Replica 2]
    end
    
    subgraph "Message Queue"
        MQ[RabbitMQ Cluster]
    end
    
    subgraph "Storage Layer"
        S3[Object Storage - S3/MinIO]
    end
    
    subgraph "Monitoring"
        PROM[Prometheus]
        GRAF[Grafana]
        LOG[Loki]
    end
    
    LB --> FE1
    LB --> FE2
    LB --> BE1
    LB --> BE2
    LB --> BE3
    
    BE1 --> RC1
    BE2 --> RC1
    BE3 --> RC1
    RC1 --> RC2
    
    BE1 --> PG1
    BE2 --> PG1
    BE3 --> PG1
    PG1 --> PG2
    PG1 --> PG3
    
    BE1 --> MQ
    BE2 --> MQ
    BE3 --> MQ
    
    BE1 --> S3
    BE2 --> S3
    BE3 --> S3
    
    BE1 --> PROM
    BE2 --> PROM
    BE3 --> PROM
    PROM --> GRAF
    BE1 --> LOG
    BE2 --> LOG
    BE3 --> LOG
```

### AMD Hardware Optimization Configuration

**AMD Ryzen AI NPU Setup**:
```python
# backend/amd_optimizer.py
import os
from typing import Optional

class AMDNPUOptimizer:
    def __init__(self):
        self.npu_available = self._detect_amd_npu()
        self.runtime = None
        
        if self.npu_available:
            self._initialize_runtime()
    
    def _detect_amd_npu(self) -> bool:
        """Detect AMD Ryzen AI NPU"""
        try:
            # Check for AMD IPU device
            if os.path.exists('/dev/accel/accel0'):
                return True
            
            # Check for AMD Ryzen AI software
            import importlib.util
            spec = importlib.util.find_spec('ryzenai')
            return spec is not None
        except Exception:
            return False
    
    def _initialize_runtime(self):
        """Initialize AMD Ryzen AI runtime"""
        try:
            from ryzenai import Runtime
            self.runtime = Runtime()
            self.runtime.set_optimization_level(3)  # Maximum optimization
            logger.info("AMD Ryzen AI NPU initialized successfully")
        except Exception as e:
            logger.warning(f"Failed to initialize AMD NPU: {e}")
            self.npu_available = False
    
    async def run_inference(self, model_name: str, input_data: dict) -> dict:
        """Run AI inference with NPU acceleration"""
        if not self.npu_available or not self.runtime:
            # Fallback to CPU
            return await self._cpu_inference(model_name, input_data)
        
        try:
            # Use NPU for inference
            result = await self.runtime.infer_async(
                model=model_name,
                inputs=input_data,
                use_npu=True
            )
            return result
        except Exception as e:
            logger.error(f"NPU inference failed: {e}, falling back to CPU")
            return await self._cpu_inference(model_name, input_data)
```

**AMD GPU Optimization for Three.js**:
```typescript
// frontend/src/amd-gpu-optimizer.ts
export class AMDGPUOptimizer {
  private renderer: THREE.WebGLRenderer;
  private isAMDGPU: boolean;
  
  constructor(renderer: THREE.WebGLRenderer) {
    this.renderer = renderer;
    this.isAMDGPU = this.detectAMDGPU();
    
    if (this.isAMDGPU) {
      this.applyAMDOptimizations();
    }
  }
  
  private detectAMDGPU(): boolean {
    const gl = this.renderer.getContext();
    const debugInfo = gl.getExtension('WEBGL_debug_renderer_info');
    
    if (debugInfo) {
      const renderer = gl.getParameter(debugInfo.UNMASKED_RENDERER_WEBGL);
      return renderer.toLowerCase().includes('amd') || 
             renderer.toLowerCase().includes('radeon');
    }
    
    return false;
  }
  
  private applyAMDOptimizations(): void {
    // Enable AMD-specific optimizations
    this.renderer.setPixelRatio(window.devicePixelRatio);
    this.renderer.powerPreference = 'high-performance';
    
    // Use instanced rendering for repeated geometries
    this.renderer.capabilities.isWebGL2 = true;
    
    // Enable AMD FidelityFX features if available
    const gl = this.renderer.getContext();
    const ext = gl.getExtension('AMD_performance_monitor');
    
    if (ext) {
      console.log('AMD GPU optimizations enabled');
    }
    
    // Optimize for AMD RDNA architecture
    this.renderer.sortObjects = false; // Reduce CPU overhead
    this.renderer.info.autoReset = false; // Manual reset for better control
  }
  
  public optimizeScene(scene: THREE.Scene): void {
    if (!this.isAMDGPU) return;
    
    // Use instanced mesh for similar nodes
    scene.traverse((object) => {
      if (object instanceof THREE.Mesh) {
        // Enable frustum culling
        object.frustumCulled = true;
        
        // Use lower precision for distant objects
        if (object.position.length() > 100) {
          object.matrixAutoUpdate = false;
        }
      }
    });
  }
}
```

### Campus Deployment Model

**Multi-Campus Architecture**:
```yaml
# Kubernetes deployment for campus-wide installation
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aura-ai-backend
  namespace: aura-ai
spec:
  replicas: 3
  selector:
    matchLabels:
      app: aura-backend
  template:
    metadata:
      labels:
        app: aura-backend
    spec:
      containers:
      - name: backend
        image: aura-ai/backend:latest
        ports:
        - containerPort: 8000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: aura-secrets
              key: database-url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: aura-secrets
              key: redis-url
        - name: AMD_NPU_ENABLED
          value: "true"
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: aura-backend-service
  namespace: aura-ai
spec:
  selector:
    app: aura-backend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8000
  type: LoadBalancer
```

**SSO Integration**:
```python
# backend/auth/sso.py
from fastapi import Depends, HTTPException
from fastapi.security import OAuth2AuthorizationCodeBearer
import jwt

class SSOAuthenticator:
    def __init__(self, saml_config: dict):
        self.saml_config = saml_config
        self.oauth2_scheme = OAuth2AuthorizationCodeBearer(
            authorizationUrl=saml_config['authorization_url'],
            tokenUrl=saml_config['token_url']
        )
    
    async def authenticate_user(self, token: str = Depends(oauth2_scheme)):
        """Authenticate user via university SSO"""
        try:
            # Verify JWT token from SSO provider
            payload = jwt.decode(
                token,
                self.saml_config['public_key'],
                algorithms=['RS256']
            )
            
            user_id = payload.get('sub')
            email = payload.get('email')
            
            # Get or create user in local database
            user = await self.get_or_create_user(user_id, email)
            
            return user
        except jwt.InvalidTokenError:
            raise HTTPException(status_code=401, detail="Invalid authentication token")
```

### Monitoring and Observability

**Prometheus Metrics**:
```python
# backend/monitoring/metrics.py
from prometheus_client import Counter, Histogram, Gauge

# Request metrics
request_count = Counter(
    'aura_requests_total',
    'Total number of requests',
    ['method', 'endpoint', 'status']
)

request_duration = Histogram(
    'aura_request_duration_seconds',
    'Request duration in seconds',
    ['method', 'endpoint']
)

# AI generation metrics
ai_generation_duration = Histogram(
    'aura_ai_generation_duration_seconds',
    'AI generation duration in seconds',
    ['operation_type']
)

ai_generation_errors = Counter(
    'aura_ai_generation_errors_total',
    'Total AI generation errors',
    ['error_type']
)

# Sync engine metrics
sync_operations = Counter(
    'aura_sync_operations_total',
    'Total sync operations',
    ['operation_type', 'status']
)

sync_conflicts = Counter(
    'aura_sync_conflicts_total',
    'Total sync conflicts',
    ['resolution_method']
)

# Collaboration metrics
active_users = Gauge(
    'aura_active_users',
    'Number of active users',
    ['project_id']
)

websocket_connections = Gauge(
    'aura_websocket_connections',
    'Number of active WebSocket connections'
)

# Performance metrics
rendering_fps = Gauge(
    'aura_rendering_fps',
    'Current rendering FPS',
    ['user_id']
)

node_count = Gauge(
    'aura_node_count',
    'Number of nodes in project',
    ['project_id']
)
```

**Logging Configuration**:
```python
# backend/logging_config.py
import structlog

def configure_logging():
    structlog.configure(
        processors=[
            structlog.stdlib.filter_by_level,
            structlog.stdlib.add_logger_name,
            structlog.stdlib.add_log_level,
            structlog.stdlib.PositionalArgumentsFormatter(),
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.StackInfoRenderer(),
            structlog.processors.format_exc_info,
            structlog.processors.UnicodeDecoder(),
            structlog.processors.JSONRenderer()
        ],
        context_class=dict,
        logger_factory=structlog.stdlib.LoggerFactory(),
        cache_logger_on_first_use=True,
    )
```

## Development Phases

### Phase 1: MVP (Weeks 1-4) - Core Functionality

**Goal**: Create a working prototype demonstrating core value proposition

**Features**:
1. Basic 3D mind map visualization with Three.js
   - Node creation and positioning
   - Simple force-directed layout
   - Basic camera controls (pan, zoom, rotate)

2. Natural language input processing
   - Simple intent extraction
   - Basic entity recognition
   - Support for API and function node types

3. Code generation for Python
   - Function generation
   - Basic API endpoint generation
   - Simple validation

4. Basic synchronization
   - Node-to-code generation
   - Manual sync trigger
   - No real-time updates yet

5. Single-user mode
   - Local storage for projects
   - No authentication
   - No collaboration features

**Deliverables**:
- Working prototype demonstrating idea → 3D visualization → code generation
- Basic UI with 3D canvas and code editor
- Docker Compose setup for local development
- Initial documentation

**Success Criteria**:
- User can describe a simple API and see it visualized in 3D
- Generated code is syntactically valid
- Demo-ready for hackathon presentation

### Phase 2: Expansion (Weeks 5-8) - Enhanced Features

**Goal**: Add real-time collaboration, multi-language support, and advanced features

**Features**:
1. Real-time bidirectional synchronization
   - WebSocket implementation
   - Automatic code-to-node updates
   - Conflict detection and resolution

2. Multi-language support
   - TypeScript code generation
   - JavaScript code generation
   - Language switching

3. Collaboration features
   - Multi-user support
   - Real-time presence indicators
   - Node locking
   - Cursor sharing

4. Advanced node types
   - Data model nodes with relationships
   - UI component nodes
   - Database nodes

5. Error detection and highlighting
   - Syntax validation
   - Dependency checking
   - Visual error indicators

6. Template library
   - 10+ pre-built templates
   - Template instantiation
   - Custom template creation

7. Version history
   - Automatic snapshots
   - Version timeline
   - Rollback functionality

**Deliverables**:
- Multi-user collaborative workspace
- Support for Python, TypeScript, JavaScript
- Template gallery
- Version control system
- Enhanced UI with error highlighting

**Success Criteria**:
- Multiple users can collaborate in real-time
- Code generation works for all three languages
- Users can start from templates
- Version history allows safe experimentation

### Phase 3: Scaling (Weeks 9-12) - Enterprise-Ready

**Goal**: Prepare for campus-wide deployment with security, scalability, and advanced features

**Features**:
1. Authentication and authorization
   - University SSO integration (SAML/OAuth)
   - Role-based access control
   - API key management

2. Multi-tenancy
   - Data isolation per user
   - Resource quotas
   - Usage tracking

3. AMD hardware optimization
   - NPU acceleration for AI inference
   - GPU optimization for 3D rendering
   - Performance benchmarking

4. Advanced AI features
   - Better intent understanding
   - Code improvement suggestions
   - Automatic refactoring

5. Project export and sharing
   - ZIP export with complete project structure
   - Public project sharing
   - Community features

6. Database schema design
   - Visual data model editor
   - Migration script generation
   - Relationship management

7. API testing interface
   - Integrated API testing
   - Request/response visualization
   - OpenAPI documentation generation

8. Monitoring and analytics
   - Performance dashboards
   - Usage analytics
   - Error tracking

**Deliverables**:
- Production-ready deployment on Kubernetes
- SSO integration for campus authentication
- AMD-optimized performance
- Complete monitoring and observability
- Admin dashboard for campus IT
- Comprehensive documentation

**Success Criteria**:
- System handles 100+ concurrent users
- AMD optimization provides 30%+ performance improvement
- 99% uptime during business hours
- Complete audit trail for compliance
- Ready for campus-wide rollout

## Repository Structure

```
aura-ai/
├── README.md
├── LICENSE
├── .gitignore
├── docker-compose.yml
├── docker-compose.prod.yml
│
├── docs/
│   ├── architecture.md
│   ├── api-reference.md
│   ├── deployment-guide.md
│   ├── user-guide.md
│   ├── developer-guide.md
│   └── amd-optimization.md
│
├── frontend/
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   ├── Dockerfile
│   ├── public/
│   │   ├── index.html
│   │   └── assets/
│   └── src/
│       ├── main.tsx
│       ├── App.tsx
│       ├── components/
│       │   ├── MindMap/
│       │   │   ├── MindMapScene.ts
│       │   │   ├── NodeRenderer.ts
│       │   │   ├── EdgeRenderer.ts
│       │   │   ├── InteractionController.ts
│       │   │   └── LayoutEngine.ts
│       │   ├── CodeEditor/
│       │   │   └── MonacoEditor.tsx
│       │   ├── Dashboard/
│       │   │   └── Dashboard.tsx
│       │   ├── PropertyPanel/
│       │   │   └── PropertyPanel.tsx
│       │   └── TemplateGallery/
│       │       └── TemplateGallery.tsx
│       ├── stores/
│       │   ├── projectStore.ts
│       │   ├── uiStore.ts
│       │   ├── authStore.ts
│       │   └── collaborationStore.ts
│       ├── services/
│       │   ├── api.ts
│       │   ├── websocket.ts
│       │   └── amd-gpu-optimizer.ts
│       ├── types/
│       │   └── index.ts
│       └── utils/
│           └── helpers.ts
│
├── backend/
│   ├── pyproject.toml
│   ├── poetry.lock
│   ├── Dockerfile
│   ├── alembic.ini
│   ├── alembic/
│   │   └── versions/
│   └── app/
│       ├── main.py
│       ├── config.py
│       ├── api/
│       │   ├── __init__.py
│       │   ├── projects.py
│       │   ├── nodes.py
│       │   ├── generation.py
│       │   ├── templates.py
│       │   └── websocket.py
│       ├── core/
│       │   ├── __init__.py
│       │   ├── sync_engine.py
│       │   ├── code_generator.py
│       │   ├── code_parser.py
│       │   ├── error_detector.py
│       │   ├── ai_orchestrator.py
│       │   ├── version_manager.py
│       │   └── amd_optimizer.py
│       ├── models/
│       │   ├── __init__.py
│       │   ├── user.py
│       │   ├── project.py
│       │   ├── node.py
│       │   ├── edge.py
│       │   └── version.py
│       ├── schemas/
│       │   ├── __init__.py
│       │   ├── project.py
│       │   ├── node.py
│       │   └── generation.py
│       ├── db/
│       │   ├── __init__.py
│       │   ├── database.py
│       │   └── session.py
│       ├── auth/
│       │   ├── __init__.py
│       │   ├── sso.py
│       │   └── jwt.py
│       ├── monitoring/
│       │   ├── __init__.py
│       │   ├── metrics.py
│       │   └── logging_config.py
│       └── tests/
│           ├── unit/
│           │   ├── test_sync_engine.py
│           │   ├── test_code_generator.py
│           │   └── test_error_detector.py
│           ├── property/
│           │   ├── test_properties_sync.py
│           │   ├── test_properties_generation.py
│           │   └── test_properties_security.py
│           └── integration/
│               ├── test_api.py
│               └── test_websocket.py
│
├── infrastructure/
│   ├── kubernetes/
│   │   ├── namespace.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── ingress.yaml
│   │   └── secrets.yaml
│   ├── terraform/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── monitoring/
│       ├── prometheus.yml
│       ├── grafana-dashboards/
│       └── alerting-rules.yml
│
└── scripts/
    ├── setup-dev.sh
    ├── run-tests.sh
    ├── deploy.sh
    └── backup-db.sh
```

## Security Considerations

### Data Protection
- All API keys encrypted with AES-256
- TLS 1.3 for all network communication
- Database encryption at rest
- Regular security audits

### Access Control
- JWT-based authentication
- Role-based access control (RBAC)
- SSO integration for campus deployment
- API rate limiting per user

### Input Validation
- Sanitize all user inputs
- Validate code before execution
- Prevent injection attacks
- Content Security Policy (CSP) headers

### Audit Logging
- Log all authentication attempts
- Track all data modifications
- Monitor API usage patterns
- Alert on suspicious activity

## Future Enhancements

### Phase 4+ Features
- Mobile app (iOS/Android) for project viewing
- Voice input for hands-free development
- AR/VR support for immersive 3D workspace
- Advanced AI model fine-tuning
- IDE integration (VS Code, JetBrains)
- Git integration for version control
- CI/CD pipeline generation
- Automated testing generation
- Performance profiling and optimization suggestions
- Multi-language translation for international users
- Accessibility improvements (screen reader support, keyboard navigation)
- Offline mode with sync when online
- Plugin system for extensibility
- Marketplace for community templates and plugins

### Business Model Evolution
- Free tier for students (campus-sponsored)
- Pro tier for individual developers
- Team tier for development teams
- Enterprise tier for companies
- White-label licensing for other universities
- SaaS offering for non-campus users
