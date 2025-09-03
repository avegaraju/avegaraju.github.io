# How I Started Using Multi-Agent AI for Better Code Development

After working with various AI tools for coding, I found myself stuck in the same pattern. I would ask ChatGPT or Claude to help with a specific problem, get some code, and then move to the next issue. But when I tried to build complete features, especially complex ones, the AI would give me huge code blocks that were hard to test and debug.

Then I discovered something that changed how I approach development completely. Instead of using one AI for everything, I started creating specialized AI agents that work like a real development team. I've been applying this approach to my personal project which is a location-based web API built with ASP.NET Core and Hexagonal architecture.

## The Problem with Single AI Approach

Let me give you a real example from my current project. I was working on adding authentication using Google and Facebook OAuth providers. When I asked one AI to help with the authentication controller, it gave me a massive controller with login, registration, JWT token generation, user creation, and error handling all in one go.

The code looked impressive but had several issues:

- No proper error handling for different OAuth scenarios
- Tests were missing for edge cases like invalid tokens
- Architecture decisions weren't well thought out
- Too much code to debug if something went wrong

This is when I realized that asking one AI to be architect, tester, and developer at the same time doesn't work well for complex features.

## Creating Specialized AI Agents

I decided to create three specialized agents, each with their own expertise:

**The Architect Agent**: This agent focuses only on system design. When I ask about authentication, it thinks about JWT token structure, user repository patterns, OAuth flow security, and how different layers in my hexagonal architecture should interact.

**The Tester Agent**: This agent is obsessed with finding problems. It creates test cases for happy paths, edge cases, error scenarios, and security vulnerabilities. It thinks about what could go wrong with OAuth tokens and how to test for it.

**The Coder Agent**: This agent takes the architecture decisions and test requirements and implements clean, focused code. It follows TDD principles and makes small, testable changes.

## Real Example: Adding User Creation Logic

Let me show you how these agents worked together on a recent feature. I needed to add user creation logic that would either create a new user or return an existing one based on email.

### Step 1: Architecture Agent

**Me:** _architect_ "I need to implement user creation logic for my authentication flow. Users can sign up via Google or Facebook OAuth. How should I structure this?"

**Architect Agent Response:**
"For user creation with OAuth providers, I would recommend this approach:

1. Create a UseCase class following your hexagonal architecture pattern
2. The UseCase should check if user exists first, then create if needed
3. Use the existing IUsersRepository interface for data operations
4. Accept a UserUseCaseRequest with email, first name, last name, picture URL, and login provider
5. Return just the user ID string to keep it simple

Key components:

- ICreateUserUseCase interface in Core layer
- CreateUserUseCase implementation in Core layer
- UserUseCaseRequest class for the input data
- IUsersRepository handles the database operations

This keeps the business logic separate from OAuth token validation and follows your existing hexagonal architecture pattern."

### Step 2: Tester Agent

**Me:** _tester_ "Based on this architecture, create comprehensive test cases for the user creation use case."

**Tester Agent Response:**
"Here are the test scenarios for user creation:

**Happy Path Tests:**

1. New user creation should return new user ID
2. Existing user lookup should return existing user ID

**Edge Cases:**

1. User exists with same email but different provider
2. Empty or null email should throw ArgumentException
3. Database connection failures should bubble up properly

**Integration Tests:**

1. Full OAuth flow with user creation
2. Multiple rapid requests for same email (race conditions)

**Specific Test Cases:**

```csharp
[Fact]
public async Task ExecuteAsync_ShouldReturnExistingUserId_WhenUserAlreadyExists()

[Fact]
public async Task ExecuteAsync_ShouldCreateUserAndReturnUserId_WhenUserDoesNotExist()

[Fact]
public async Task ExecuteAsync_ShouldThrowException_WhenEmailIsNull()
```

Each test should use AutoFixture for test data generation and Moq for repository mocking."

### Step 3: Coder Agent (First Iteration)

**Me:** _coder_ "Implement the user creation use case, but start small and testable."

**Coder Agent Response:**
"I'll start with the minimal implementation:

