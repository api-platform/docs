# MCP: Exposing Your API to AI Agents

API Platform integrates with the [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) to
expose your API as tools and resources that AI agents (LLMs) can discover and interact with.

MCP defines a standard way for AI models to discover available tools, understand their input
schemas, and invoke them. API Platform leverages its existing metadata system — state processors,
validation, serialization — to turn your PHP classes into MCP-compliant tool definitions.

## Installation

Install the [MCP Bundle](https://github.com/symfony-tools/mcp-bundle):

```console
composer require symfony/mcp-bundle
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

## Validation

MCP tool inputs support validation using the same mechanisms as regular API Platform operations.

On Symfony, use [Symfony Validator constraints](../symfony/validation.md):

```php
<?php
namespace App\ApiResource;

use ApiPlatform\Metadata\McpTool;
use Symfony\Component\Validator\Constraints as Assert;

#[McpTool(
    name: 'submit_contact',
    description: 'Submit a contact form',
    processor: [self::class, 'process']
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

| Option              | Description                                                               |
| ------------------- | ------------------------------------------------------------------------- |
| `name`              | Tool name exposed to AI agents (defaults to the class short name)         |
| `description`       | Human-readable description of the tool (defaults to class DocBlock)       |
| `structuredContent` | Whether to include JSON structured content in responses (default: `true`) |
| `input`             | A separate DTO class to use as the tool's input schema                    |
| `output`            | A separate DTO class to use as the tool's output representation           |
| `annotations`       | MCP tool annotations describing behavior hints                            |
| `icons`             | List of icon URLs representing the tool                                   |
| `meta`              | Arbitrary metadata                                                        |
| `rules`             | Laravel validation rules (Laravel only)                                   |

## McpResource Options

The `McpResource` attribute accepts all standard [operation options](operations.md) plus:

| Option              | Description                                                                |
| ------------------- | -------------------------------------------------------------------------- |
| `uri`               | Unique URI identifying this resource (required, uses `resource://` scheme) |
| `name`              | Human-readable name for the resource                                       |
| `description`       | Description of the resource (defaults to class DocBlock)                   |
| `structuredContent` | Whether to include JSON structured content (default: `true`)               |
| `mimeType`          | MIME type of the resource content                                          |
| `size`              | Size in bytes, if known                                                    |
| `annotations`       | MCP resource annotations                                                   |
| `icons`             | List of icon URLs                                                          |
| `meta`              | Arbitrary metadata                                                         |
