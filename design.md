# Design Document: ConceptBridge

## Overview

ConceptBridge is an AI-powered learning platform that personalizes education by translating complex concepts into analogies drawn from each user's unique knowledge base. The system architecture follows a serverless, event-driven design on AWS, leveraging Amazon Bedrock for AI capabilities, DynamoDB for user data, and OpenSearch for vector-based concept matching.

The core innovation lies in the analogy generation pipeline: when a user wants to learn a new concept, the system retrieves their knowledge profile, performs vector similarity search to find familiar concepts, identifies knowledge gaps, and uses large language models to craft custom analogies that bridge the gap between known and unknown.

### Key Design Principles

1. **Personalization-First**: Every explanation is tailored to the individual user's background
2. **Serverless Architecture**: Scalable, cost-effective Lambda-based compute
3. **AI-Native**: Deep integration with Amazon Bedrock for LLM and embedding capabilities
4. **Multilingual by Design**: Translation and voice services integrated at the core
5. **Privacy-Focused**: User data encrypted and isolated per user
6. **Observable**: Comprehensive logging and tracing for debugging and optimization

## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Frontend Layer                            │
│  Next.js App (AWS Amplify) + D3.js Concept Maps             │
└────────────────────────┬────────────────────────────────────┘
                         │ HTTPS/REST
┌────────────────────────▼────────────────────────────────────┐
│                  API Gateway Layer                           │
│  Amazon API Gateway (REST API + WebSocket for chat)         │
│  - Authentication (Cognito Authorizer)                       │
│  - Rate Limiting & Throttling                                │
│  - Request Validation                                        │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│                  Lambda Function Layer                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Profile     │  │  Analogy     │  │  Gap         │      │
│  │  Manager     │  │  Engine      │  │  Analyzer    │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Concept Map │  │  Learning    │  │  Progress    │      │
│  │  Generator   │  │  Session     │  │  Tracker     │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│                   AI Services Layer                          │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Amazon Bedrock                                      │   │
│  │  - Claude 3.5 Sonnet (Analogy Generation)           │   │
│  │  - Amazon Titan (Embeddings)                        │   │
│  │  - Bedrock Knowledge Bases (RAG)                    │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Translate   │  │  Polly       │  │  Transcribe  │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│                   Data Layer                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  DynamoDB    │  │  OpenSearch  │  │  S3          │      │
│  │  (Profiles,  │  │  Serverless  │  │  (Knowledge  │      │
│  │   Progress)  │  │  (Vectors)   │  │   Base)      │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│  ┌──────────────┐  ┌──────────────┐                         │
│  │  Cognito     │  │  CloudWatch  │                         │
│  │  (Auth)      │  │  (Logs)      │                         │
│  └──────────────┘  └──────────────┘                         │
└─────────────────────────────────────────────────────────────┘
```

### Data Flow: Learning a New Concept

```
User Request: "Explain Kubernetes"
         │
         ▼
┌─────────────────────┐
│ 1. Profile Manager  │ → Fetch user's Knowledge_Profile from DynamoDB
└─────────────────────┘   (profession: farmer, hobbies: cricket)
         │
         ▼
┌─────────────────────┐
│ 2. Gap Analyzer     │ → Check prerequisites for Kubernetes
└─────────────────────┘   → Identify missing concepts (containers, orchestration)
         │
         ▼
┌─────────────────────┐
│ 3. Vector Search    │ → Generate embedding for "Kubernetes"
└─────────────────────┘   → Query OpenSearch for similar concepts in user's profile
         │                 → Returns: ["farm management", "cricket team coordination"]
         ▼
┌─────────────────────┐
│ 4. Analogy Engine   │ → Prompt Bedrock Claude:
└─────────────────────┘     "Explain Kubernetes using farm management concepts"
         │                 → RAG: Retrieve Kubernetes docs from Knowledge Base
         ▼                 → Generate: "Kubernetes is like managing multiple farms..."
┌─────────────────────┐
│ 5. Concept Map Gen  │ → Build graph: Kubernetes → Farm Management → User's Farm
└─────────────────────┘   → Return D3.js-compatible JSON
         │
         ▼
┌─────────────────────┐
│ 6. Translation      │ → If user language != English, translate via Amazon Translate
└─────────────────────┘
         │
         ▼
