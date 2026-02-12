# Design Document

## Overview

The Social Media Sentiment & Trend Analyzer is a full-stack web application that combines a React frontend with a Node.js/Express backend, integrated with Google Cloud AI services for sentiment analysis and text generation. The system uses BigQuery as the primary data store for social media posts (optimized for time-series queries and large-scale analytics) and MongoDB for configuration, caching, and alert management.

The architecture prioritizes cost efficiency and scalability by batching AI service calls, implementing aggressive caching, and using time-based data partitioning. The system is designed to handle bursts of social media data while maintaining sub-second query response times for dashboard visualizations.

## Architecture

### High-Level Architecture

```mermaid
graph TB
    subgraph "Frontend Layer"
        A[React Dashboard]
        B[Visualization Components]
        C[Alert Notifications]
    end
    
    subgraph "Backend Layer"
        D[Express.js API Server]
        E[Sentiment Analysis Service]
        F[Trend Detection Service]
        G[Alert Service]
        H[Summary Generation Service]
    end
    
    subgraph "Data Layer"
        I[MongoDB - Config & Cache]
        J[BigQuery - Post Storage]
    end
    
    subgraph "Google Cloud Services"
        K[Vertex AI - Text Classification]
        L[Gemini API - Summaries]
    end
    
    A --> D
    B --> D
    C --> D
    D --> E
    D --> F
    D --> G
    D --> H
    E --> K
    E --> J
    F --> J
    G --> D
    H --> L
    H --> J
    D --> I
```

### Technology Stack

**Frontend:**
- React 18+ with functional components and hooks
- Recharts for data visualization (lightweight, no external dependencies)
- Axios for API communication with request/response interceptors
- TanStack Query (React Query) for server state management and automatic caching
- Tailwind CSS for styling

**Backend:**
- Node.js 20+ with Express.js framework
- Google Cloud Client Libraries:
  - @google-cloud/aiplatform (Vertex AI sentiment analysis)
  - @google-cloud/bigquery (post storage and analytics)
  - @google-cloud/vertexai (Gemini text generation)
- Mongoose ODM for MongoDB operations
- node-cron for scheduled background tasks (risk monitoring, alert checking)
- Winston for structured logging
- express-rate-limit for API rate limiting
- jsonwebtoken for authentication

**Data Storage:**
- BigQuery: Primary post storage (partitioned by timestamp for efficient time-range queries)
- MongoDB: Configuration, user sessions, alerts, and query result caching
- Redis (optional): High-frequency cache for dashboard data (5-minute TTL)

**Infrastructure:**
- Google Cloud Platform (Vertex AI, Gemini API, BigQuery)
- MongoDB Atlas or self-hosted MongoDB
- Environment-based configuration with validation

## Components and Interfaces

### Frontend Components

#### 1. Dashboard Component
- **Responsibility**: Main container orchestrating all child components and managing global state
- **State**: 
  - `timeRange: { start: Date, end: Date }` - Currently selected time filter
  - `refreshInterval: number` - Auto-refresh timer (30s default)
  - `isLoading: boolean` - Global loading state
- **Data Fetching**: Uses TanStack Query with 30-second stale time
- **Key Methods**: 
  - `handleTimeRangeChange(range)`: Updates time filter and invalidates cached queries
  - `handleManualRefresh()`: Forces immediate data refetch

#### 2. SentimentChart Component
- **Responsibility**: Renders sentiment distribution as a pie chart
- **Props**: `data: { positive: number, negative: number, neutral: number, total: number }`
- **Validation**: Ensures percentages sum to 100% ± 0.1%
- **Accessibility**: Includes ARIA labels and keyboard navigation

#### 3. TrendingTopics Component
- **Responsibility**: Displays top 10 trending topics with sentiment indicators
- **Props**: `topics: Array<{ topic: string, count: number, avgSentiment: number }>`
- **Features**: 
  - Sortable by count or sentiment (client-side)
  - Click to filter posts by topic
  - Color-coded sentiment badges (green/amber/red)

#### 4. BrandRiskIndicator Component
- **Responsibility**: Displays current brand risk level with visual emphasis
- **Props**: `riskLevel: 'low' | 'medium' | 'high', riskScore: number, postCount: number`
- **Styling**: 
  - Low: #10B981 (green) background
  - Medium: #F59E0B (amber) background with pulse animation
  - High: #DC2626 (red) background with pulse animation and alert icon

