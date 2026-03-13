# MCP: Exposing Your API to AI Agents

API Platform integrates with the [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) to
expose your API as tools and resources that AI agents (LLMs) can discover and interact with.

MCP defines a standard way for AI models to discover available tools, understand their input
schemas, and invoke them. API Platform leverages its existing metadata system — state processors,
validation, serialization — to turn your PHP classes into MCP-compliant tool definitions.

> **Note:** The MCP integration is marked `@experimental`. The API may change between minor
> releases.

## Installation

### Symfony

Install the required packages:

```console
composer require api-platform/mcp symfony/mcp-bundle
```

### Laravel

MCP support is optional in Laravel. Install the required package:

```console
composer require api-platform/mcp
```

## Configuring the MCP Server

### Symfony

Enable the MCP server and configure the transport in your Symfony configuration:

```yaml
# config/packages/mcp.yaml
mcp:
    client_transports:
        http: true
        stdio: false
    http:
        path: "/mcp"
        session:
            store: "file"
            directory: "%kernel.cache_dir%/mcp"
            ttl: 3600
```

You can also configure API Platform's MCP integration in `api_platform.yaml`:

```yaml
# config/packages/api_platform.yaml
api_platform:
    mcp:
        enabled: true # default: true
        format: jsonld # default: 'jsonld'
```

The `format` option sets the serialization format used for MCP tool structured content. It must be a
format registered in `api_platform.formats` (e.g. `jsonld`, `json`, `jsonapi`). The default `jsonld`
produces rich semantic output with `@context`, `@id`, and `@type` fields.

### Laravel

MCP is enabled by default in the Laravel configuration:

```php
// config/api-platform.php
return [
    // ...
    'mcp' => [
        'enabled' => true,
    ],
];
```

The MCP endpoint is automatically registered at `/mcp`.

## Architecture

API Platform registers its own `Handler` with the MCP SDK. The handler's `supports()` method returns
`true` only for tools and resources that are registered through API Platform metadata. If a
requested tool or resource is not found in the API Platform registry, the handler returns `false`
and the MCP SDK proceeds through its own handler chain.

This means you can register both API Platform MCP tools and native `mcp/sdk` tools on the same
server — they coexist without conflict.

## Declaring MCP Tools

MCP tools let AI agents invoke operations on your API. The primary pattern uses `#[McpTool]` as a
class attribute: the class properties define the tool's input schema, and a
[state processor](state-processors.md) handles the command.

This follows a CQRS-style approach: tools receive input from AI agents and process it through your
application logic.

### Simple Tool

```php
<?php
// api/src/ApiResource/ProcessMessage.php with Symfony or app/ApiResource/ProcessMessage.php with Laravel
namespace App\ApiResource;

use ApiPlatform\Metadata\McpTool;

#[McpTool(
    name: 'process_message',
    description: 'Process a message with priority',
    processor: [self::class, 'process']
)]
class ProcessMessage
{
    public function __construct(
        private string $message,
        private int $priority = 1,
    ) {}

    public function getMessage(): string
    {
        return $this->message;
    }

    public function setMessage(string $message): void
    {
        $this->message = $message;
    }

    public function getPriority(): int
    {
        return $this->priority;
    }

    public function setPriority(int $priority): void
    {
        $this->priority = $priority;
    }

    public static function process($data): mixed
    {
        $data->setMessage('Processed: ' . $data->getMessage());
        $data->setPriority($data->getPriority() + 10);

        return $data;
    }
}
```

The class properties (`$message`, `$priority`) become the tool's `inputSchema`. When an AI agent
calls this tool, API Platform deserializes the input into a `ProcessMessage` instance and passes it
to the processor. The returned object is serialized back as structured content.

You can also use a [dedicated state processor service](state-processors.md) instead of a static
method — any callable or service class implementing `ProcessorInterface` works.

### Using a Separate Input DTO

When the tool's input schema should differ from the class itself, use the `input` option to specify
a separate DTO:

```php
<?php
namespace App\Dto;

class SearchQuery
{
    public string $search;
}
```

```php
<?php
namespace App\ApiResource;

use ApiPlatform\Metadata\McpTool;
use App\Dto\SearchQuery;
use App\State\SearchBooksProcessor;

#[McpTool(
    name: 'search_books',
    description: 'Search books by keyword',
    input: SearchQuery::class,
    processor: SearchBooksProcessor::class,
)]
class BookSearchResult
{
    public int $id;
    public string $title;
    public string $isbn;
}
```

Here, `SearchQuery` defines the tool's `inputSchema` (what the AI agent sends), while
`BookSearchResult` defines the output structure. The processor receives a `SearchQuery` instance and
returns the result:

```php
<?php
namespace App\State;

use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProcessorInterface;
use App\Entity\Book;
use Doctrine\Persistence\ManagerRegistry;

class SearchBooksProcessor implements ProcessorInterface
{
    public function __construct(private readonly ManagerRegistry $managerRegistry) {}

    public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []): ?iterable
    {
        return $this->managerRegistry->getRepository(Book::class)->findAll();
    }
}
```

### Returning a Collection with McpToolCollection

Use `McpToolCollection` instead of `McpTool` when a tool returns a collection of items. It extends
`McpTool` and implements `CollectionOperationInterface`, which signals to the serializer and schema
factory that the output is a list.

```php
<?php
namespace App\ApiResource;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\McpToolCollection;
use App\Dto\SearchQuery;
use App\State\SearchBooksProcessor;

#[ApiResource(
    operations: [],
    mcp: [
        'list_books' => new McpToolCollection(
            description: 'List Books',
            input: SearchQuery::class,
            processor: SearchBooksProcessor::class,
            structuredContent: true,
        ),
    ]
)]
class Book
{
    public ?int $id = null;
    public ?string $title = null;
    public ?string $isbn = null;
}
```

When `structuredContent: true`, the structured content response includes `@context`,
`hydra:totalItems`, and a `hydra:member` array containing the serialized items. The output schema
published to MCP clients reflects this collection structure.

### Returning Custom Results

By default, tool results are serialized using API Platform's [serialization](serialization.md)
system with structured content (JSON). If you need full control over the response, return a
`CallToolResult` directly from your processor and set `structuredContent: false`:

```php
<?php
namespace App\ApiResource;

use ApiPlatform\Metadata\McpTool;
use Mcp\Schema\Content\TextContent;
use Mcp\Schema\Result\CallToolResult;

#[McpTool(
    name: 'generate_report',
    description: 'Generate a markdown report',
    processor: [self::class, 'process'],
    structuredContent: false
)]
class Report
{
    public function __construct(
        private string $title,
        private string $content,
    ) {}

    // getters and setters...

    public static function process($data): CallToolResult
    {
        $markdown = "# {$data->getTitle()}\n\n{$data->getContent()}";

        return new CallToolResult(
            [new TextContent($markdown)],
            false
        );
    }
}
```

Setting `structuredContent: false` disables the automatic JSON serialization. When returning a
`CallToolResult`, the response is sent as-is to the AI agent.

## Using McpTool with ApiResource

The standalone `#[McpTool]` class attribute is convenient for simple tools, but you can also declare
MCP tools inside an `#[ApiResource]` attribute using the `mcp` parameter. This is the appropriate
pattern when:

- The class should not expose any HTTP endpoints (`operations: []`)
- You need to combine multiple MCP operations on a single class
- You need fine-grained control that is cleaner to express at the resource level

The `mcp` parameter accepts an associative array of `McpTool` or `McpResource` instances, keyed by
the operation name. The array key is the tool name — the `name` parameter inside `new McpTool(...)`
is redundant when using this pattern and should be omitted. Setting `operations: []` means no HTTP
routes are registered for the class — it exists solely as an MCP tool definition.

### Simple Tool with a Dedicated Processor

```php
<?php
// api/src/ApiResource/ReadHydraResource.php with Symfony or app/ApiResource/ReadHydraResource.php with Laravel
namespace App\ApiResource;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\McpTool;
use App\State\ReadHydraResourceProcessor;

#[ApiResource(
    operations: [],
    mcp: [
        'read_hydra_resource' => new McpTool(
            description: 'Navigate to a Hydra API resource by URI.',
            processor: ReadHydraResourceProcessor::class,
            structuredContent: false,
        ),
    ],
)]
class ReadHydraResource
{
    public string $uri;
}
```

The class properties define the tool's `inputSchema`. The processor receives a `ReadHydraResource`
instance and returns the result. Because `structuredContent: false` is set, the processor can return
a `CallToolResult` directly, bypassing automatic JSON serialization.

### Customizing Property Schemas with ApiProperty

Some LLM providers reject JSON Schema union types such as `["array", "null"]` that PHP nullable
types produce by default. Use `#[ApiProperty(schema: [...])]` to override the generated schema for a
specific property:

```php
<?php
// api/src/ApiResource/InvokeHydraOperation.php with Symfony or app/ApiResource/InvokeHydraOperation.php with Laravel
namespace App\ApiResource;

use ApiPlatform\Metadata\ApiProperty;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\McpTool;
use App\State\InvokeHydraOperationProcessor;

#[ApiResource(
    operations: [],
    mcp: [
        'invoke_hydra_operation' => new McpTool(
            description: 'Execute a state-changing operation on a Hydra API resource.',
            processor: InvokeHydraOperationProcessor::class,
            structuredContent: false,
        ),
    ],
)]
class InvokeHydraOperation
{
    public string $uri;
    public string $method;
    #[ApiProperty(schema: ['type' => 'object', 'description' => 'JSON payload for the request'])]
    public ?array $payload = null;
}
```