┌─────────────────────┐
│ 7. Response         │ → Return to frontend with analogy + concept map + gaps
└─────────────────────┘
```

## Components and Interfaces

### 1. Profile Manager Lambda

**Responsibility**: Manage user knowledge profiles, including creation, updates, and retrieval.

**Interfaces**:
```python
# Input
{
  "user_id": "string",
  "action": "create" | "update" | "get",
  "profile_data": {
    "profession": "string",
    "hobbies": ["string"],
    "interests": ["string"],
    "language": "string",
    "known_concepts": ["string"]
  }
}

# Output
{
  "user_id": "string",
  "profile": {
    "profession": "string",
    "hobbies": ["string"],
    "interests": ["string"],
    "language": "string",
    "known_concepts": ["string"],
    "concept_embeddings": [{"concept": "string", "embedding": [float]}]
  },
  "timestamp": "ISO8601"
}
```

**Key Operations**:
- `create_profile(user_id, profile_data)`: Initialize new user profile in DynamoDB
- `update_profile(user_id, updates)`: Merge updates into existing profile
- `get_profile(user_id)`: Retrieve profile from DynamoDB
- `add_known_concept(user_id, concept)`: Add concept to user's knowledge base and generate embedding

**Dependencies**:
- DynamoDB table: `conceptbridge-users`
- Amazon Titan Embeddings for concept vectorization
- OpenSearch for storing concept embeddings

### 2. Analogy Engine Lambda

**Responsibility**: Generate personalized analogies by mapping target concepts to familiar concepts using LLMs.

**Interfaces**:
```python
# Input
{
  "user_id": "string",
  "target_concept": "string",
  "familiar_concepts": ["string"],  # From vector search
  "context": "string",  # Optional: additional context from RAG
  "language": "string"
}

# Output
{
  "analogy": {
    "target_concept": "string",
    "familiar_concept": "string",
    "explanation": "string",
    "mapping": [
      {"target_attribute": "string", "familiar_attribute": "string"}
    ]
  },
  "confidence_score": float,
  "alternative_analogies": [...]  # Backup options
}
```

**Key Operations**:
- `generate_analogy(target, familiar, context)`: Use Bedrock Claude to create analogy
- `build_prompt(target, familiar, user_profile)`: Construct LLM prompt with user context
- `validate_analogy(analogy)`: Check that analogy has clear mappings
- `retrieve_knowledge_base_context(target)`: Use Bedrock Knowledge Bases RAG for grounding

**Prompt Template**:
```
You are an expert educator creating personalized analogies.

User Profile:
- Profession: {profession}
- Hobbies: {hobbies}
- Interests: {interests}

Task: Explain "{target_concept}" using concepts from "{familiar_concept}".

Requirements:
1. Create a clear analogy that maps key attributes
2. Use terminology familiar to someone with the user's background
3. Highlight similarities and acknowledge differences
4. Keep explanation concise (3-4 sentences)

Context from knowledge base:
{rag_context}

Generate the analogy:
```

**Dependencies**:
- Amazon Bedrock (Claude 3.5 Sonnet)
- Bedrock Knowledge Bases for RAG
- Amazon Translate for multilingual output

### 3. Gap Analyzer Lambda

**Responsibility**: Identify missing prerequisite knowledge required to understand a target concept.

**Interfaces**:
```python
# Input
{
  "user_id": "string",
  "target_concept": "string",
  "user_known_concepts": ["string"]
}

# Output
{
  "target_concept": "string",
  "prerequisites": [
    {
      "concept": "string",
      "importance": "critical" | "helpful" | "optional",
      "known": boolean
    }
  ],
  "knowledge_gaps": ["string"],  # Concepts user doesn't know
  "learning_path": ["string"]  # Ordered list of concepts to learn
}
```

**Key Operations**:
- `identify_prerequisites(concept)`: Query knowledge base for concept dependencies
- `check_user_knowledge(user_id, prerequisites)`: Compare prerequisites against user profile
- `generate_learning_path(gaps)`: Order concepts by dependency graph
- `create_diagnostic_quiz(concept)`: Generate questions to assess understanding

**Algorithm**:
```python
def analyze_gaps(target_concept, user_profile):
    # 1. Retrieve concept dependency graph from knowledge base
    prerequisites = get_prerequisites(target_concept)
    
    # 2. Check which prerequisites user knows
    gaps = []
    for prereq in prerequisites:
        if prereq not in user_profile.known_concepts:
            gaps.append(prereq)
    
    # 3. Build learning path using topological sort
    learning_path = topological_sort(gaps)
    
    return {
        "gaps": gaps,
        "learning_path": learning_path
    }
