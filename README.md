# Symfony AI - Demo Application using amazee.ai

Symfony application demoing [Symfony AI](https://symfony.com/doc/current/ai/index.html) components with the
[amazee.ai](https://amazee.ai/) Private AI Gateway: LLMs and a managed pgvector database, in the region you choose.

This is a fork of [symfony/ai-demo](https://github.com/symfony/ai-demo) where every chat use case runs through
amazee.ai.

## Quick start

```shell
git clone https://github.com/amazeeio-demos/symfony-ai-demo.git
cd symfony-ai-demo
composer install

# Sends a code to your email, then writes the LLM and vector database credentials to .env.local
php bin/console ai:amazee:configure you@example.com

# Index the Symfony blog into the amazee.ai vector database
php bin/console ai:store:setup ai.store.postgres.symfony_blog
php bin/console ai:store:index blog -vv

symfony serve -d
```

Open https://localhost:8000/ and start chatting.

> [!NOTE]
> Without the [Symfony CLI](https://symfony.com/download), use `php -S 127.0.0.1:8000 -t public`
> (set `PHP_CLI_SERVER_WORKERS=4` so streamed answers don't block other requests) and open http://127.0.0.1:8000/.

## Examples

![demo.png](demo.png)

| Use case     | Provider                                         | Model                         |
|--------------|--------------------------------------------------|-------------------------------|
| YouTube      | amazee.ai                                        | `chat`                        |
| Recipe       | amazee.ai                                        | `chat_with_complex_json`      |
| Movies       | amazee.ai                                        | `chat_with_complex_json`      |
| Wikipedia    | amazee.ai                                        | `chat`                        |
| MCP          | amazee.ai                                        | `chat`                        |
| Symfony Blog | amazee.ai (LLM + vector database)                | `chat` · `embeddings`         |
| Video        | amazee.ai                                        | `chat_with_image_vision`      |
| Turbo Stream | amazee.ai                                        | `chat`                        |
| Speech       | amazee.ai + OpenAI for speech-to-text and text-to-speech | `whisper-1` · `chat` · `tts-1` |
| Document OCR | Mistral                                          | `mistral-ocr-latest` · `mistral-medium-latest` |
| Smart Crop   | Hugging Face                                     | `facebook/detr-resnet-50`     |

## Models

`chat`, `chat_with_complex_json`, `chat_with_image_vision` and `embeddings` are aliases that amazee.ai
resolves in each region, so the demo works whichever region you pick.
You can use any model your key has access to instead, by changing the `model` of an agent
in `config/packages/ai.yaml`. List the available models with:

```shell
curl -s -H "Authorization: Bearer $AMAZEEAI_LLM_KEY" "$AMAZEEAI_LLM_API_URL/v1/models"
```

The platform bridge discovers these models and their capabilities from the gateway's `/model/info` endpoint,
so no model catalog needs to be maintained in the application.

> [!IMPORTANT]
> The vector table is created with `vector_size: 1024`, which matches `embeddings` (Mistral Embed).
> If you switch to an embedding model with other dimensions, update `setup_options.vector_size` in
> `config/packages/ai.yaml`, then run `ai:store:drop`, `ai:store:setup` and `ai:store:index` again.

## Requirements

* [PHP >= 8.4](https://www.php.net/releases/8.4/en.php) with the `gd`, `intl` and `pdo_pgsql` extensions
* [Composer](https://getcomposer.org/)
* An email address to sign in to [amazee.ai](https://amazee.ai/) with `ai:amazee:configure`
* Optional: the [Symfony CLI](https://symfony.com/download)
* Optional: [Node.js](https://nodejs.org/), only for the MCP example: one of its three servers speaks the
  legacy HTTP+SSE transport and is reached through `npx mcp-remote`
* Optional API keys, in `.env.local`, for the use cases that are not served by amazee.ai:
  * `OPENAI_API_KEY`: speech-to-text and text-to-speech of the Speech example
  * `MISTRAL_API_KEY`: Document OCR
  * `HUGGINGFACE_API_KEY`: Smart Crop

## Configuration

`ai:amazee:configure` writes the following to `.env.local`:

```dotenv
AMAZEEAI_LLM_KEY=sk-...
AMAZEEAI_LLM_API_URL=https://llm.[region].amazee.ai
AMAZEEAI_VDB_HOST=vectordb1.[region].amazee.ai
AMAZEEAI_VDB_PORT=5432
AMAZEEAI_VDB_NAME=db_abcd1234
AMAZEEAI_VDB_USER=user_abcd1234
AMAZEEAI_VDB_PASSWORD=...
```

`AMAZEEAI_VDB_DSN` is composed from these in `.env`. Check the result with `php bin/console debug:dotenv`.

In production (`APP_ENV=prod`), the command stores `AMAZEEAI_LLM_KEY` and `AMAZEEAI_VDB_PASSWORD` as
[Symfony secrets](https://symfony.com/doc/current/configuration/secrets.html) instead.

The platform, vector store and agents are wired in `config/packages/ai.yaml`:

```yaml
ai:
    platform:
        amazeeai:
            base_url: '%env(AMAZEEAI_LLM_API_URL)%'
            api_key: '%env(AMAZEEAI_LLM_KEY)%'
    agent:
        blog:
            platform: 'ai.platform.amazeeai'
            model: 'chat'
    store:
        postgres:
            symfony_blog:
                dsn: '%env(AMAZEEAI_VDB_DSN)%'
                username: '%env(AMAZEEAI_VDB_USER)%'
                password: '%env(AMAZEEAI_VDB_PASSWORD)%'
```

To use amazee.ai in your own project:

```shell
composer require symfony/ai-bundle symfony/ai-amazee-ai-platform amazeeio/symfony-amazeeai-configure
php bin/console ai:amazee:configure you@example.com
```

## Technology

* [PHP >= 8.4](https://www.php.net/releases/8.4/en.php)
* [Symfony 8.1 incl. Twig, Asset Mapper & UX](https://symfony.com/)
* [Symfony AI](https://symfony.com/doc/current/ai/index.html) with the
  [amazee.ai platform bridge](https://github.com/symfony/ai-amazee-ai-platform)
* [amazee.ai](https://amazee.ai/) LLM gateway and vector database
  ([PostgreSQL with pgvector](https://github.com/pgvector/pgvector))

## Testing

```shell
vendor/bin/phpunit                  # unit and integration tests
vendor/bin/phpunit --testsuite e2e  # end-to-end tests in a real browser
```

### End-to-End Tests

> [!WARNING]
> The end-to-end suite is inherited from upstream as is: it still expects OpenAI models, `OPENAI_API_KEY`
> and the local PostgreSQL started with `docker compose up -d`, so it does not cover the amazee.ai setup yet.

The `e2e` suite uses [Symfony Panther](https://github.com/symfony/panther) to click through all eleven
use cases and assert the Symfony AI panel of the profiler for the very request the click triggered.
Every test calls an AI platform for real, which costs money and takes time - the suite is therefore
excluded from the default one, and meant to be run locally.

Next to the setup above, it needs:

* **Chrome or Chromium** with a matching `chromedriver`, which `vendor/bin/bdi detect drivers`
  downloads into `drivers/`. If only a Snap or Flatpak Chromium is installed, point Panther at it
  with `PANTHER_CHROME_BINARY` in `.env.test.local`.
* **API keys** in `.env.local`, or exported in your environment - a test is skipped when the key of
  its use case is missing: `OPENAI_API_KEY` for nine of them, `HUGGINGFACE_API_KEY` for the image
  cropping, `MISTRAL_API_KEY` for the document OCR.
* **ffmpeg** (optional) to convert the audio fixture for the fake microphone of the speech use case.

The blog store does not need to be indexed beforehand: `StoreTest` drives the indexing pipeline
through the console commands, and `BlogTest` sets the store up and indexes it when it is empty. Both
skip themselves when the database is not running.

Panther boots the application in the **dev** environment, because the profiler - and with it the
Symfony AI panel - only collects data with `kernel.debug` enabled. The web server therefore reads
the real API keys from `.env.local` itself. Chrome fakes camera and microphone, so the video and
speech use cases run without a human in front of the screen.

```shell
vendor/bin/phpunit --testsuite e2e --filter BlogTest      # a single use case
PANTHER_NO_HEADLESS=1 vendor/bin/phpunit --testsuite e2e  # watch the browser
```

Screenshots of failing tests are written to `var/error-screenshots/`.

## Functionality

* The chatbot application is a simple and small Symfony 8.1 application.
* The UI is coupled to a [Twig LiveComponent](https://symfony.com/bundles/ux-live-component/current/index.html), that integrates different `Chat` implementations on top of the user's session.
* You can reset the chat context by hitting the `Reset chat` button in the top right corner.
* You find eleven different usage scenarios in the upper navbar.

### MCP

Demo MCP server exposing a `current-time` tool and a **Movies** MCP App — an interactive HTML UI (`#[AsMcpApp]`)
that renders the movie collection as a searchable grid in hosts supporting [MCP Apps](https://github.com/modelcontextprotocol/ext-apps).

To add the server, add the following configuration to your MCP Client's settings, e.g. your IDE:
```json
{
    "servers": {
        "symfony": {
            "command": "php",
            "args": [
                "/your/full/path/to/bin/console",
                "mcp:server"
            ]
        }
    }
}
```

#### Testing the MCP Server

You can test the MCP server by running the following command to start the MCP client:

```shell
symfony console mcp:server
```

**With plain JSON RPC requests**

Then, you can initialize the MCP session with the following JSON RPC request:

```json
{ "jsonrpc": "2.0", "id": 1, "method": "initialize", "params": { "protocolVersion": "2024-11-05", "capabilities": {}, "clientInfo": { "name": "demo-client", "version": "dev" } } }
```

And, to request the list of available tools:

```json
{ "jsonrpc": "2.0", "id": 2, "method": "tools/list" }
```

**With MCP Inspector**

For testing, you can also use the [MCP Inspector](https://modelcontextprotocol.io/docs/tools/inspector):

```shell
npx @modelcontextprotocol/inspector php bin/console mcp:server
```

Which opens a web UI to interactively test the MCP server.

## AI Mate - Development CLI

[Symfony AI Mate](https://github.com/symfony/ai-mate) is a command-line assistant that gives coding
agents Symfony-specific knowledge about this application. There is no server to start: the agent
runs `vendor/bin/mate` like any other command.

### Installation & Setup

**This demo is already configured!** For new projects you can set up AI Mate as follows:

```shell
# Install AI Mate
composer require --dev symfony/ai-mate

# Initialize configuration
vendor/bin/mate init

# Discover available tools
vendor/bin/mate discover
```

`mate init` writes the instructions your agent reads (`mate/AGENT_INSTRUCTIONS.md` plus a managed
block in `AGENTS.md`, imported by `CLAUDE.md`) and installs the skills into `.agents/skills/`, with
a mirror in `.claude/skills/`. No client-specific configuration file is involved.

### Running Tools

```shell
vendor/bin/mate tools:list                            # what is available
vendor/bin/mate tools:inspect symfony-ai-features     # parameters and JSON input schema
vendor/bin/mate tools:call symfony-ai-features        # run it
vendor/bin/mate tools:call symfony-profiler-list --limit=1
```

### Custom Capability Example

This demo includes a **`symfony-ai-features`** tool (see `mate/src/SymfonyAiFeaturesTool.php`) that analyzes the project's
AI configuration and reports all available platforms, agents, tools, stores, and packages.

**Try it in your coding agent:**

> "Which Symfony AI features are available in this demo?"
>
> "What AI agents are configured in this project?"
>
> "Show me all the Symfony AI tools and their configuration"
>
> "What is the current PHP version used in this project?"
>
> "Is the php extension intl installed?"

The agent will call `symfony-ai-features` and the other Mate tools to answer from the project
itself rather than from reading the code.

### Creating Custom Tools

Add a class with a public method carrying `#[MateTool]` under `mate/src/`, then run
`composer dump-autoload` and verify with `vendor/bin/mate tools:list`. See the
[AI Mate documentation](https://symfony.com/doc/current/ai/components/mate.html) for detailed guides.
