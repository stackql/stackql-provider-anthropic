# `anthropic` provider for [`stackql`](https://github.com/stackql/stackql)

This repository is used to generate and document the `anthropic` provider for StackQL, allowing you to query and interact with Anthropic's services using SQL-like syntax. The provider is built using the `@stackql/provider-utils` package, which provides tools for converting OpenAPI specifications into StackQL-compatible provider schemas.

## Prerequisites

To use the Anthropic provider with StackQL, you'll need:

1. An Anthropic account with appropriate API credentials
2. An Anthropic API key with sufficient permissions for the resources you want to access
3. StackQL CLI installed on your system (see [StackQL](https://github.com/stackql/stackql))

## 1. Create the Open API Specification

Since Anthropic doesn't currently provide an official OpenAPI specification, we need to create one based on their API documentation:

```bash
mkdir -p provider-dev/downloaded

# Create a JSON OpenAPI spec based on Anthropic's API documentation
cat > provider-dev/downloaded/anthropic-openapi.json << 'EOF'
{
  "openapi": "3.0.0",
  "info": {
    "title": "Anthropic API",
    "description": "Anthropic API for Claude and other AI models",
    "version": "1.0.0"
  },
  "servers": [
    {
      "url": "https://api.anthropic.com"
    }
  ],
  "components": {
    "securitySchemes": {
      "ApiKeyAuth": {
        "type": "apiKey",
        "in": "header",
        "name": "x-api-key"
      },
      "AnthropicVersionHeader": {
        "type": "apiKey",
        "in": "header",
        "name": "anthropic-version"
      }
    }
  },
  "security": [
    {
      "ApiKeyAuth": [],
      "AnthropicVersionHeader": []
    }
  ],
  "paths": {
    "/v1/messages": {
      "post": {
        "operationId": "createMessage",
        "summary": "Create a message",
        "description": "Create a message and receive a response from Claude",
        "tags": ["messages"],
        "requestBody": {
          "required": true,
          "content": {
            "application/json": {
              "schema": {
                "type": "object",
                "required": ["model", "messages"],
                "properties": {
                  "model": {
                    "type": "string",
                    "description": "The model that will complete your prompt",
                    "example": "claude-3-opus-20240229"
                  },
                  "messages": {
                    "type": "array",
                    "description": "A list of messages comprising the conversation so far",
                    "items": {
                      "type": "object",
                      "required": ["role", "content"],
                      "properties": {
                        "role": {
                          "type": "string",
                          "enum": ["user", "assistant"],
                          "description": "The role of the message's author"
                        },
                        "content": {
                          "type": "string",
                          "description": "The content of the message"
                        }
                      }
                    }
                  },
                  "max_tokens": {
                    "type": "integer",
                    "description": "The maximum number of tokens to generate",
                    "default": 1024
                  },
                  "temperature": {
                    "type": "number",
                    "description": "Amount of randomness injected into the response",
                    "default": 1.0
                  },
                  "system": {
                    "type": "string",
                    "description": "System prompt to guide Claude's behavior"
                  },
                  "metadata": {
                    "type": "object",
                    "description": "An object containing metadata about the request"
                  },
                  "stream": {
                    "type": "boolean",
                    "description": "Whether to stream the response",
                    "default": false
                  }
                }
              }
            }
          }
        },
        "responses": {
          "200": {
            "description": "Message created successfully",
            "content": {
              "application/json": {
                "schema": {
                  "type": "object",
                  "properties": {
                    "id": {
                      "type": "string",
                      "description": "The identifier for the message"
                    },
                    "type": {
                      "type": "string",
                      "description": "The type of object"
                    },
                    "role": {
                      "type": "string",
                      "description": "The role of the message author"
                    },
                    "content": {
                      "type": "array",
                      "description": "The content of the message",
                      "items": {
                        "type": "object",
                        "properties": {
                          "type": {
                            "type": "string",
                            "description": "The type of content"
                          },
                          "text": {
                            "type": "string",
                            "description": "The text content"
                          }
                        }
                      }
                    },
                    "model": {
                      "type": "string",
                      "description": "The model used"
                    },
                    "stop_reason": {
                      "type": "string",
                      "description": "The reason the model stopped generating"
                    },
                    "usage": {
                      "type": "object",
                      "description": "Usage statistics for the request",
                      "properties": {
                        "input_tokens": {
                          "type": "integer",
                          "description": "Number of tokens in the input"
                        },
                        "output_tokens": {
                          "type": "integer",
                          "description": "Number of tokens in the output"
                        }
                      }
                    }
                  }
                }
              }
            }
          }
        }
      }
    },
    "/v1/completions": {
      "post": {
        "operationId": "createCompletion",
        "summary": "Create a completion",
        "description": "Create a completion (legacy API)",
        "tags": ["completions"],
        "requestBody": {
          "required": true,
          "content": {
            "application/json": {
              "schema": {
                "type": "object",
                "required": ["model", "prompt"],
                "properties": {
                  "model": {
                    "type": "string",
                    "description": "The model that will complete your prompt"
                  },
                  "prompt": {
                    "type": "string",
                    "description": "The prompt to complete"
                  },
                  "max_tokens_to_sample": {
                    "type": "integer",
                    "description": "The maximum number of tokens to generate",
                    "default": 1024
                  },
                  "temperature": {
                    "type": "number",
                    "description": "Amount of randomness injected into the response",
                    "default": 1.0
                  },
                  "stop_sequences": {
                    "type": "array",
                    "description": "Sequences that will cause the model to stop generating",
                    "items": {
                      "type": "string"
                    }
                  },
                  "stream": {
                    "type": "boolean",
                    "description": "Whether to stream the response",
                    "default": false
                  }
                }
              }
            }
          }
        },
        "responses": {
          "200": {
            "description": "Completion created successfully",
            "content": {
              "application/json": {
                "schema": {
                  "type": "object",
                  "properties": {
                    "completion": {
                      "type": "string",
                      "description": "The completion text"
                    },
                    "model": {
                      "type": "string",
                      "description": "The model used"
                    },
                    "stop_reason": {
                      "type": "string",
                      "description": "The reason the model stopped generating"
                    }
                  }
                }
              }
            }
          }
        }
      }
    },
    "/v1/models": {
      "get": {
        "operationId": "listModels",
        "summary": "List models",
        "description": "List available models",
        "tags": ["models"],
        "responses": {
          "200": {
            "description": "List of available models",
            "content": {
              "application/json": {
                "schema": {
                  "type": "object",
                  "properties": {
                    "models": {
                      "type": "array",
                      "description": "List of available models",
                      "items": {
                        "type": "object",
                        "properties": {
                          "name": {
                            "type": "string",
                            "description": "Model identifier"
                          },
                          "description": {
                            "type": "string",
                            "description": "Model description"
                          },
                          "context_window": {
                            "type": "integer",
                            "description": "Context window size in tokens"
                          },
                          "max_output_tokens": {
                            "type": "integer",
                            "description": "Maximum tokens in the output"
                          }
                        }
                      }
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
EOF
```

## 2. Split into Service Specs

Next, split the OpenAPI specification into service-specific files:

```bash
rm -rf provider-dev/source/*
npm run split -- \
  --provider-name anthropic \
  --api-doc provider-dev/downloaded/anthropic-openapi.json \
  --svc-discriminator tag \
  --output-dir provider-dev/source \
  --overwrite \
  --svc-name-overrides "$(cat <<EOF
{
  "messages": "messages",
  "completions": "completions",
  "models": "models"
}
EOF
)"
```

## 3. Generate Mappings

Generate the mapping configuration that connects OpenAPI operations to StackQL resources:

```bash
npm run generate-mappings -- \
  --provider-name anthropic \
  --input-dir provider-dev/source \
  --output-dir provider-dev/config
```

Update the resultant `provider-dev/config/all_services.csv` to add the `stackql_resource_name`, `stackql_method_name`, `stackql_verb` values for each operation.

## 4. Generate Provider

This step transforms the split OpenAPI service specs into a fully-functional StackQL provider by applying the resource and method mappings defined in your CSV file.

```bash
rm -rf provider-dev/openapi/*
npm run generate-provider -- \
  --provider-name anthropic \
  --input-dir provider-dev/source \
  --output-dir provider-dev/openapi/src/anthropic \
  --config-path provider-dev/config/all_services.csv \
  --servers '[{"url": "https://api.anthropic.com"}]' \
  --provider-config '{"auth": {"type": "multi_header", "credentials": {"headers": [{"name": "x-api-key", "envVar": "ANTHROPIC_API_KEY"}, {"name": "anthropic-version", "value": "2023-06-01"}]}}}' \
  --overwrite
```

## 5. Test Provider

### Starting the StackQL Server

Before running tests, start a StackQL server with your provider:

```bash
PROVIDER_REGISTRY_ROOT_DIR="$(pwd)/provider-dev/openapi"
npm run start-server -- --provider anthropic --registry $PROVIDER_REGISTRY_ROOT_DIR
```

### Test Meta Routes

Test all metadata routes (services, resources, methods) in the provider:

```bash
npm run test-meta-routes -- anthropic --verbose
```

When you're done testing, stop the StackQL server:

```bash
npm run stop-server
```

Use this command to view the server status:

```bash
npm run server-status
```

### Run test queries

Run some test queries against the provider using the `stackql shell`:

```bash
PROVIDER_REGISTRY_ROOT_DIR="$(pwd)/provider-dev/openapi"
REG_STR='{"url": "file://'${PROVIDER_REGISTRY_ROOT_DIR}'", "localDocRoot": "'${PROVIDER_REGISTRY_ROOT_DIR}'", "verifyConfig": {"nopVerify": true}}'
./stackql shell --registry="${REG_STR}"
```

Example queries to try:

```sql
-- List available models
SELECT 
*
FROM anthropic.models.models;

-- Create a message to get a response from Claude
INSERT INTO anthropic.messages.messages (
  json_data
) VALUES (
  '{
    "model": "claude-3-opus-20240229",
    "messages": [
      {
        "role": "user",
        "content": "What are the main differences between Claude 3 Opus and Claude 3 Sonnet?"
      }
    ],
    "max_tokens": 500,
    "temperature": 0.7
  }'
);

-- Create a completion (legacy API)
INSERT INTO anthropic.completions.completions (
  json_data
) VALUES (
  '{
    "model": "claude-2.1",
    "prompt": "\n\nHuman: Explain quantum computing in simple terms\n\nAssistant: ",
    "max_tokens_to_sample": 300,
    "temperature": 0.7,
    "stop_sequences": ["\n\nHuman:"]
  }'
);
```

## 6. Publish the provider

To publish the provider push the `anthropic` dir to `providers/src` in a feature branch of the [`stackql-provider-registry`](https://github.com/stackql/stackql-provider-registry). Follow the [registry release flow](https://github.com/stackql/stackql-provider-registry/blob/dev/docs/build-and-deployment.md).  

Launch the StackQL shell:

```bash
export DEV_REG="{ \"url\": \"https://registry-dev.stackql.app/providers\" }"
./stackql --registry="${DEV_REG}" shell
```

Pull the latest dev `anthropic` provider:

```sql
registry pull anthropic;
```

Run some test queries to verify the provider works as expected.

## 7. Generate web docs

Provider doc microsites are built using Docusaurus and published using GitHub Pages.  

a. Update `headerContent1.txt` and `headerContent2.txt` accordingly in `provider-dev/docgen/provider-data/`  

b. Update the following in `website/docusaurus.config.js`:  

```js
// Provider configuration - change these for different providers
const providerName = "anthropic";
const providerTitle = "Anthropic Provider";
```

c. Then generate docs using...

```bash
npm run generate-docs -- \
  --provider-name anthropic \
  --provider-dir ./provider-dev/openapi/src/anthropic/v00.00.00000 \
  --output-dir ./website \
  --provider-data-dir ./provider-dev/docgen/provider-data
```  

## 8. Test web docs locally

```bash
cd website
# test build
yarn build

# run local dev server
yarn start
```

## 9. Publish web docs to GitHub Pages

Under __Pages__ in the repository, in the __Build and deployment__ section select __GitHub Actions__ as the __Source__. In Netlify DNS create the following records:

| Source Domain | Record Type  | Target |
|---------------|--------------|--------|
| anthropic-provider.stackql.io | CNAME | stackql.github.io. |

## License

MIT

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.