```

**Dependencies**:
- DynamoDB for user profiles
- S3/Knowledge Base for concept dependency data
- Bedrock for generating diagnostic questions

### 4. Concept Map Generator Lambda

**Responsibility**: Create visual graph data connecting target concepts to familiar concepts.

**Interfaces**:
```python
# Input
{
  "user_id": "string",
  "target_concept": "string",
  "familiar_concepts": ["string"],
  "analogies": [...]  # From Analogy Engine
}

# Output
{
  "nodes": [
    {
      "id": "string",
      "label": "string",
      "type": "target" | "familiar" | "gap",
      "description": "string"
    }
  ],
  "edges": [
    {
      "source": "string",
      "target": "string",
      "relationship": "similar_to" | "prerequisite_of" | "example_of",
      "strength": float  # 0-1
    }
  ],
  "layout": "force" | "hierarchical"
}
```

**Key Operations**:
- `build_concept_graph(target, familiar, analogies)`: Construct node and edge lists
- `calculate_edge_weights(concept_a, concept_b)`: Use embedding similarity for edge strength
- `add_gap_nodes(gaps)`: Include missing prerequisites in graph
- `optimize_layout(graph)`: Suggest layout algorithm based on graph structure

**Dependencies**:
- OpenSearch for concept similarity scores
- DynamoDB for user's known concepts

### 5. Learning Session Lambda

**Responsibility**: Manage conversational learning interactions, maintaining context and adapting responses.

**Interfaces**:
```python
# Input (WebSocket message)
{
  "user_id": "string",
  "session_id": "string",
  "message": "string",
  "message_type": "question" | "clarification" | "feedback"
}

# Output
{
  "session_id": "string",
  "response": "string",
  "response_type": "explanation" | "question" | "encouragement",
  "difficulty_level": int,  # 1-5
  "suggested_actions": ["string"]  # e.g., ["view_concept_map", "take_quiz"]
}
```

**Key Operations**:
- `process_message(user_id, session_id, message)`: Handle user input and generate response
- `maintain_context(session_id)`: Store conversation history in DynamoDB
- `detect_confusion(message, history)`: Analyze user responses for comprehension signals
- `adjust_difficulty(session_id, direction)`: Increase or decrease explanation complexity
- `generate_response(context, user_profile)`: Use Bedrock to create conversational reply

**Context Management**:
```python
# Session stored in DynamoDB
{
  "session_id": "string",
  "user_id": "string",
  "target_concept": "string",
  "messages": [
    {"role": "user" | "assistant", "content": "string", "timestamp": "ISO8601"}
  ],
  "difficulty_level": int,
  "concepts_covered": ["string"],
  "user_comprehension_score": float
}
```

**Dependencies**:
- Amazon Bedrock for conversational AI
- DynamoDB for session storage
- Profile Manager for user context
- Analogy Engine for generating explanations

### 6. Progress Tracker Lambda

**Responsibility**: Record learning activities, calculate metrics, and manage spaced repetition schedules.

**Interfaces**:
```python
# Input
{
  "user_id": "string",
  "action": "record_learning" | "get_progress" | "get_reviews_due",
  "data": {
    "concept": "string",
    "time_spent": int,  # seconds
    "quiz_score": float,  # 0-1
    "completed": boolean
  }
}

# Output
{
  "user_id": "string",
  "progress": {
    "concepts_learned": int,
    "total_time_minutes": int,
    "average_quiz_score": float,
    "current_streak_days": int,
    "reviews_due": [
      {
        "concept": "string",
        "due_date": "ISO8601",
        "review_count": int
      }
    ]
  }
}
```

**Key Operations**:
- `record_concept_completion(user_id, concept, score)`: Log completed concept
- `calculate_next_review(concept, performance)`: Compute spaced repetition interval
- `get_reviews_due(user_id)`: Retrieve concepts scheduled for review
- `update_streak(user_id)`: Calculate consecutive learning days
- `generate_progress_report(user_id)`: Aggregate metrics for dashboard

**Spaced Repetition Algorithm**:
```python
def calculate_next_review(concept, performance_score, review_count):
    """
    SM-2 algorithm variant
    performance_score: 0-1 (quiz score)
    review_count: number of times reviewed
    """
    if performance_score >= 0.8:
        # Good recall: increase interval
        interval_days = [1, 3, 7, 14, 30, 60, 120][min(review_count, 6)]
    elif performance_score >= 0.6:
        # Moderate recall: maintain interval
        interval_days = [1, 2, 4, 7, 14, 30][min(review_count, 5)]
    else:
        # Poor recall: reset to short interval
        interval_days = 1
    
    next_review = datetime.now() + timedelta(days=interval_days)
    return next_review