#### 5. CrisisAlertPanel Component
- **Responsibility**: Shows active crisis alerts with dismissal capability
- **Props**: `alerts: Array<Alert>`
- **Features**: 
  - Real-time updates via polling (10-second interval when alerts exist)
  - Dismissible alerts (marks as acknowledged, doesn't delete)
  - Expandable to show sample posts

#### 6. PostList Component
- **Responsibility**: Displays paginated, filtered list of posts
- **Props**: `posts: Array<Post>, totalCount: number, currentPage: number, onPageChange: (page) => void`
- **Features**: 
  - Virtual scrolling for performance (react-window)
  - Sentiment badges with scores
  - Truncated text with "show more" expansion
  - Copy post ID to clipboard

#### 7. FilterControls Component
- **Responsibility**: Provides comprehensive filtering interface
- **Props**: `onFilterChange: (filters: FilterState) => void, initialFilters: FilterState`
- **Controls**: 
  - Sentiment multi-select (checkboxes)
  - Date range picker with presets (24h, 7d, 30d, custom)
  - Keyword search with debounce (500ms)
  - Hashtag search with autocomplete
  - Clear all filters button

#### 8. AuthenticationGuard Component
- **Responsibility**: Protects routes requiring authentication
- **Behavior**: Redirects to login if no valid JWT token
- **Token Refresh**: Automatically refreshes tokens 5 minutes before expiry

### Backend Services

#### 1. SentimentAnalysisService
```javascript
class SentimentAnalysisService {
  // Analyzes single post using Vertex AI
  async analyzeSentiment(postText: string): Promise<{ 
    category: 'positive' | 'negative' | 'neutral', 
    score: number 
  }>
  
  // Batch analyzes up to 100 posts (Vertex AI limit)
  async batchAnalyzeSentiment(posts: Array<Post>): Promise<Array<{
    postId: string,
    category: string,
    score: number
  }>>
  
  // Retrieves cached sentiment distribution from MongoDB
  async getSentimentDistribution(timeRange: TimeRange): Promise<{
    positive: number,
    negative: number,
    neutral: number,
    total: number
  }>
  
  // Maps sentiment score to category using thresholds
  private categorizeSentiment(score: number): string
}
```

#### 2. TrendDetectionService
```javascript
class TrendDetectionService {
  // Extracts hashtags using regex pattern /#(\w+)/g
  extractHashtags(postText: string): Array<string>
  
  // Queries BigQuery for topic frequencies within time window
  async identifyTrendingTopics(timeWindow: number, threshold: number): Promise<Array<{
    topic: string,
    count: number,
    avgSentiment: number,
    posts: Array<string>
  }>>
  
  // Calculates average sentiment for specific topic
  async getTopicSentiment(topic: string, timeRange: TimeRange): Promise<number>
  
  // Caches trending topics in MongoDB (5-minute TTL)
  private cacheTrendingTopics(topics: Array<TrendingTopic>): Promise<void>
}
```

#### 3. BrandRiskService
```javascript
class BrandRiskService {
  // Calculates risk level using time-weighted sentiment
  async calculateBrandRisk(brand: string, timeWindow: number): Promise<{
    riskLevel: 'low' | 'medium' | 'high',
    riskScore: number,
    negativePercentage: number,
    postCount: number
  }>
  
  // Background job checking for risk level changes
  async monitorRiskChanges(): Promise<void>
  
  // Retrieves historical risk data for charting
  async getRiskHistory(brand: string, timeRange: TimeRange): Promise<Array<{
    timestamp: Date,
    riskLevel: string,
    riskScore: number
  }>>
  
  // Applies time-decay weighting (recent 25% of window gets 2x weight)
  private calculateTimeWeightedScore(posts: Array<Post>, timeWindow: number): number
}
```

#### 4. AlertService
```javascript
class AlertService {
  // Checks if crisis conditions are met
  async checkForCrisisConditions(): Promise<Array<{
    severity: 'medium' | 'high' | 'critical',
    affectedTopics: Array<string>,
    samplePosts: Array<string>,
    percentageIncrease: number
  }>>
  
  // Sends alert to configured channels (email, webhook, in-app)
  async sendAlert(alert: CrisisAlert, channels: Array<string>): Promise<void>
  
  // Consolidates alerts within 30-minute window
  async consolidateAlerts(alerts: Array<CrisisAlert>): Promise<CrisisAlert>
  
  // Retrieves active (unacknowledged) alerts
  async getActiveAlerts(): Promise<Array<CrisisAlert>>
  
  // Marks alert as acknowledged
  async acknowledgeAlert(alertId: string, userId: string): Promise<void>
}
```

#### 5. SummaryGenerationService
```javascript
class SummaryGenerationService {
  // Generates AI summary using Gemini with retry logic
  async generateSummary(posts: Array<Post>, timeRange: TimeRange): Promise<string>
  
  // Fallback statistical summary when Gemini fails
  async generateFallbackSummary(stats: {
    sentimentDistribution: object,
    topTopics: Array<string>,
    totalPosts: number
  }): Promise<string>
  
  // Checks cache before generating new summary
  private async getCachedSummary(cacheKey: string): Promise<string | null>
  
  // Caches summary for 5 minutes
  private async cacheSummary(cacheKey: string, summary: string): Promise<void>
}
```

#### 6. BigQueryService
```javascript
class BigQueryService {
  // Inserts single post with auto-generated UUID
  async insertPost(post: Post): Promise<string>
  
  // Batch inserts posts (uses streaming insert for 100+ posts)
  async batchInsertPosts(posts: Array<Post>): Promise<void>
  
  // Queries posts with filters and pagination
  async queryPosts(filters: {
    sentiment?: string,
    keyword?: string,
    hashtag?: string,
    startDate?: Date,
    endDate?: Date,
    limit: number,
    offset: number
  }): Promise<{ posts: Array<Post>, total: number }>
  
  // Aggregates sentiment counts for time range
  async aggregateSentiment(timeRange: TimeRange): Promise<{
    positive: number,
    negative: number,
    neutral: number
  }>
  
  // Deletes posts older than retention period (90 days)
  async deleteExpiredPosts(): Promise<number>
  
  // Implements exponential backoff retry (1s, 2s, 4s delays)
  private async retryWithBackoff<T>(operation: () => Promise<T>): Promise<T>
}
```

#### 7. AuthenticationService
```javascript
class AuthenticationService {
  // Validates credentials and generates JWT token
  async login(username: string, password: string): Promise<{
    token: string,
    refreshToken: string,
    expiresIn: number
  }>
  
  // Verifies JWT token and extracts user info
  async verifyToken(token: string): Promise<{
    userId: string,
    role: 'user' | 'admin'
  }>
  
  // Refreshes expired token
  async refreshToken(refreshToken: string): Promise<string>
  
  // Tracks failed login attempts for rate limiting
  async recordFailedLogin(source: string): Promise<void>
  
  // Checks if source is blocked due to failed attempts
  async isBlocked(source: string): Promise<boolean>
}
```

### API Endpoints

#### POST /api/auth/login
- **Purpose**: Authenticate user and receive JWT token
- **Request Body**: `{ username: string, password: string }`
- **Response**: `{ token: string, refreshToken: string, expiresIn: number, role: string }`
- **Rate Limit**: 5 requests per 15 minutes per IP

#### POST /api/auth/refresh
- **Purpose**: Refresh expired JWT token
- **Request Body**: `{ refreshToken: string }`
- **Response**: `{ token: string, expiresIn: number }`

#### POST /api/posts
- **Purpose**: Submit new posts for storage and analysis
- **Authentication**: Required (JWT)
- **Request Body**: `{ posts: Array<{ text: string, author: string, platform: string, timestamp: string }> }`
- **Response**: `{ success: boolean, insertedIds: Array<string>, failedCount: number }`
- **Rate Limit**: 100 requests per hour per user

#### GET /api/sentiment/distribution
- **Purpose**: Get sentiment distribution for time range
- **Authentication**: Required (JWT)
- **Query Params**: `startDate: ISO8601, endDate: ISO8601`
- **Response**: `{ positive: number, negative: number, neutral: number, total: number }`
- **Cache**: 30 seconds

#### GET /api/trends/topics
- **Purpose**: Get trending topics
- **Authentication**: Required (JWT)
- **Query Params**: `timeWindow: number (minutes), limit: number (default 10, max 50)`
- **Response**: `{ topics: Array<{ topic: string, count: number, avgSentiment: number, posts: Array<string> }> }`
- **Cache**: 5 minutes

#### GET /api/risk/assessment
- **Purpose**: Get brand risk assessment
- **Authentication**: Required (JWT)
- **Query Params**: `brand: string, timeWindow: number (minutes)`
- **Response**: `{ riskLevel: string, riskScore: number, negativePercentage: number, postCount: number }`
- **Cache**: 1 minute

#### GET /api/alerts/active
- **Purpose**: Get active (unacknowledged) crisis alerts
- **Authentication**: Required (JWT)
- **Response**: `{ alerts: Array<{ id: string, severity: string, affectedTopics: Array<string>, samplePosts: Array<string>, triggeredAt: string }> }`

#### POST /api/alerts/:id/acknowledge
- **Purpose**: Mark alert as acknowledged
- **Authentication**: Required (JWT)
- **Response**: `{ success: boolean }`

#### POST /api/summary/generate
- **Purpose**: Generate AI summary of sentiment trends
- **Authentication**: Required (JWT)
- **Request Body**: `{ timeRange: { start: string, end: string }, topics?: Array<string> }`
- **Response**: `{ summary: string, generatedAt: string, cached: boolean }`
- **Timeout**: 15 seconds
- **Rate Limit**: 10 requests per hour per user

#### GET /api/posts/search
- **Purpose**: Search and filter posts with pagination
- **Authentication**: Required (JWT)
- **Query Params**: `sentiment?: string, keyword?: string, hashtag?: string, startDate?: string, endDate?: string, limit?: number (default 50, max 100), offset?: number`
- **Response**: `{ posts: Array<Post>, total: number, page: number, pageSize: number }`

#### GET /api/config
- **Purpose**: Get current configuration values
- **Authentication**: Required (Admin role)
- **Response**: `{ crisisThreshold: number, riskTimeWindow: number, trendingThreshold: number, trendingTimeWindow: number, lowRiskThreshold: number, highRiskThreshold: number, consolidationWindow: number }`

#### PUT /api/config
- **Purpose**: Update configuration values
- **Authentication**: Required (Admin role)
- **Request Body**: Partial configuration object
- **Response**: `{ success: boolean, updatedConfig: object, errors?: Array<string> }`

## Data Models

### Post Model (BigQuery Schema)
```sql
CREATE TABLE posts (
  id STRING NOT NULL,  -- UUID v4
  text STRING NOT NULL,  -- Original post text (max 10,000 chars)
  author STRING,  -- Author identifier (not PII)
  platform STRING NOT NULL,  -- Source platform (twitter, facebook, etc.)
  timestamp TIMESTAMP NOT NULL,  -- Post creation time
  sentiment STRING,  -- positive, negative, or neutral
  sentiment_score FLOAT64,  -- Range: -1.0 to 1.0
  hashtags ARRAY<STRING>,  -- Extracted hashtags (lowercase, no # symbol)
  analyzed_at TIMESTAMP,  -- When sentiment analysis completed
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP()
)
PARTITION BY DATE(timestamp)
CLUSTER BY sentiment, platform;

-- Indexes for common queries
CREATE INDEX idx_posts_timestamp ON posts(timestamp DESC);
CREATE INDEX idx_posts_sentiment ON posts(sentiment);
```

### Configuration Model (MongoDB)
```javascript
{
  _id: ObjectId,
  crisisThreshold: Number,  // Percentage increase to trigger alert (default: 50)
  riskTimeWindow: Number,  // Minutes for risk calculation (default: 60)
  trendingThreshold: Number,  // Min post count for trending (default: 10)
  trendingTimeWindow: Number,  // Minutes for trending window (default: 60)
  lowRiskThreshold: Number,  // Percentage for low risk (default: 15)
  highRiskThreshold: Number,  // Percentage for high risk (default: 30)
  consolidationWindow: Number,  // Minutes to consolidate alerts (default: 30)
  alertChannels: Array<String>,  // ['email', 'webhook', 'in-app']
  webhookUrl: String,  // Optional webhook endpoint
  updatedBy: String,  // Admin user ID
  updatedAt: Date,
  createdAt: Date
}
```

### Alert Model (MongoDB)
```javascript
{
  _id: ObjectId,
  type: String,  // 'crisis' or 'risk'
  severity: String,  // 'medium', 'high', 'critical'
  triggeredAt: Date,
  acknowledgedAt: Date,  // null if not acknowledged
  acknowledgedBy: String,  // User ID who acknowledged
  affectedTopics: Array<String>,  // Top 5 topics
  samplePostIds: Array<String>,  // Up to 5 post IDs
  percentageIncrease: Number,  // For crisis alerts
  negativePercentage: Number,  // For risk alerts
  notificationsSent: Array<{
    channel: String,
    sentAt: Date,
    success: Boolean,
    error: String
  }>,
  status: String,  // 'active', 'acknowledged', 'resolved'
  consolidatedFrom: Array<ObjectId>,  // IDs of alerts consolidated into this one
  createdAt: Date
}

// Indexes
db.alerts.createIndex({ status: 1, triggeredAt: -1 });
db.alerts.createIndex({ triggeredAt: -1 });
```

### User Model (MongoDB)
```javascript
{
  _id: ObjectId,
  username: String,  // Unique
  passwordHash: String,  // bcrypt hash
  role: String,  // 'user' or 'admin'
  email: String,
  failedLoginAttempts: Number,
  blockedUntil: Date,  // null if not blocked
  lastLoginAt: Date,
  createdAt: Date,
  updatedAt: Date
}

// Indexes
db.users.createIndex({ username: 1 }, { unique: true });
db.users.createIndex({ email: 1 });
```

### Cache Model (MongoDB)
```javascript
{
  _id: ObjectId,
  key: String,  // Unique cache key (e.g., 'trending_topics_60')
  value: Mixed,  // Cached data (any type)
  expiresAt: Date,  // TTL for automatic deletion
  createdAt: Date
}

// TTL Index for automatic expiration
db.cache.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 });
db.cache.createIndex({ key: 1 }, { unique: true });
```

### Session Model (MongoDB)
```javascript
{
  _id: ObjectId,
  userId: ObjectId,
  refreshToken: String,  // Hashed refresh token
  expiresAt: Date,
  ipAddress: String,
  userAgent: String,
  createdAt: Date
}

// TTL Index
db.sessions.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 });
db.sessions.createIndex({ userId: 1 });
```


## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system-essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Post storage round-trip preservation
*For any* valid post with text, author, platform, and timestamp, storing the post and then retrieving it by ID should return a post with identical values for these fields.
**Validates: Requirements 1.4**

### Property 2: Unique post identifiers
*For any* batch of posts stored in a single operation, all generated post IDs should be unique with no duplicates.
**Validates: Requirements 1.3**

### Property 3: Sentiment score range constraint
*For any* post that undergoes sentiment analysis, the resulting sentiment score should be between -1.0 and 1.0 inclusive.
**Validates: Requirements 2.2**

### Property 4: Sentiment category consistency
*For any* post with sentiment score less than -0.2, the sentiment category should be 'negative'; for scores greater than 0.2, category should be 'positive'; for scores between -0.2 and 0.2 inclusive, category should be 'neutral'.
**Validates: Requirements 2.3, 2.4, 2.5**

### Property 5: Hashtag extraction completeness
*For any* post text containing hashtags (pattern #\w+), all hashtags should be extracted, converted to lowercase, and stored in the hashtags array without the # symbol.
**Validates: Requirements 3.1, 3.2**

### Property 6: Topic frequency accuracy
*For any* set of posts within a time window, the frequency count for each topic should equal the number of posts containing that topic in their hashtags array.
**Validates: Requirements 3.4**

### Property 7: Topic ranking monotonicity
*For any* list of trending topics, each topic's frequency should be greater than or equal to the next topic's frequency (descending order).
**Validates: Requirements 3.4**

### Property 8: Average sentiment calculation correctness
*For any* trending topic, the average sentiment score should equal the arithmetic mean of sentiment scores from all posts containing that topic, within 0.01 tolerance.
**Validates: Requirements 3.7**

### Property 9: Brand risk classification thresholds
*For any* brand risk calculation where negative percentage is less than lowRiskThreshold, risk level should be 'low'; where negative percentage is greater than or equal to highRiskThreshold, risk level should be 'high'; otherwise risk level should be 'medium'.
**Validates: Requirements 4.2, 4.3, 4.4**

### Property 10: Time-decay weighting correctness
*For any* two posts in a risk calculation where post A is in the most recent 25% of the time window and post B is older, post A's weight should be 2.0 and post B's weight should be 1.0.
**Validates: Requirements 4.5**

### Property 11: Crisis alert trigger threshold
*For any* comparison between current and previous time windows, when negative post count increases by configured crisis threshold percentage or more, a crisis alert should be triggered.
**Validates: Requirements 5.2**

### Property 12: Alert severity classification
*For any* crisis alert where percentage increase is 50-100%, severity should be 'medium'; 100-200% should be 'high'; above 200% should be 'critical'.
**Validates: Requirements 5.3**

### Property 13: Alert consolidation within window
*For any* set of crisis alerts where all triggeredAt timestamps are within the configured consolidation window, they should be consolidated into a single alert with the highest severity.
**Validates: Requirements 5.5**

### Property 14: Sentiment percentage sum invariant
*For any* sentiment distribution calculation, the sum of positive percentage, negative percentage, and neutral percentage should equal 100% within 0.1% tolerance.
**Validates: Requirements 6.2**

### Property 15: Time range filtering correctness
*For any* query with start and end date filters, all returned posts should have timestamps greater than or equal to start date AND less than or equal to end date.
**Validates: Requirements 6.3, 9.4**

### Property 16: Filter conjunction correctness
*For any* query with multiple filters (sentiment, keyword, hashtag, date range), all returned posts should satisfy ALL filter criteria simultaneously.
**Validates: Requirements 9.5**

### Property 17: Configuration validation boundaries
*For any* configuration update, values should be accepted if within specified ranges (crisis: 10-500, risk window: 15-1440, trending: 5-1000, low risk: 5-50, high risk: lowRisk+1 to 80) and rejected otherwise.
**Validates: Requirements 8.2, 8.3, 8.4, 8.5, 8.6, 8.7**

### Property 18: Exponential backoff retry timing
*For any* failed operation with retry logic, retry delays should follow the pattern: 1 second, 2 seconds, 4 seconds, with maximum 3 attempts.
**Validates: Requirements 2.7, 10.4**

### Property 19: Batch processing threshold
*For any* post submission where count is 100 or greater, the system should use batch insertion; for counts less than 100, individual insertion is acceptable.
**Validates: Requirements 10.1**

### Property 20: Authentication session expiry
*For any* user session, when inactivity duration exceeds 8 hours (28,800 seconds), the session should be terminated and require re-authentication.
**Validates: Requirements 11.4**

## Error Handling

### Retry Strategy

**Exponential Backoff Configuration:**
```javascript
const retryConfig = {
  initialDelay: 1000,  // 1 second
  multiplier: 2,
  maxAttempts: 3,
  retryableErrors: [
    'ECONNRESET',
    'ETIMEDOUT',
    'ENOTFOUND',
    'RATE_LIMIT_EXCEEDED',
    'SERVICE_UNAVAILABLE'
  ]
};
```

**Retry Logic:**
- Attempt 1: Immediate
- Attempt 2: After 1 second delay
- Attempt 3: After 2 seconds delay
- Attempt 4: After 4 seconds delay
- After 3 retries: Log error and return failure to client

### Error Categories and Responses

#### 1. External Service Errors

**Vertex AI Failures:**
- Error: Rate limit exceeded
- Action: Queue request for retry after 60 seconds
- Response: 429 Too Many Requests with Retry-After header

**Gemini API Failures:**
- Error: Service unavailable or timeout
- Action: Generate fallback statistical summary
- Response: 200 OK with `{ summary: string, fallback: true }`

**BigQuery Errors:**
- Error: Connection timeout
- Action: Retry with exponential backoff
- Response: 503 Service Unavailable after all retries fail

#### 2. Data Validation Errors

**Invalid Post Format:**
```javascript
{
  success: false,
  error: {
    code: "VALIDATION_ERROR",
    message: "Post validation failed",
    details: {
      missingFields: ["text", "timestamp"],
      invalidFields: { "timestamp": "Invalid ISO8601 format" }
    }
  }
}
```
- HTTP Status: 400 Bad Request

**Invalid Configuration:**
```javascript
{
  success: false,
  error: {
    code: "CONFIG_VALIDATION_ERROR",
    message: "Configuration validation failed",
    details: {
      crisisThreshold: "Value 600 exceeds maximum of 500",
      highRiskThreshold: "Value 15 must be greater than lowRiskThreshold (20)"
    }
  }
}
```
- HTTP Status: 400 Bad Request

#### 3. Authentication/Authorization Errors

**Invalid Token:**
```javascript
{
  success: false,
  error: {
    code: "INVALID_TOKEN",
    message: "JWT token is invalid or expired"
  }
}
```
- HTTP Status: 401 Unauthorized

**Insufficient Permissions:**
```javascript
{
  success: false,
  error: {
    code: "FORBIDDEN",
    message: "Admin role required for this operation"
  }
}
```
- HTTP Status: 403 Forbidden

**Rate Limit Exceeded:**
```javascript
{
  success: false,
  error: {
    code: "RATE_LIMIT_EXCEEDED",
    message: "Too many requests",
    details: {
      limit: 100,
      window: "1 hour",
      retryAfter: 3600
    }
  }
}
```
- HTTP Status: 429 Too Many Requests

#### 4. Business Logic Errors

**Insufficient Data:**
```javascript
{
  success: false,
  error: {
    code: "INSUFFICIENT_DATA",
    message: "Not enough posts for trend analysis",
    details: {
      required: 10,
      found: 3
    }
  }
}
```
- HTTP Status: 200 OK (not a server error)

**Queue Full:**
```javascript
{
  success: false,
  error: {
    code: "QUEUE_FULL",
    message: "Summary generation queue is full",
    details: {
      maxQueueSize: 10,
      retryAfter: 30
    }
  }
}
```
- HTTP Status: 503 Service Unavailable

### Logging Strategy

**Log Levels and Content:**

**ERROR:**
- All exceptions with full stack traces
- Failed external service calls after all retries
- Authentication failures
- Data corruption or inconsistency issues
- Format: `{ level: 'error', timestamp, userId, endpoint, error: { message, stack }, context }`

**WARN:**
- Retry attempts (with attempt number)
- Fallback usage (e.g., statistical summary instead of AI)
- Approaching rate limits (>80% of quota)
- Configuration changes
- Format: `{ level: 'warn', timestamp, userId, message, context }`

**INFO:**
- Successful API requests (with response time)
- Alert triggers
- Batch processing completion
- User login/logout
- Format: `{ level: 'info', timestamp, userId, endpoint, duration, statusCode }`

**DEBUG:**
- External service request/response payloads (sanitized)
- Query execution plans
- Cache hits/misses
- Format: `{ level: 'debug', timestamp, message, data }`

**Log Rotation:**
- Daily rotation
- Compress logs older than 7 days
- Delete logs older than 90 days
- Max log file size: 100MB

## Testing Strategy

### Unit Testing

**Backend Services:**
- Test each service method in isolation with mocked dependencies
- Verify correct handling of edge cases (empty arrays, null values, boundary conditions)
- Test error handling and retry logic
- Mock Google Cloud services to avoid external dependencies

**Frontend Components:**
- Test component rendering with various prop combinations
- Verify user interaction handlers (clicks, form submissions)
- Test conditional rendering based on state
- Mock API calls using MSW (Mock Service Worker)

**Key Unit Test Examples:**
- SentimentAnalysisService.analyzeSentiment() with various text inputs
- TrendDetectionService.identifyTrendingTopics() with different post volumes
- BrandRiskService.calculateBrandRisk() with varying sentiment distributions
- FilterControls component with different filter combinations

### Property-Based Testing

The system will use **fast-check** library for JavaScript/TypeScript property-based testing. Each property test should run a minimum of 100 iterations.

**Property Test Configuration:**
```javascript
import fc from 'fast-check';

// Configure test runs
const testConfig = {
  numRuns: 100,
  verbose: true,
  seed: Date.now()
};
```

**Generator Strategies:**
- **Post Generator**: Random text (10-280 chars), random author, random platform, random timestamp
- **Sentiment Score Generator**: Float between -1.0 and 1.0
- **Keyword Generator**: Array of 1-10 random words
- **Time Range Generator**: Valid start/end date pairs
- **Configuration Generator**: Valid threshold values within acceptable ranges

**Property Test Examples:**
- Generate random posts, store them, verify unique IDs (Property 2)
- Generate random posts, analyze sentiment, verify score range (Property 3)
- Generate posts with known keywords, verify frequency counts (Property 5)
- Generate time ranges, filter posts, verify all results within range (Property 17)
- Generate filter combinations, verify all results match all filters (Property 27)

### Integration Testing

**API Integration Tests:**
- Test complete request/response cycles for all endpoints
- Verify proper error responses for invalid requests
- Test authentication and authorization flows
- Verify data persistence across service boundaries

**Google Cloud Integration:**
- Test Vertex AI sentiment analysis with real API calls (in staging environment)
- Verify BigQuery insert and query operations
- Test Gemini summary generation with sample data
- Verify proper credential handling and authentication

**Database Integration:**
- Test MongoDB CRUD operations for configuration and alerts
- Verify BigQuery schema compatibility
- Test data migration and seeding scripts

### End-to-End Testing

**User Workflows:**
- Submit posts → View sentiment distribution → Verify accuracy
- Apply filters → Search posts → Verify filtered results
- Trigger crisis conditions → Verify alert generation and notification
- Update configuration → Verify new thresholds applied
- Request summary → Verify Gemini integration and display

**Performance Testing:**
- Load test with 10,000+ posts
- Verify batch processing efficiency
- Test concurrent user sessions
- Measure API response times under load

### Test Organization

```
tests/
├── unit/
│   ├── services/
│   │   ├── sentimentAnalysis.test.js
│   │   ├── trendDetection.test.js
│   │   └── brandRisk.test.js
│   └── components/
│       ├── Dashboard.test.jsx
│       └── SentimentChart.test.jsx
├── property/
│   ├── storage.property.test.js
│   ├── sentiment.property.test.js
│   ├── filtering.property.test.js
│   └── validation.property.test.js
├── integration/
│   ├── api.integration.test.js
│   ├── bigquery.integration.test.js
│   └── vertexai.integration.test.js
└── e2e/
    ├── userWorkflows.e2e.test.js
    └── alerting.e2e.test.js
```

### Testing Tools
- **Jest**: Unit and integration testing framework
- **fast-check**: Property-based testing library
- **React Testing Library**: Component testing
- **Supertest**: API endpoint testing
- **MSW**: API mocking for frontend tests
- **Playwright or Cypress**: End-to-end testing

## Implementation Notes

### Google Cloud Setup and Cost Management

**Required APIs:**
1. Vertex AI API (AI Platform)
2. Generative AI API (Gemini)
3. BigQuery API

**Service Account Permissions:**
```
roles/aiplatform.user
roles/bigquery.dataEditor
roles/bigquery.jobUser
roles/aiplatform.serviceAgent
```

**Cost Optimization Strategies:**

**Vertex AI Sentiment Analysis:**
- Cost: ~$0.001 per prediction
- Batch requests (100 posts) to reduce overhead
- Cache sentiment results in MongoDB (never re-analyze same post)
- Estimated cost for 1M posts: $1,000

**Gemini API:**
- Cost: ~$0.00025 per 1K characters (input) + $0.0005 per 1K characters (output)
- Cache summaries for 5 minutes
- Limit summary requests to 10 per hour per user
- Use fallback statistical summaries when possible
- Estimated cost for 1,000 summaries: $5-10

**BigQuery:**
- Storage: $0.02 per GB per month
- Queries: $5 per TB scanned
- Use partitioning by date to reduce scan costs
- Use clustering on sentiment and platform
- Implement 90-day retention to control storage costs
- Estimated cost for 10M posts: $50/month storage + $20/month queries

**Total Estimated Monthly Cost (1M posts/month, 1K summaries):**
- Vertex AI: $1,000
- Gemini: $10
- BigQuery: $70
- **Total: ~$1,080/month**

### Environment Variables

```bash
# Google Cloud
GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account-key.json
GOOGLE_CLOUD_PROJECT=your-project-id
BIGQUERY_DATASET=social_media_data
BIGQUERY_TABLE=posts

# MongoDB
MONGODB_URI=mongodb+srv://user:pass@cluster.mongodb.net/sentiment_analyzer
MONGODB_DB_NAME=sentiment_analyzer

# Redis (optional)
REDIS_URL=redis://localhost:6379
REDIS_ENABLED=false

# API Configuration
PORT=5000
NODE_ENV=production
API_BASE_URL=https://api.yourdomain.com

# JWT Authentication
JWT_SECRET=your-256-bit-secret-key-here
JWT_EXPIRY=8h
REFRESH_TOKEN_EXPIRY=7d

# Rate Limiting
RATE_LIMIT_WINDOW_MS=3600000  # 1 hour
RATE_LIMIT_MAX_REQUESTS=100

# Alert Configuration
ALERT_EMAIL_ENABLED=true
ALERT_EMAIL_FROM=alerts@yourdomain.com
ALERT_EMAIL_TO=team@yourdomain.com
ALERT_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASS=your-app-password

# Gemini Configuration
GEMINI_MODEL=gemini-1.5-flash  # Cheaper than gemini-pro
GEMINI_MAX_TOKENS=500
GEMINI_TEMPERATURE=0.7

# System Configuration
DEFAULT_CRISIS_THRESHOLD=50
DEFAULT_RISK_TIME_WINDOW=60
DEFAULT_TRENDING_THRESHOLD=10
DEFAULT_TRENDING_TIME_WINDOW=60
DEFAULT_LOW_RISK_THRESHOLD=15
DEFAULT_HIGH_RISK_THRESHOLD=30
DEFAULT_CONSOLIDATION_WINDOW=30

# Data Retention
POST_RETENTION_DAYS=90
LOG_RETENTION_DAYS=90
```

### Performance Optimizations

**Database Indexing:**
```javascript
// BigQuery - Automatic via partitioning and clustering
// MongoDB indexes
db.alerts.createIndex({ status: 1, triggeredAt: -1 });
db.cache.createIndex({ key: 1 }, { unique: true });
db.cache.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 });
db.users.createIndex({ username: 1 }, { unique: true });
db.sessions.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 });
```

**Caching Strategy:**
- Sentiment distribution: 30-second cache
- Trending topics: 5-minute cache
- Brand risk: 1-minute cache
- AI summaries: 5-minute cache
- Configuration: In-memory cache, invalidate on update

**Query Optimization:**
- Use BigQuery partitioning to scan only relevant dates
- Limit result sets to 50-100 items with pagination
- Use projection to fetch only required fields
- Implement connection pooling (MongoDB: 10-50 connections)

**Frontend Optimization:**
- Lazy load components with React.lazy()
- Implement virtual scrolling for post lists (react-window)
- Debounce search inputs (500ms)
- Use TanStack Query for automatic request deduplication
- Compress API responses with gzip

### Security Implementation

**Input Validation:**
```javascript
// Sanitize post text to prevent XSS
const sanitizePostText = (text) => {
  return text
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#x27;')
    .substring(0, 10000);  // Max length
};

// Validate configuration values
const validateConfig = (config) => {
  const rules = {
    crisisThreshold: { min: 10, max: 500 },
    riskTimeWindow: { min: 15, max: 1440 },
    trendingThreshold: { min: 5, max: 1000 },
    lowRiskThreshold: { min: 5, max: 50 },
    highRiskThreshold: { min: config.lowRiskThreshold + 1, max: 80 }
  };
  // Validation logic...
};
```

**Authentication Flow:**
1. User submits credentials
2. Server validates against MongoDB users collection
3. Generate JWT token (8-hour expiry) and refresh token (7-day expiry)
4. Store refresh token hash in sessions collection
5. Client stores tokens in httpOnly cookies
6. Client includes JWT in Authorization header for API requests
7. Server validates JWT on each request
8. Client refreshes token 5 minutes before expiry

**Rate Limiting:**
```javascript
// Per-endpoint rate limits
const rateLimits = {
  '/api/auth/login': { windowMs: 900000, max: 5 },  // 5 per 15 min
  '/api/posts': { windowMs: 3600000, max: 100 },  // 100 per hour
  '/api/summary/generate': { windowMs: 3600000, max: 10 },  // 10 per hour
  'default': { windowMs: 3600000, max: 1000 }  // 1000 per hour
};
```

**Data Privacy:**
- Never store email addresses or phone numbers from posts
- Hash all passwords with bcrypt (cost factor: 12)
- Implement automatic data deletion after 90 days
- Provide data export and deletion endpoints for GDPR compliance
- Log all data access for audit trails

### Deployment Considerations

**Scaling Strategy:**
- Horizontal scaling: Run multiple Express instances behind load balancer
- Database scaling: Use MongoDB replica sets, BigQuery auto-scales
- Caching: Add Redis for distributed caching across instances
- Background jobs: Use separate worker processes for alert monitoring

**Monitoring:**
- Track API response times (target: p95 < 500ms)
- Monitor Google Cloud API quotas and costs
- Alert on error rates > 1%
- Track cache hit rates (target: > 80%)
- Monitor BigQuery query costs

**Health Checks:**
```javascript
GET /health
Response: {
  status: 'healthy',
  timestamp: '2024-02-07T12:00:00Z',
  services: {
    mongodb: 'connected',
    bigquery: 'connected',
    vertexai: 'available',
    gemini: 'available'
  },
  uptime: 86400
}
```
