# Simple Bank API Documentation

## Overview

Simple Bank is a comprehensive banking service built with Go that provides both REST HTTP APIs and gRPC APIs for managing bank accounts, users, and transfers. The service includes authentication, authorization, email verification, and asynchronous task processing.

## Table of Contents

1. [Architecture](#architecture)
2. [Configuration](#configuration)
3. [Authentication](#authentication)
4. [HTTP REST API](#http-rest-api)
5. [gRPC API](#grpc-api)
6. [Database Layer](#database-layer)
7. [Utility Functions](#utility-functions)
8. [Worker System](#worker-system)
9. [Error Handling](#error-handling)
10. [Examples](#examples)

## Architecture

The application follows a clean architecture pattern with the following layers:

- **API Layer**: HTTP REST endpoints using Gin framework
- **gRPC Layer**: gRPC services with HTTP gateway
- **Service Layer**: Business logic and transaction handling
- **Database Layer**: PostgreSQL with SQLC for type-safe queries
- **Worker Layer**: Redis-based asynchronous task processing
- **Utility Layer**: Common utilities for validation, tokens, and email

## Configuration

### Config Structure

```go
type Config struct {
    Environment          string        `mapstructure:"ENVIRONMENT"`
    DBSource             string        `mapstructure:"DB_SOURCE"`
    MigrationURL         string        `mapstructure:"MIGRATION_URL"`
    RedisAddress         string        `mapstructure:"REDIS_ADDRESS"`
    HTTPServerAddress    string        `mapstructure:"HTTP_SERVER_ADDRESS"`
    GRPCServerAddress    string        `mapstructure:"GRPC_SERVER_ADDRESS"`
    TokenSymmetricKey    string        `mapstructure:"TOKEN_SYMMETRIC_KEY"`
    AccessTokenDuration  time.Duration `mapstructure:"ACCESS_TOKEN_DURATION"`
    RefreshTokenDuration time.Duration `mapstructure:"REFRESH_TOKEN_DURATION"`
    EmailSenderName      string        `mapstructure:"EMAIL_SENDER_NAME"`
    EmailSenderAddress   string        `mapstructure:"EMAIL_SENDER_ADDRESS"`
    EmailSenderPassword  string        `mapstructure:"EMAIL_SENDER_PASSWORD"`
}
```

### Loading Configuration

```go
func LoadConfig(path string) (config Config, err error)
```

**Usage:**
```go
config, err := util.LoadConfig(".")
if err != nil {
    log.Fatal().Err(err).Msg("cannot load config")
}
```

## Authentication

### Token Maker Interface

```go
type Maker interface {
    CreateToken(username string, role string, duration time.Duration) (string, *Payload, error)
    VerifyToken(token string) (*Payload, error)
}
```

### Token Payload

```go
type Payload struct {
    ID        uuid.UUID `json:"id"`
    Username  string    `json:"username"`
    Role      string    `json:"role"`
    IssuedAt  time.Time `json:"issued_at"`
    ExpiredAt time.Time `json:"expired_at"`
}
```

### Token Implementations

#### PASETO Token Maker

```go
func NewPasetoMaker(symmetricKey string) (Maker, error)
```

**Usage:**
```go
tokenMaker, err := token.NewPasetoMaker(config.TokenSymmetricKey)
if err != nil {
    return nil, fmt.Errorf("cannot create token maker: %w", err)
}
```

#### JWT Token Maker

```go
func NewJWTMaker(secretKey string) (Maker, error)
```

**Usage:**
```go
tokenMaker, err := token.NewJWTMaker(secretKey)
if err != nil {
    return nil, fmt.Errorf("cannot create JWT maker: %w", err)
}
```

## HTTP REST API

### Server Structure

```go
type Server struct {
    config     util.Config
    store      db.Store
    tokenMaker token.Maker
    router     *gin.Engine
}
```

### Creating a Server

```go
func NewServer(config util.Config, store db.Store) (*Server, error)
```

**Usage:**
```go
server, err := api.NewServer(config, store)
if err != nil {
    log.Fatal().Err(err).Msg("cannot create server")
}
```

### Starting the Server

```go
func (server *Server) Start(address string) error
```

**Usage:**
```go
err = server.Start(config.HTTPServerAddress)
if err != nil {
    log.Fatal().Err(err).Msg("cannot start server")
}
```

### API Endpoints

#### User Management

##### Create User
- **Method**: `POST`
- **Path**: `/users`
- **Authentication**: None required
- **Request Body**:
```json
{
    "username": "johndoe",
    "password": "secretpassword",
    "full_name": "John Doe",
    "email": "john@example.com"
}
```
- **Response**:
```json
{
    "username": "johndoe",
    "full_name": "John Doe",
    "email": "john@example.com",
    "password_changed_at": "2023-01-01T00:00:00Z",
    "created_at": "2023-01-01T00:00:00Z"
}
```

##### Login User
- **Method**: `POST`
- **Path**: `/users/login`
- **Authentication**: None required
- **Request Body**:
```json
{
    "username": "johndoe",
    "password": "secretpassword"
}
```
- **Response**:
```json
{
    "session_id": "123e4567-e89b-12d3-a456-426614174000",
    "access_token": "v2.local.abc123...",
    "access_token_expires_at": "2023-01-01T01:00:00Z",
    "refresh_token": "v2.local.def456...",
    "refresh_token_expires_at": "2023-01-08T00:00:00Z",
    "user": {
        "username": "johndoe",
        "full_name": "John Doe",
        "email": "john@example.com",
        "password_changed_at": "2023-01-01T00:00:00Z",
        "created_at": "2023-01-01T00:00:00Z"
    }
}
```

##### Renew Access Token
- **Method**: `POST`
- **Path**: `/tokens/renew_access`
- **Authentication**: None required
- **Request Body**:
```json
{
    "refresh_token": "v2.local.def456..."
}
```

#### Account Management (Requires Authentication)

##### Create Account
- **Method**: `POST`
- **Path**: `/accounts`
- **Authentication**: Required
- **Request Body**:
```json
{
    "currency": "USD"
}
```
- **Response**:
```json
{
    "id": 1,
    "owner": "johndoe",
    "balance": 0,
    "currency": "USD",
    "created_at": "2023-01-01T00:00:00Z"
}
```

##### Get Account
- **Method**: `GET`
- **Path**: `/accounts/{id}`
- **Authentication**: Required
- **Response**:
```json
{
    "id": 1,
    "owner": "johndoe",
    "balance": 1000,
    "currency": "USD",
    "created_at": "2023-01-01T00:00:00Z"
}
```

##### List Accounts
- **Method**: `GET`
- **Path**: `/accounts?page_id=1&page_size=5`
- **Authentication**: Required
- **Query Parameters**:
  - `page_id`: Page number (required, min: 1)
  - `page_size`: Items per page (required, min: 5, max: 10)
- **Response**:
```json
[
    {
        "id": 1,
        "owner": "johndoe",
        "balance": 1000,
        "currency": "USD",
        "created_at": "2023-01-01T00:00:00Z"
    }
]
```

#### Transfer Management (Requires Authentication)

##### Create Transfer
- **Method**: `POST`
- **Path**: `/transfers`
- **Authentication**: Required
- **Request Body**:
```json
{
    "from_account_id": 1,
    "to_account_id": 2,
    "amount": 100,
    "currency": "USD"
}
```
- **Response**:
```json
{
    "transfer": {
        "id": 1,
        "from_account_id": 1,
        "to_account_id": 2,
        "amount": 100,
        "created_at": "2023-01-01T00:00:00Z"
    },
    "from_account": {
        "id": 1,
        "owner": "johndoe",
        "balance": 900,
        "currency": "USD",
        "created_at": "2023-01-01T00:00:00Z"
    },
    "to_account": {
        "id": 2,
        "owner": "janedoe",
        "balance": 1100,
        "currency": "USD",
        "created_at": "2023-01-01T00:00:00Z"
    },
    "from_entry": {
        "id": 1,
        "account_id": 1,
        "amount": -100,
        "created_at": "2023-01-01T00:00:00Z"
    },
    "to_entry": {
        "id": 2,
        "account_id": 2,
        "amount": 100,
        "created_at": "2023-01-01T00:00:00Z"
    }
}
```

## gRPC API

### Service Definition

```protobuf
service SimpleBank {
    rpc CreateUser (CreateUserRequest) returns (CreateUserResponse);
    rpc UpdateUser (UpdateUserRequest) returns (UpdateUserResponse);
    rpc LoginUser (LoginUserRequest) returns (LoginUserResponse);
    rpc VerifyEmail (VerifyEmailRequest) returns (VerifyEmailResponse);
}
```

### Server Structure

```go
type Server struct {
    pb.UnimplementedSimpleBankServer
    config          util.Config
    store           db.Store
    tokenMaker      token.Maker
    taskDistributor worker.TaskDistributor
}
```

### Creating a gRPC Server

```go
func NewServer(config util.Config, store db.Store, taskDistributor worker.TaskDistributor) (*Server, error)
```

### gRPC Methods

#### CreateUser

**Request:**
```protobuf
message CreateUserRequest {
    string username = 1;
    string full_name = 2;
    string email = 3;
    string password = 4;
}
```

**Response:**
```protobuf
message CreateUserResponse {
    User user = 1;
}
```

**Usage:**
```go
req := &pb.CreateUserRequest{
    Username: "johndoe",
    FullName: "John Doe",
    Email:    "john@example.com",
    Password: "secretpassword",
}
resp, err := client.CreateUser(ctx, req)
```

#### UpdateUser

**Request:**
```protobuf
message UpdateUserRequest {
    string username = 1;
    optional string full_name = 2;
    optional string email = 3;
    optional string password = 4;
}
```

**Response:**
```protobuf
message UpdateUserResponse {
    User user = 1;
}
```

#### LoginUser

**Request:**
```protobuf
message LoginUserRequest {
    string username = 1;
    string password = 2;
}
```

**Response:**
```protobuf
message LoginUserResponse {
    User user = 1;
    string session_id = 2;
    string access_token = 3;
    string refresh_token = 4;
    google.protobuf.Timestamp access_token_expires_at = 5;
    google.protobuf.Timestamp refresh_token_expires_at = 6;
}
```

#### VerifyEmail

**Request:**
```protobuf
message VerifyEmailRequest {
    int64 email_id = 1;
    string secret_code = 2;
}
```

**Response:**
```protobuf
message VerifyEmailResponse {
    bool is_verified = 1;
}
```

## Database Layer

### Store Interface

```go
type Store interface {
    Querier
    TransferTx(ctx context.Context, arg TransferTxParams) (TransferTxResult, error)
    CreateUserTx(ctx context.Context, arg CreateUserTxParams) (CreateUserTxResult, error)
    VerifyEmailTx(ctx context.Context, arg VerifyEmailTxParams) (VerifyEmailTxResult, error)
}
```

### Querier Interface

```go
type Querier interface {
    AddAccountBalance(ctx context.Context, arg AddAccountBalanceParams) (Account, error)
    CreateAccount(ctx context.Context, arg CreateAccountParams) (Account, error)
    CreateEntry(ctx context.Context, arg CreateEntryParams) (Entry, error)
    CreateSession(ctx context.Context, arg CreateSessionParams) (Session, error)
    CreateTransfer(ctx context.Context, arg CreateTransferParams) (Transfer, error)
    CreateUser(ctx context.Context, arg CreateUserParams) (User, error)
    CreateVerifyEmail(ctx context.Context, arg CreateVerifyEmailParams) (VerifyEmail, error)
    DeleteAccount(ctx context.Context, id int64) error
    GetAccount(ctx context.Context, id int64) (Account, error)
    GetAccountForUpdate(ctx context.Context, id int64) (Account, error)
    GetEntry(ctx context.Context, id int64) (Entry, error)
    GetSession(ctx context.Context, id uuid.UUID) (Session, error)
    GetTransfer(ctx context.Context, id int64) (Transfer, error)
    GetUser(ctx context.Context, username string) (User, error)
    ListAccounts(ctx context.Context, arg ListAccountsParams) ([]Account, error)
    ListEntries(ctx context.Context, arg ListEntriesParams) ([]Entry, error)
    ListTransfers(ctx context.Context, arg ListTransfersParams) ([]Transfer, error)
    UpdateAccount(ctx context.Context, arg UpdateAccountParams) (Account, error)
    UpdateUser(ctx context.Context, arg UpdateUserParams) (User, error)
    UpdateVerifyEmail(ctx context.Context, arg UpdateVerifyEmailParams) (VerifyEmail, error)
}
```

### Data Models

#### User
```go
type User struct {
    Username          string    `json:"username"`
    HashedPassword    string    `json:"hashed_password"`
    FullName          string    `json:"full_name"`
    Email             string    `json:"email"`
    PasswordChangedAt time.Time `json:"password_changed_at"`
    CreatedAt         time.Time `json:"created_at"`
    IsEmailVerified   bool      `json:"is_email_verified"`
    Role              string    `json:"role"`
}
```

#### Account
```go
type Account struct {
    ID        int64     `json:"id"`
    Owner     string    `json:"owner"`
    Balance   int64     `json:"balance"`
    Currency  string    `json:"currency"`
    CreatedAt time.Time `json:"created_at"`
}
```

#### Transfer
```go
type Transfer struct {
    ID            int64 `json:"id"`
    FromAccountID int64 `json:"from_account_id"`
    ToAccountID   int64 `json:"to_account_id"`
    Amount        int64 `json:"amount"`
    CreatedAt     time.Time `json:"created_at"`
}
```

#### Entry
```go
type Entry struct {
    ID        int64 `json:"id"`
    AccountID int64 `json:"account_id"`
    Amount    int64 `json:"amount"`
    CreatedAt time.Time `json:"created_at"`
}
```

#### Session
```go
type Session struct {
    ID           uuid.UUID `json:"id"`
    Username     string    `json:"username"`
    RefreshToken string    `json:"refresh_token"`
    UserAgent    string    `json:"user_agent"`
    ClientIp     string    `json:"client_ip"`
    IsBlocked    bool      `json:"is_blocked"`
    ExpiresAt    time.Time `json:"expires_at"`
    CreatedAt    time.Time `json:"created_at"`
}
```

### Transaction Functions

#### TransferTx
```go
func (store *SQLStore) TransferTx(ctx context.Context, arg TransferTxParams) (TransferTxResult, error)
```

**Parameters:**
```go
type TransferTxParams struct {
    FromAccountID int64 `json:"from_account_id"`
    ToAccountID   int64 `json:"to_account_id"`
    Amount        int64 `json:"amount"`
}
```

**Result:**
```go
type TransferTxResult struct {
    Transfer    Transfer `json:"transfer"`
    FromAccount Account  `json:"from_account"`
    ToAccount   Account  `json:"to_account"`
    FromEntry   Entry    `json:"from_entry"`
    ToEntry     Entry    `json:"to_entry"`
}
```

#### CreateUserTx
```go
func (store *SQLStore) CreateUserTx(ctx context.Context, arg CreateUserTxParams) (CreateUserTxResult, error)
```

#### VerifyEmailTx
```go
func (store *SQLStore) VerifyEmailTx(ctx context.Context, arg VerifyEmailTxParams) (VerifyEmailTxResult, error)
```

## Utility Functions

### Password Utilities

#### HashPassword
```go
func HashPassword(password string) (string, error)
```

**Usage:**
```go
hashedPassword, err := util.HashPassword("mypassword")
if err != nil {
    return fmt.Errorf("failed to hash password: %w", err)
}
```

#### CheckPassword
```go
func CheckPassword(password string, hashedPassword string) error
```

**Usage:**
```go
err := util.CheckPassword("mypassword", hashedPassword)
if err != nil {
    return fmt.Errorf("invalid password: %w", err)
}
```

### Validation Functions

#### ValidateUsername
```go
func ValidateUsername(value string) error
```

**Validation Rules:**
- Length: 3-100 characters
- Only lowercase letters, digits, or underscore

#### ValidateFullName
```go
func ValidateFullName(value string) error
```

**Validation Rules:**
- Length: 3-100 characters
- Only letters or spaces

#### ValidatePassword
```go
func ValidatePassword(value string) error
```

**Validation Rules:**
- Length: 6-100 characters

#### ValidateEmail
```go
func ValidateEmail(value string) error
```

**Validation Rules:**
- Length: 3-200 characters
- Valid email format

#### ValidateEmailId
```go
func ValidateEmailId(value int64) error
```

**Validation Rules:**
- Must be a positive integer

#### ValidateSecretCode
```go
func ValidateSecretCode(value string) error
```

**Validation Rules:**
- Length: 32-128 characters

### String Validation Helper
```go
func ValidateString(value string, minLength int, maxLength int) error
```

## Worker System

### Task Distributor Interface

```go
type TaskDistributor interface {
    DistributeTaskSendVerifyEmail(
        ctx context.Context,
        payload *PayloadSendVerifyEmail,
        opts ...asynq.Option,
    ) error
}
```

### Task Processor Interface

```go
type TaskProcessor interface {
    Start() error
    Shutdown()
    ProcessTaskSendVerifyEmail(ctx context.Context, task *asynq.Task) error
}
```

### Redis Task Distributor

```go
func NewRedisTaskDistributor(redisOpt asynq.RedisClientOpt) TaskDistributor
```

**Usage:**
```go
redisOpt := asynq.RedisClientOpt{
    Addr: config.RedisAddress,
}
taskDistributor := worker.NewRedisTaskDistributor(redisOpt)
```

### Redis Task Processor

```go
func NewRedisTaskProcessor(redisOpt asynq.RedisClientOpt, store db.Store, mailer mail.EmailSender) TaskProcessor
```

**Usage:**
```go
mailer := mail.NewGmailSender(config.EmailSenderName, config.EmailSenderAddress, config.EmailSenderPassword)
taskProcessor := worker.NewRedisTaskProcessor(redisOpt, store, mailer)
```

### Task Queues

- **QueueCritical**: High priority tasks (weight: 10)
- **QueueDefault**: Normal priority tasks (weight: 5)

## Email System

### Email Sender Interface

```go
type EmailSender interface {
    SendEmail(
        subject string,
        content string,
        to []string,
        cc []string,
        bcc []string,
        attachFiles []string,
    ) error
}
```

### Gmail Sender

```go
func NewGmailSender(name string, fromEmailAddress string, fromEmailPassword string) EmailSender
```

**Usage:**
```go
mailer := mail.NewGmailSender(
    "Simple Bank",
    "noreply@simplebank.com",
    "app_password",
)
```

**Send Email:**
```go
err := mailer.SendEmail(
    "Welcome to Simple Bank",
    "<h1>Welcome!</h1><p>Thank you for joining Simple Bank.</p>",
    []string{"user@example.com"},
    []string{},
    []string{},
    []string{},
)
```

## Error Handling

### Database Errors

The application uses custom error handling for database operations:

```go
var (
    ErrRecordNotFound = errors.New("record not found")
)
```

### Error Code Functions

```go
func ErrorCode(err error) string
```

**Common Error Codes:**
- `23505`: Unique violation
- `23503`: Foreign key violation
- `23502`: Not null violation

### gRPC Error Handling

The gRPC layer provides structured error responses:

```go
func invalidArgumentError(violations []*errdetails.BadRequest_FieldViolation) error
func unauthenticatedError(err error) error
func fieldViolation(field string, err error) *errdetails.BadRequest_FieldViolation
```

## Examples

### Complete HTTP Server Setup

```go
package main

import (
    "log"
    "github.com/techschool/simplebank/api"
    "github.com/techschool/simplebank/util"
    "github.com/techschool/simplebank/db/sqlc"
    "github.com/jackc/pgx/v5/pgxpool"
)

func main() {
    // Load configuration
    config, err := util.LoadConfig(".")
    if err != nil {
        log.Fatal("cannot load config:", err)
    }

    // Connect to database
    connPool, err := pgxpool.New(context.Background(), config.DBSource)
    if err != nil {
        log.Fatal("cannot connect to db:", err)
    }

    // Create store
    store := db.NewStore(connPool)

    // Create server
    server, err := api.NewServer(config, store)
    if err != nil {
        log.Fatal("cannot create server:", err)
    }

    // Start server
    err = server.Start(config.HTTPServerAddress)
    if err != nil {
        log.Fatal("cannot start server:", err)
    }
}
```

### Complete gRPC Server Setup

```go
package main

import (
    "log"
    "github.com/techschool/simplebank/gapi"
    "github.com/techschool/simplebank/util"
    "github.com/techschool/simplebank/db/sqlc"
    "github.com/techschool/simplebank/worker"
    "github.com/jackc/pgx/v5/pgxpool"
    "github.com/hibiken/asynq"
    "google.golang.org/grpc"
)

func main() {
    // Load configuration
    config, err := util.LoadConfig(".")
    if err != nil {
        log.Fatal("cannot load config:", err)
    }

    // Connect to database
    connPool, err := pgxpool.New(context.Background(), config.DBSource)
    if err != nil {
        log.Fatal("cannot connect to db:", err)
    }

    // Create store
    store := db.NewStore(connPool)

    // Setup Redis task distributor
    redisOpt := asynq.RedisClientOpt{
        Addr: config.RedisAddress,
    }
    taskDistributor := worker.NewRedisTaskDistributor(redisOpt)

    // Create gRPC server
    server, err := gapi.NewServer(config, store, taskDistributor)
    if err != nil {
        log.Fatal("cannot create server:", err)
    }

    // Create gRPC server
    grpcServer := grpc.NewServer()
    pb.RegisterSimpleBankServer(grpcServer, server)

    // Start server
    listener, err := net.Listen("tcp", config.GRPCServerAddress)
    if err != nil {
        log.Fatal("cannot create listener:", err)
    }

    err = grpcServer.Serve(listener)
    if err != nil {
        log.Fatal("cannot start gRPC server:", err)
    }
}
```

### Creating a User with Email Verification

```go
// Create user with email verification
arg := db.CreateUserTxParams{
    CreateUserParams: db.CreateUserParams{
        Username:       "johndoe",
        HashedPassword: hashedPassword,
        FullName:       "John Doe",
        Email:          "john@example.com",
    },
    AfterCreate: func(user db.User) error {
        // Send verification email
        taskPayload := &worker.PayloadSendVerifyEmail{
            Username: user.Username,
        }
        opts := []asynq.Option{
            asynq.MaxRetry(10),
            asynq.ProcessIn(10 * time.Second),
            asynq.Queue(worker.QueueCritical),
        }
        return taskDistributor.DistributeTaskSendVerifyEmail(ctx, taskPayload, opts...)
    },
}

txResult, err := store.CreateUserTx(ctx, arg)
if err != nil {
    return fmt.Errorf("failed to create user: %w", err)
}
```

### Making a Transfer

```go
// Create transfer
arg := db.TransferTxParams{
    FromAccountID: 1,
    ToAccountID:   2,
    Amount:        100,
}

result, err := store.TransferTx(ctx, arg)
if err != nil {
    return fmt.Errorf("failed to transfer money: %w", err)
}

fmt.Printf("Transfer completed: %+v\n", result)
```

### Token Management

```go
// Create token maker
tokenMaker, err := token.NewPasetoMaker(config.TokenSymmetricKey)
if err != nil {
    return fmt.Errorf("cannot create token maker: %w", err)
}

// Create access token
accessToken, accessPayload, err := tokenMaker.CreateToken(
    "johndoe",
    "depositor",
    config.AccessTokenDuration,
)
if err != nil {
    return fmt.Errorf("cannot create access token: %w", err)
}

// Verify token
payload, err := tokenMaker.VerifyToken(accessToken)
if err != nil {
    return fmt.Errorf("invalid token: %w", err)
}

fmt.Printf("Token payload: %+v\n", payload)
```

## Security Considerations

1. **Password Hashing**: Uses bcrypt with default cost for password hashing
2. **Token Security**: Supports both JWT and PASETO tokens
3. **Input Validation**: Comprehensive validation for all inputs
4. **SQL Injection Prevention**: Uses SQLC for type-safe SQL queries
5. **Authentication**: JWT/PASETO-based authentication with refresh tokens
6. **Authorization**: Role-based access control (RBAC)
7. **HTTPS**: Supports HTTPS with automatic TLS certificate management
8. **Rate Limiting**: Can be implemented at the gateway level

## Performance Considerations

1. **Database Connection Pooling**: Uses pgxpool for efficient database connections
2. **Asynchronous Processing**: Redis-based task queue for background operations
3. **Caching**: Redis can be used for caching frequently accessed data
4. **Database Indexing**: Proper indexing on frequently queried columns
5. **Connection Limits**: Configurable connection limits for database and Redis

## Monitoring and Logging

1. **Structured Logging**: Uses zerolog for structured logging
2. **Request Tracing**: Built-in request/response logging
3. **Error Tracking**: Comprehensive error handling and logging
4. **Metrics**: Can be integrated with Prometheus for metrics collection
5. **Health Checks**: Built-in health check endpoints

## Deployment

The application supports deployment to:

1. **Docker**: Multi-stage Dockerfile for minimal image size
2. **Kubernetes**: Complete Kubernetes manifests for EKS deployment
3. **AWS**: Integration with AWS services (RDS, ElastiCache, EKS)
4. **CI/CD**: GitHub Actions for automated testing and deployment

## API Versioning

- **HTTP API**: Version 1 (v1)
- **gRPC API**: Version 1.2
- **Swagger Documentation**: Available at `/swagger/`

## Rate Limits

- **Account Creation**: No specific limits (handled by database constraints)
- **Login Attempts**: No specific limits (can be implemented)
- **Transfer Operations**: No specific limits (handled by business logic)
- **API Calls**: No specific limits (can be implemented at gateway level)

## Error Codes

### HTTP Status Codes
- `200`: Success
- `400`: Bad Request (validation errors)
- `401`: Unauthorized (authentication required)
- `403`: Forbidden (authorization failed)
- `404`: Not Found
- `500`: Internal Server Error

### gRPC Status Codes
- `OK`: Success
- `INVALID_ARGUMENT`: Invalid request parameters
- `UNAUTHENTICATED`: Authentication required
- `PERMISSION_DENIED`: Authorization failed
- `NOT_FOUND`: Resource not found
- `ALREADY_EXISTS`: Resource already exists
- `INTERNAL`: Internal server error

This documentation provides a comprehensive overview of all public APIs, functions, and components in the Simple Bank application. For more specific implementation details, refer to the source code and test files.