```

**Dependencies**:
- DynamoDB table: `conceptbridge-progress`
- CloudWatch for metrics

## Data Models

### DynamoDB Tables

#### Table: `conceptbridge-users`

**Primary Key**: `user_id` (String, Partition Key)

**Attributes**:
```json
{
  "user_id": "string (UUID)",
  "email": "string",
  "cognito_sub": "string",
  "profile": {
    "profession": "string",
    "hobbies": ["string"],
    "interests": ["string"],
    "language": "string (ISO 639-1 code)",
    "cultural_context": "string"
  },
  "known_concepts": [
    {
      "concept": "string",
      "confidence": "float (0-1)",
      "learned_date": "ISO8601",
      "embedding_id": "string"
    }
  ],
  "preferences": {
    "voice_enabled": "boolean",
    "difficulty_preference": "int (1-5)",
    "notification_enabled": "boolean"
  },
  "created_at": "ISO8601",
  "updated_at": "ISO8601"
}
```

**Indexes**:
- GSI: `email-index` (for login lookup)

#### Table: `conceptbridge-progress`

**Primary Key**: 
- Partition Key: `user_id` (String)
- Sort Key: `concept_id` (String)

**Attributes**:
```json
{
  "user_id": "string",
  "concept_id": "string",
  "concept_name": "string",
  "status": "learning | completed | reviewing",
  "learning_history": [
    {
      "session_id": "string",
      "timestamp": "ISO8601",
      "time_spent_seconds": "int",
      "quiz_score": "float",
      "difficulty_level": "int"
    }
  ],
  "spaced_repetition": {
    "next_review_date": "ISO8601",
    "review_count": "int",
    "last_performance": "float"
  },
  "total_time_seconds": "int",
  "completion_date": "ISO8601",
  "created_at": "ISO8601",
  "updated_at": "ISO8601"
}
```

**Indexes**:
- GSI: `user_id-next_review_date-index` (for querying reviews due)

#### Table: `conceptbridge-sessions`

**Primary Key**: `session_id` (String, Partition Key)

**Attributes**:
```json
{
  "session_id": "string (UUID)",
  "user_id": "string",
  "target_concept": "string",
  "start_time": "ISO8601",
  "end_time": "ISO8601",
  "messages": [
    {
      "role": "user | assistant",
      "content": "string",
      "timestamp": "ISO8601"
    }
  ],
  "difficulty_level": "int",
  "concepts_covered": ["string"],
  "analogies_used": [
    {
      "target": "string",
      "familiar": "string",
      "user_feedback": "helpful | unclear | confusing"
    }
  ],
  "comprehension_score": "float",
  "ttl": "int (Unix timestamp for auto-deletion after 30 days)"
}
```

**Indexes**:
- GSI: `user_id-start_time-index` (for user session history)

### OpenSearch Serverless Collection

**Collection**: `conceptbridge-vectors`

**Index**: `concept-embeddings`

**Document Schema**:
```json
{
  "concept_id": "string",
  "concept_name": "string",
  "description": "string",
  "embedding": [float],  // 1536 dimensions (Titan Embeddings)
  "category": "string",
  "source": "user_profile | knowledge_base",
  "user_id": "string (if from user profile)",
  "metadata": {
    "domain": "string",
    "difficulty": "int",
    "prerequisites": ["string"]
  }
}
```

**Vector Search Query**:
```python
{
  "size": 5,
  "query": {
    "knn": {
      "embedding": {
        "vector": [float],  # Query embedding
        "k": 5
      }
    }
  },
  "filter": {
    "term": {
      "user_id": "string"  # Only search user's concepts
    }
  }
}
```

### S3 Bucket Structure

**Bucket**: `conceptbridge-knowledge-base`

```
/documents/
  /domains/
    /technology/
      kubernetes.md
      apis.md
      ...
    /science/
      quantum-computing.md
      ...
  /concept-graphs/
    concept-dependencies.json
    