```csharp
public interface ICreateUserUseCase
{
    Task<string> ExecuteAsync(UserUseCaseRequest request);
}

public class CreateUserUseCase : ICreateUserUseCase
{
    private readonly IUsersRepository _usersRepository;

    public CreateUserUseCase(IUsersRepository usersRepository)
    {
        _usersRepository = usersRepository;
    }

    public async Task<string> ExecuteAsync(UserUseCaseRequest request)
    {
        // Check if user already exists
        var existingUserId = await _usersRepository.GetUserByEmailAsync(request.Email);
        if (existingUserId != null)
        {
            return existingUserId;
        }

        var userId = await _usersRepository.CreateUserAsync(request);
        return userId;
    }
}
```

And the corresponding test:

```csharp
[Fact]
public async Task ExecuteAsync_ShouldReturnExistingUserId_WhenUserAlreadyExists()
{
    // Arrange
    var fixture = new Fixture();
    var request = fixture.Create<UserUseCaseRequest>();
    var existingUserId = fixture.Create<string>();
    var repoMock = new Mock<IUsersRepository>();
    repoMock.Setup(r => r.GetUserByEmailAsync(request.Email)).ReturnsAsync(existingUserId);

    // Act & Assert
    var result = await useCase.ExecuteAsync(request);
    result.Should().Be(existingUserId);
}
```

Next iteration: Add input validation and error handling."

### Step 4: Testing with HTTP Client

After implementing the use case, I could test it through my AuthController using HTTP client:

```http
### Test Google OAuth signup
POST http://localhost:5282/api/auth/google-signup
Content-Type: application/json

{
  "idToken": "fake-google-token-for-testing",
  "email": "test@example.com",
  "firstName": "Test",
  "lastName": "User",
  "picture": "https://example.com/picture.jpg"
}

### Test Facebook OAuth signup
POST http://localhost:5282/api/auth/facebook-signup
Content-Type: application/json

{
  "accessToken": "fake-facebook-token",
  "email": "test@example.com"
}
```

## Why This Approach Works Better

**Better Architecture**: The architect agent thinks about long-term implications that I might miss when coding quickly. It suggested using the existing hexagonal architecture pattern instead of putting everything in the controller.

**Comprehensive Testing**: The tester agent finds edge cases I wouldn't think of. It caught issues like race conditions with multiple rapid requests for the same email.

**Incremental Development**: The coder agent makes small changes that are easy to test and debug. No more massive code dumps.

**Learning**: Each agent's approach teaches me something. I've become better at system design by seeing how the architect agent thinks through problems.

## Setting Up Your Own Agents

You can set up similar agents using any AI tool that supports custom instructions or configurations. The key is giving each agent a specific role and making them stick to it.

For each agent, I created specific instructions:

- Architect: Focus only on design decisions, don't write implementation code
- Tester: Create specific test cases, think about edge cases and error scenarios
- Coder: Implement minimal code to pass tests, prefer small incremental changes

## Another Example: Location Search Feature

Here's how the agents helped with adding location search functionality to my API:

**Architect**: Designed the use case pattern with ITomTomMapsClient port and SearchLocationUseCase implementation, following hexagonal architecture.

**Tester**: Created tests for invalid queries, API failures, and response mapping scenarios.

**Coder**: Implemented in small steps - first the basic use case, then input validation, then proper error handling.

I could test each iteration using Rider:

```http
### Test location search
GET http://localhost:5282/api/maps/search?query=Amsterdam
Accept: application/json

### Test empty query (should return 400)
GET http://localhost:5282/api/maps/search?query=
Accept: application/json
```

## The Results

Since adopting this approach on my project:

- My code quality improved because every feature goes through architectural review
- I catch bugs earlier because of comprehensive test scenarios
- Development feels less overwhelming because I work in small, manageable pieces
- I ship features faster because there's less debugging of large code changes

The best part is that each agent's output becomes documentation for the feature. The architect's decisions explain why the system works a certain way, the tester's scenarios become regression tests, and the coder's incremental approach makes the git history very clear.

## Getting Started

Here's how to set this up on your own projects:

### 1. Install Claude Code and Initialize

```bash
# Install from Anthropic's website, then:
claude init
```

### 2. Update default Claude markdown file

**`.claude.md`**:

```markdown
## Other project specific guidance below

<Removed for brevity>

## Development Commands

### Build and Test

# Build the entire solution

dotnet build

# Run all tests

dotnet test

# Run specific test project

dotnet test Riderverse.Core.Tests

# Run the API

dotnet run --project Riderverse.Api

### Project Structure

- Controllers handle HTTP requests and delegate to use cases
- Use cases contain business logic and orchestrate repository operations
- Repositories are abstracted behind interfaces in Core.Ports
- External API clients (TomTom) are wrapped in adapter projects

## Configuration

The API requires these configuration values:

- `Jwt:Secret`, `Jwt:Issuer`, `Jwt:Audience` for JWT authentication
- `Maps:ApiKey` for TomTom Maps API
- `CosmosDb:ConnectionString`, `CosmosDb:DatabaseId`, `CosmosDb:ContainerId` for Cosmos DB

## Testing Framework

- Uses xUnit for test framework
- AutoFixture for test data generation
- Moq for mocking dependencies
- FluentAssertions for readable assertions

## Special Agents

When the user prefixes their request with one of these agent names, follow the corresponding instructions:

- **architect**: Refer to `.claude-artchitect.md` - Act as a software architecture specialist focused on system design, scalability, and architectural decisions
- **tester**: Refer to `.claude-tester.md` - Act as a QA engineer generating comprehensive test cases with detailed scenarios, prerequisites, steps, and edge cases
- **coder**: Refer to `.claude-coder.md` - Act as a senior developer implementing features and tests with clean, maintainable code following best practices

## Multi-Agent Collaboration

- Reference outputs from other agents when available
- Ask for clarification if architectural decisions affect your domain
- Provide outputs in formats that other agents can easily consume
```

### 2. Create Agent Configuration Files

**`.claude-architect.md`**:

```markdown
# Architecture Agent

You are a software architect focused on system design and architectural decisions.

## Your Role

- Analyze requirements and propose architectural solutions
- Design system components and their interactions
- Consider scalability, maintainability, and performance
- Focus on separation of concerns and clean architecture patterns

## Guidelines

- Provide detailed architectural explanations with pros/cons
- Suggest design patterns and architectural approaches
- Think about long-term maintainability
- Don't write implementation code - focus on design decisions

When asked about features, think through the system design first.
```

**`.claude-tester.md`**:

```markdown
# Test Agent

You are a QA specialist focused on comprehensive testing strategies.

## Your Role

- Generate test cases covering happy paths, edge cases, and error scenarios
- Design both unit and integration testing approaches
- Think about what could go wrong and how to test for it
- Consider security, performance, and reliability testing

## Guidelines

- Create specific, actionable test cases
- Include test data setup and expected outcomes
- Think about concurrent access and race conditions
- Cover both positive and negative test scenarios
- Use testing frameworks like xUnit, Moq, FluentAssertions

Focus on thorough test coverage that catches issues early.
```

**`.claude-coder.md`**:

```markdown
# Coding Agent

You are a senior developer focused on clean, testable implementations.

## Your Role

- Implement features following TDD principles
- Write small, incremental changes that are easy to test
- Follow established coding patterns and conventions
- Focus on clean, maintainable code

## Guidelines

- Make minimal implementations that pass tests first
- Refactor only after tests are green
- Write self-documenting code with clear naming
- Handle errors appropriately
- Keep changes small and focused

Implement one small piece at a time, ensuring it works before moving on.
```

### 4. Start Using Your AI Development Team

Now you can invoke your specialized agents from any terminal:

```bash
# Get architectural guidance
architect "How should I design user authentication with JWT tokens?"

# Generate comprehensive test cases
tester "Create test cases for user login including edge cases and security scenarios"

# Implement the feature incrementally
coder "Implement the user login endpoint following TDD principles"
```

The command-line approach makes it feel like you have a real development team working with you. Instead of trying to think about architecture, testing, and implementation all at once, you can focus on one aspect at a time while ensuring nothing gets missed.

This approach has made my side project development more systematic and way less stressful. Even with limited time on weekends, I can build quality features incrementally.

Have you tried anything similar? I'd be interested to know how others are structuring their AI development workflows.