Without the `#[ApiProperty]` override, `?array $payload` would generate
`{"type": ["array", "null"]}`. The explicit schema replaces it with `{"type": "object", ...}`, which
all major LLM providers accept.

## Validation

The MCP SDK already validates tool inputs against the JSON Schema at the transport level (types,
required fields, etc.). API Platform's own validation pipeline is disabled by default for MCP tools.
Set `validate: true` to enable business-level validation — constraints like email format, string
length, or custom rules that go beyond structural schema checks.

On Symfony, use [Symfony Validator constraints](../symfony/validation.md):

```php
<?php
namespace App\ApiResource;

use ApiPlatform\Metadata\McpTool;
use Symfony\Component\Validator\Constraints as Assert;

#[McpTool(
    name: 'submit_contact',
    description: 'Submit a contact form',
    processor: [self::class, 'process'],
    validate: true  // Must be explicitly enabled for MCP tools
)]
class ContactForm
{
    #[Assert\NotBlank]
    #[Assert\Length(min: 3, max: 50)]
    private ?string $name = null;

    #[Assert\NotNull]
    #[Assert\Email]
    private ?string $email = null;

    #[Assert\Positive]
    private ?int $age = null;

    // getters, setters, and processor...
}
```

On Laravel, use [validation rules](../laravel/validation.md):

```php
#[McpTool(
    name: 'submit_contact',
    description: 'Submit a contact form',
    processor: [self::class, 'process'],
    validate: true,  // Must be explicitly enabled for MCP tools
    rules: [
        'name' => 'required|min:3|max:50',
        'email' => 'required|email',
    ]
)]
```

## Declaring MCP Resources

MCP resources expose read-only content that AI agents can retrieve — documentation, configuration,
reference data, etc. Use the `McpResource` attribute with a [state provider](state-providers.md):

```php
<?php
namespace App\ApiResource;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\McpResource;

#[ApiResource(
    operations: [],
    mcp: [
        'api_docs' => new McpResource(
            uri: 'resource://my-app/documentation',
            name: 'App-Documentation',
            description: 'Application API documentation',
            mimeType: 'text/markdown',
            provider: [self::class, 'provide']
        ),
    ]
)]
class Documentation
{
    public function __construct(
        private string $content,
        private string $uri,
    ) {}

    // getters and setters...

    public static function provide(): self
    {
        return new self(
            content: '# My API Documentation\n\nWelcome to the API.',
            uri: 'resource://my-app/documentation'
        );
    }
}
```

The `uri` must be unique across the MCP server and follows the `resource://` URI scheme.

## McpTool Options

The `McpTool` attribute accepts all standard [operation options](operations.md) plus:

| Option               | Description                                                               |
| -------------------- | ------------------------------------------------------------------------- |
| `name`               | Tool name exposed to AI agents (defaults to the class short name)         |
| `description`        | Human-readable description of the tool (defaults to class DocBlock)       |
| `structuredContent`  | Whether to include JSON structured content in responses (default: `true`) |
| `input`              | A separate DTO class to use as the tool's input schema                    |
| `output`             | A separate DTO class to use as the tool's output representation           |
| `inputFormats`       | Serialization formats for deserializing the tool input (e.g. `['json']`)  |
| `outputFormats`      | Serialization formats for structured content (e.g. `['jsonld']`)          |
| `contentNegotiation` | Whether to enable HTTP content negotiation (default: `false` for MCP)     |
| `validate`           | Whether to run the validation pipeline (default: `false` for MCP)         |
| `annotations`        | MCP tool annotations describing behavior hints                            |
| `icons`              | List of icon URLs representing the tool                                   |
| `meta`               | Arbitrary metadata                                                        |
| `rules`              | Laravel validation rules (Laravel only)                                   |

## McpResource Options

The `McpResource` attribute accepts all standard [operation options](operations.md) plus:

| Option               | Description                                                                |
| -------------------- | -------------------------------------------------------------------------- |
| `uri`                | Unique URI identifying this resource (required, uses `resource://` scheme) |
| `name`               | Human-readable name for the resource                                       |
| `description`        | Description of the resource (defaults to class DocBlock)                   |
| `structuredContent`  | Whether to include JSON structured content (default: `true`)               |
| `contentNegotiation` | Whether to enable HTTP content negotiation (default: `false` for MCP)      |
| `mimeType`           | MIME type of the resource content                                          |
| `size`               | Size in bytes, if known                                                    |
| `annotations`        | MCP resource annotations                                                   |
| `icons`              | List of icon URLs                                                          |
| `meta`               | Arbitrary metadata                                                         |