/user-exports/
  /{user_id}/
    concept-maps/
      {concept_id}.png
```

### Bedrock Knowledge Base

**Data Source**: S3 bucket `conceptbridge-knowledge-base/documents/`

**Chunking Strategy**:
- Chunk size: 300 tokens
- Overlap: 20%
- Metadata: domain, difficulty, prerequisites

**Embedding Model**: Amazon Titan Embeddings G1 - Text

**Retrieval Configuration**:
- Number of results: 5
- Search type: Hybrid (semantic + keyword)


## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property Reflection

After analyzing all acceptance criteria, I identified several areas of redundancy:

1. **Profile persistence properties** (1.2, 1.3, 6.6): These can be combined into a single round-trip property for profile data
2. **Difficulty adjustment properties** (7.1, 7.2, 7.5): These describe the same adaptive mechanism in different directions
3. **Spaced repetition interval properties** (8.2, 8.4, 8.5): These all test interval calculation based on performance
4. **Authorization properties** (10.5): This is the core security property that subsumes other access control checks
5. **Vector search properties** (12.3, 12.4): Ranking and retrieval can be tested together

The following properties represent the unique, non-redundant correctness guarantees for ConceptBridge:

### Core Properties

**Property 1: Profile Data Round-Trip**

*For any* user profile data (profession, hobbies, interests, language), storing the profile and then retrieving it should return equivalent data.

**Validates: Requirements 1.2, 1.3, 6.6**

**Property 2: Analogy Generation Completeness**

*For any* user with a non-empty knowledge profile and any target concept, requesting an explanation should return at least one analogy that references concepts from the user's profile.

**Validates: Requirements 2.1, 1.5**

**Property 3: Analogy Structure Validity**

*For any* generated analogy, the response should contain a mapping structure with at least one target attribute and corresponding familiar attribute.

**Validates: Requirements 2.3**

**Property 4: Concept Selection by Similarity**

*For any* target concept and user profile with multiple familiar concepts, the selected familiar concept for analogy generation should have the highest embedding similarity score among available options.

**Validates: Requirements 2.4**

**Property 5: Alternative Analogy Diversity**

*For any* analogy marked as unclear, requesting an alternative should return an analogy using a different familiar concept than the original.

**Validates: Requirements 2.5**

**Property 6: Analogy Caching**

*For any* successfully generated analogy, generating the same analogy again for a user with an equivalent profile should retrieve the cached version (idempotence).

**Validates: Requirements 2.6**

**Property 7: Gap Analysis Completeness**

*For any* target concept and user profile, gap analysis should return all prerequisite concepts that are not in the user's known concepts list.

**Validates: Requirements 3.1, 3.2**

**Property 8: Quiz Updates Profile**

*For any* diagnostic quiz completion with passing scores, the user's knowledge profile should be updated to include the assessed concepts.

**Validates: Requirements 3.5**

**Property 9: Concept Map Structure**

*For any* target concept, the generated concept map should contain at least one node for the target concept, at least one node for a familiar concept, and at least one edge connecting them.

**Validates: Requirements 4.1, 4.2**

**Property 10: Concept Map Node Types**

*For any* concept map, all nodes should have a type field with value "target", "familiar", or "gap".

**Validates: Requirements 4.4**

**Property 11: Concept Map Reflects Profile Updates**

*For any* user profile, if a new concept is added to the profile and a concept map is regenerated for a previously learned target concept, the new concept should appear in the updated map if it's relevant (similarity above threshold).

**Validates: Requirements 4.5**

**Property 12: Session Context Preservation**

*For any* learning session with multiple messages, sending a follow-up message should receive a response that references information from earlier messages in the session.

**Validates: Requirements 5.4**

**Property 13: Multilingual Support Coverage**

*For any* of the 8 supported languages (Hindi, Tamil, Telugu, Bengali, Kannada, Malayalam, Marathi, Gujarati), setting that language as preference should result in responses in that language.

**Validates: Requirements 6.1, 6.2**

**Property 14: Language Switching Consistency**

*For any* active learning session, switching the user's language preference should result in subsequent responses being in the new language while maintaining the same conceptual content.

**Validates: Requirements 6.5**

**Property 15: Adaptive Difficulty Increases with Success**

*For any* learning session, providing correct answers should result in the difficulty level increasing or staying the same, never decreasing.

**Validates: Requirements 7.1**

**Property 16: Adaptive Difficulty Decreases with Struggle**

*For any* learning session, providing incorrect answers or requesting clarification should result in the difficulty level decreasing or staying the same, never increasing.

**Validates: Requirements 7.2**

**Property 17: Difficulty Based on Prior Knowledge**

*For any* two users learning the same target concept, the user with more related concepts in their profile should receive a higher initial difficulty level.

**Validates: Requirements 7.4**

**Property 18: Spaced Repetition Scheduling**

*For any* completed concept, the system should create a review schedule entry with a next review date in the future.

**Validates: Requirements 8.1**

**Property 19: Performance-Based Interval Adjustment**

*For any* concept review, higher performance scores (>0.8) should result in longer next review intervals than lower performance scores (<0.6).

**Validates: Requirements 8.2, 8.4, 8.5**

**Property 20: Review Retrieval Accuracy**

*For any* user, querying for reviews due on a specific date should return only concepts with next review dates on or before that date.

**Validates: Requirements 8.3**

**Property 21: Voice Mode Audio Inclusion**

*For any* response when voice mode is enabled, the response should include audio data or an audio URL.

**Validates: Requirements 9.3**

**Property 22: Voice Language Support**

*For any* supported language, text-to-speech and speech-to-text functionality should be available for that language.

**Validates: Requirements 9.5**

**Property 23: Low Confidence Transcription Handling**

*For any* speech transcription with confidence score below 0.7, the system should request confirmation or repetition from the user.

**Validates: Requirements 9.6**

**Property 24: User Data Isolation**

*For any* two distinct users, user A should not be able to retrieve user B's knowledge profile or learning progress data through any API endpoint.

**Validates: Requirements 10.5**

**Property 25: Progress Tracking Completeness**

*For any* user, the progress data should include all required metrics: concepts learned count, total time spent, quiz scores, and retention rates.

**Validates: Requirements 11.2**

**Property 26: Learning Streak Calculation**

*For any* user with learning activity on N consecutive days, the calculated streak should equal N.

**Validates: Requirements 11.5**

**Property 27: Concept Embedding Generation**

*For any* new concept added to the system, a vector embedding should be generated and stored.

**Validates: Requirements 12.1, 12.5**

**Property 28: Vector Search Result Limit**

*For any* concept similarity search, the system should return at most 5 results.

**Validates: Requirements 12.3**

**Property 29: Vector Search Ranking**

*For any* vector search results, concepts should be ordered by descending similarity score (highest similarity first).

**Validates: Requirements 12.4**

**Property 30: RAG Context Inclusion**

*For any* explanation that uses RAG, the response should include context retrieved from the knowledge base.

**Validates: Requirements 13.3**

**Property 31: Document Indexing**

*For any* new document added to the knowledge base, embeddings should be generated and the document should become searchable.

**Validates: Requirements 13.5**

**Property 32: Relevance Filtering**

*For any* knowledge base retrieval, only content with relevance score above 0.5 should be included in responses.

**Validates: Requirements 13.6**

**Property 33: Authentication Required**

*For any* protected API endpoint, requests without valid authentication tokens should be rejected with 401 status.

**Validates: Requirements 15.3**

**Property 34: Invalid Request Error Handling**

*For any* API request with invalid parameters, the system should return a 4xx status code and an error message describing the validation failure.

**Validates: Requirements 15.4**

**Property 35: Rate Limiting Enforcement**

*For any* user exceeding the rate limit threshold, subsequent requests should be rejected with 429 status until the rate limit window resets.

**Validates: Requirements 15.5**

## Error Handling

### Error Categories

1. **User Input Errors** (4xx)
   - Invalid profile data
   - Malformed requests
   - Missing required fields
   - Authentication failures

2. **System Errors** (5xx)
   - Bedrock API failures
   - DynamoDB throttling
   - OpenSearch unavailability
   - Lambda timeouts

3. **AI Service Errors**
   - Model invocation failures
   - Content filtering violations
   - Embedding generation failures
   - Translation service errors

### Error Handling Strategies

#### Graceful Degradation

When AI services are unavailable:
```python
def generate_analogy_with_fallback(target, familiar, user_profile):
    try:
        # Primary: Use Bedrock Claude
        return bedrock_generate_analogy(target, familiar)
    except BedrockException as e:
        log_error(e)
        try:
            # Fallback: Use cached analogies
            return get_cached_analogy(target, familiar)
        except CacheException:
            # Last resort: Template-based analogy
            return template_analogy(target, familiar, user_profile)
