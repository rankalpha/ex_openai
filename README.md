# Elixir SDK for OpenAI APIs

[![Hex.pm Version](https://img.shields.io/hexpm/v/ex_openai)](https://hex.pm/packages/ex_openai)
[![Hex Docs](https://img.shields.io/badge/hex-docs-lightgreen.svg)](https://hexdocs.pm/ex_openai)
[![Hex.pm Download Total](https://img.shields.io/hexpm/dt/ex_openai)](https://hex.pm/packages/ex_openai)

ExOpenAI is an (unofficial) Elixir SDK for interacting with the [OpenAI APIs](https://platform.openai.com/docs/api-reference/introduction)

This SDK is fully auto-generated using [metaprogramming](https://elixirschool.com/en/lessons/advanced/metaprogramming/) and should always reflect the latest state of the OpenAI API.

**Note:** Due to the nature of auto-generating something, you may encounter stuff that isn't working yet. Make sure to report if you notice anything acting up.

## Features

- Up-to-date and complete thanks to metaprogramming and code-generation
- Implements _everything_ the OpenAI has to offer
- Strictly follows the official OpenAI APIs for argument/function naming
- Handling of required arguments as function parameters and optional arguments as Keyword list in true Elixir-fashion
- Auto-generated embedded function documentation
- Auto-generated @spec definitions for dialyzer, for strict parameter typing
- Streaming data

## Installation

Add **_:ex_openai_** as a dependency in your mix.exs file.

```elixir
def deps do
  [
    {:ex_openai, "~> 1.0.4"}
  ]
end
```

## Supported endpoints (basically everything)

- /answers
- /chat/completions
- /classifications
- /completions
- /edits
- /embeddings
- /engines/{engine_id}
- /engines
- /files
- /files/{file_id}/content
- /fine-tunes/{fine_tune_id}/events
- /fine-tunes/{fine_tune_id}/cancel
- /fine-tunes/{fine_tune_id}
- /fine-tunes
- /images/generations
- /images/variations
- /images/edits
- /models
- /moderations
- /engines/{engine_id}/search
- /audio/translations
- /audio/transcriptions

### Editor features: Autocomplete, specs, docs

#### Autocompletion/type-hinting through LSP / ElixirSense

<img src="images/autocomplete.png" width="500" />

#### Typechecking and diagnostics through strict @spec definitions

<img src="images/diagnostics.png" width="500" />

#### Inline docs and signatures thanks to @spec and @doc

<img src="images/functiondocs.png" width="600" />

## To Do's / What's not working yet

- Typespecs for `oneOf` input types, currently represented as `any()`
- Streams need some work and proper types

## Configuration

```elixir
import Config

config :ex_openai,
  # find it at https://platform.openai.com/account/api-keys
  api_key: System.get_env("OPENAI_API_KEY"),
  # find it at https://platform.openai.com/account/api-keys
  organization_key: System.get_env("OPENAI_ORGANIZATION_KEY"),
  # optional, passed to [HTTPoison.Request](https://hexdocs.pm/httpoison/HTTPoison.Request.html) options
  http_options: [recv_timeout: 50_000]
```

## Usage

Make sure to checkout the docs: https://hexdocs.pm/ex_openai

```elixir
ExOpenAI.Models.list_models
{:ok,
 %{
   data: [
     %{
       "created": 1649358449,
       "id": "babbage",
       "object": "model",
       "owned_by": "openai",
       "parent": nil,
       "permission": [
         %{
           "allow_create_engine": false,
           "allow_fine_tuning": false,
           "allow_logprobs": true,
           "allow_sampling": true,
           "allow_search_indices": false,
           "allow_view": true,
           "created": 1669085501,
           "group": nil,
           "id": "modelperm-49FUp5v084tBB49tC4z8LPH5",
           "is_blocking": false,
           "object": "model_permission",
           "organization": "*"
         }
       ],
       "root": "babbage"
     },
  ...
```

Required parameters are converted into function arguments, optional parameters into the opts keyword list:

```elixir
ExOpenAI.Completions.create_completion "text-davinci-003", prompt: "The sky is"
{:ok,
 %{
   choices: [
     %{
       "finish_reason": "length",
       "index": 0,
       "logprobs": nil,
       "text": " blue\n\nThe sky is a light blue hue that may have a few white"
     }
   ],
   created: 1677929239,
   id: "cmpl-6qKKllDPsQRtyJ5oHTbkQVS9w7iKM",
   model: "text-davinci-003",
   object: "text_completion",
   usage: %{
     "completion_tokens": 16,
     "prompt_tokens": 3,
     "total_tokens": 19
   }
 }}
```

### Using ChatGPT APIs

```elixir
msgs = [
  %{role: "user", content: "Hello!"},
  %{role: "assistant", content: "What's up?"},
  %{role: "user", content: "What ist the color of the sky?"}
]

{:ok, res} =
  ExOpenAI.Chat.create_chat_completion(msgs, "gpt-3.5-turbo",
    logit_bias: %{
      "8043" => -100
    }
  )
```

### Usage of endpoints that require files to upload

Load your file into memory, then pass it into the file parameter

```elixir
duck = File.read!("#{__DIR__}/testdata/duck.png")

{:ok, res} = ExOpenAI.Images.create_image_variation(duck)

IO.inspect(res.data)
```

### Streaming data (experimental)

Streaming data is still experimental, YMMV!

**At the time of writing, streaming support is not yet released to hex, so use the master of this repository**

Create a new client for receiving the streamed data with `use ExOpenAI.StreamingClient`. You'll have to implement the `@behaviour ExOpenAI.StreamingClient`:

```elixir
defmodule MyStreamingClient do
  use ExOpenAI.StreamingClient

  @impl true
  # callback on data
  def handle_data(data, state) do
    IO.puts("got data: #{inspect(data)}")
    {:noreply, state}
  end

  @impl true
  # callback on error
  def handle_error(e, state) do
    IO.puts("got error: #{inspect(e)}")
    {:noreply, state}
  end

  @impl true
  # callback on finish
  def handle_finish(state) do
    IO.puts("finished!!")
    {:noreply, state}
  end
end
```

Then use it in requests that support streaming by setting `stream: true` and specifying `stream_to: pid`:

```elixir
{:ok, pid} = MyStreamingClient.start_link nil
ExOpenAI.Completions.create_completion "text-davinci-003", prompt: "hello world", stream: true, stream_to: pid
```

Your client will now receive the streamed chunks

#### Caveats

- Type information for streamed data is not correct yet. For Completions.create_completion it's fine, however Chat.create_chat_completion requests use a different struct with a `delta` field
- `stream_to: ` is not part of the typespec yet as it's a custom field created by this SDK and not in OpenAI API docs
- Return types for when setting `stream: true` is incorrect


## How to update once OpenAI changes something?

Run `mix update_openai_docs` and commit the new `docs.yaml` file

## Some stuff built using this SDK (add yours with a PR!)

- [Elixir ChatGPT](https://github.com/dvcrn/elixir-chatgpt)
- https://fixmyjp.d.sh
- https://github.com/dvcrn/gpt-slack-bot
- https://david.coffee/mini-chatgpt-in-elixir-and-genserver/

## License

The package is available as open source under the terms of the [MIT License](https://opensource.org/licenses/MIT).

## Attribution

- Inspired by https://github.com/BlakeWilliams/Elixir-Slack
- Client/config handling from https://github.com/mgallo/openai.ex