```

#### Retry Logic with Exponential Backoff

For transient failures:
```python
@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10),
    retry=retry_if_exception_type(ThrottlingException)
)
def query_dynamodb(table, key):
    return table.get_item(Key=key)
```

#### Circuit Breaker Pattern

For external service calls:
```python
class BedrockCircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.last_failure_time = None
        self.state = "CLOSED"  # CLOSED, OPEN, HALF_OPEN
    
    def call(self, func, *args, **kwargs):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.timeout:
                self.state = "HALF_OPEN"
            else:
                raise CircuitBreakerOpenException()
        
        try:
            result = func(*args, **kwargs)
            if self.state == "HALF_OPEN":
                self.state = "CLOSED"
                self.failure_count = 0
            return result
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.failure_threshold:
                self.state = "OPEN"
            raise
```

### Error Response Format

All API errors follow this structure:
```json
{
  "error": {
    "code": "string",
    "message": "string",
    "details": {},
    "request_id": "string",
    "timestamp": "ISO8601"
  }
}
```

### Error Codes

| Code | HTTP Status | Description | User Action |
|------|-------------|-------------|-------------|
| `INVALID_PROFILE` | 400 | Profile data validation failed | Check required fields |
| `CONCEPT_NOT_FOUND` | 404 | Target concept not in knowledge base | Try different concept |
| `INSUFFICIENT_KNOWLEDGE` | 400 | User profile lacks familiar concepts | Complete profile setup |
| `AUTHENTICATION_FAILED` | 401 | Invalid or expired token | Re-authenticate |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests | Wait and retry |
| `AI_SERVICE_UNAVAILABLE` | 503 | Bedrock temporarily unavailable | Retry later |
| `TRANSLATION_FAILED` | 500 | Translation service error | Use default language |
| `EMBEDDING_GENERATION_FAILED` | 500 | Vector embedding creation failed | Retry or skip |

## Testing Strategy

### Dual Testing Approach

ConceptBridge requires both unit testing and property-based testing for comprehensive coverage:

- **Unit tests**: Verify specific examples, edge cases, and error conditions
- **Property tests**: Verify universal properties across all inputs

Both approaches are complementary and necessary. Unit tests catch concrete bugs in specific scenarios, while property tests verify general correctness across a wide input space.

### Property-Based Testing

**Framework**: Use `hypothesis` (Python) for property-based testing

**Configuration**:
- Minimum 100 iterations per property test (due to randomization)
- Each property test must reference its design document property
- Tag format: `# Feature: conceptbridge, Property {number}: {property_text}`

**Example Property Test**:
```python
from hypothesis import given, strategies as st
import pytest

# Feature: conceptbridge, Property 1: Profile Data Round-Trip
@given(
    profession=st.text(min_size=1, max_size=100),
    hobbies=st.lists(st.text(min_size=1, max_size=50), min_size=1, max_size=10),
    interests=st.lists(st.text(min_size=1, max_size=50), min_size=1, max_size=10),
    language=st.sampled_from(['en', 'hi', 'ta', 'te', 'bn', 'kn', 'ml', 'mr', 'gu'])
)
@pytest.mark.property_test
def test_profile_round_trip(profession, hobbies, interests, language):
    """Property 1: Profile data round-trip"""
    user_id = create_test_user()
    
    profile_data = {
        "profession": profession,
        "hobbies": hobbies,
        "interests": interests,
        "language": language
    }
    
    # Store profile
    store_profile(user_id, profile_data)
    
    # Retrieve profile
    retrieved = get_profile(user_id)
    
    # Assert equivalence
    assert retrieved["profession"] == profession
    assert set(retrieved["hobbies"]) == set(hobbies)
    assert set(retrieved["interests"]) == set(interests)
    assert retrieved["language"] == language
```

### Unit Testing

**Framework**: `pytest` (Python), `jest` (TypeScript/Frontend)

**Focus Areas**:
- Specific examples demonstrating correct behavior
- Edge cases (empty inputs, boundary values)
- Error conditions and exception handling
- Integration points between components

**Example Unit Tests**:
```python
def test_analogy_generation_with_farmer_profile():
    """Example: Farmer learning about APIs"""
    user_id = create_test_user()
    profile = {
        "profession": "farmer",
        "hobbies": ["cricket"],
        "interests": ["agriculture"],
        "language": "en"
    }
    store_profile(user_id, profile)
    
    response = generate_analogy(user_id, "API")
    
    assert "analogy" in response
    assert "supplier" in response["analogy"]["explanation"].lower() or \
           "farm" in response["analogy"]["explanation"].lower()

def test_empty_profile_returns_error():
    """Edge case: User with no profile data"""
    user_id = create_test_user()
    
    with pytest.raises(InsufficientKnowledgeError):
        generate_analogy(user_id, "Kubernetes")

def test_invalid_language_code():
    """Error condition: Invalid language"""
    user_id = create_test_user()
    
    with pytest.raises(ValidationError):
        update_profile(user_id, {"language": "invalid"})
```

### Integration Testing

**Scope**: Test interactions between Lambda functions, databases, and AI services

**Key Integration Tests**:
1. End-to-end learning flow: Profile creation → Concept request → Analogy generation → Concept map
2. Spaced repetition cycle: Learn concept → Schedule review → Retrieve due reviews → Update interval
3. Multilingual flow: Set language → Generate analogy → Translate → Verify language
4. Voice learning flow: Audio input → Transcribe → Process → Generate response → TTS

### Load Testing

**Tool**: Locust or AWS Distributed Load Testing

**Scenarios**:
- 1000 concurrent users requesting analogies
- 100 users simultaneously updating profiles
- Burst traffic: 0 to 500 requests/second in 10 seconds

**Metrics to Monitor**:
- API Gateway latency (p50, p95, p99)
- Lambda cold start frequency
- DynamoDB throttling events
- Bedrock API rate limits

### Monitoring and Observability

**CloudWatch Metrics**:
- Custom metric: `AnalogyGenerationTime`
- Custom metric: `ConceptMapComplexity` (node count)
- Custom metric: `UserEngagementScore`

**X-Ray Tracing**:
- Trace analogy generation pipeline end-to-end
- Identify bottlenecks in vector search
- Monitor Bedrock API call latency

**Alarms**:
- Lambda error rate > 1%
- API Gateway 5xx errors > 0.5%
- DynamoDB consumed capacity > 80%
- Bedrock throttling events > 10/minute

### Test Data Management

**Synthetic User Profiles**:
```python
TEST_PROFILES = [
    {
        "name": "Rural Farmer",
        "profession": "farmer",
        "hobbies": ["cricket", "cooking"],
        "interests": ["agriculture", "weather"],
        "language": "hi"
    },
    {
        "name": "Software Developer",
        "profession": "software engineer",
        "hobbies": ["gaming", "reading"],
        "interests": ["technology", "AI"],
        "language": "en"
    },
    {
        "name": "Teacher",
        "profession": "school teacher",
        "hobbies": ["music", "gardening"],
        "interests": ["education", "child development"],
        "language": "ta"
    }
]
```

**Test Concepts**:
- Simple: "API", "Database", "Cloud Computing"
- Intermediate: "Kubernetes", "Machine Learning", "Blockchain"
- Complex: "Distributed Consensus", "Quantum Computing", "Neural Architecture Search"

### Continuous Integration

**Pipeline Stages**:
1. **Lint**: `pylint`, `black`, `mypy` for Python; `eslint`, `prettier` for TypeScript
2. **Unit Tests**: Run all unit tests with coverage > 80%
3. **Property Tests**: Run property tests with 100 iterations
4. **Integration Tests**: Deploy to test environment and run integration suite
5. **Security Scan**: `bandit` for Python, `npm audit` for Node.js
6. **Deploy**: Deploy to staging, then production with approval

**Test Coverage Requirements**:
- Overall coverage: > 80%
- Critical paths (analogy generation, gap analysis): > 95%
- Error handling: > 90%
