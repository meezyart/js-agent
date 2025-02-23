# JS Agent: Build GPT Agents with JS & TS

JS Agent is a composable and extensible framework for creating GPT agents with JavaScript and TypeScript.

While creating an agent prototype is easy, increasing its reliability and robustness is difficult and requires considerable experimentation. JS Agent provides robust building blocks and tooling to help you develop rock-solid agents faster.

**⚠️ JS Agent is currently in its initial experimental phase. Prior to reaching version 0.1, there may breaking changes in each release.**

## Quick Start

```sh
npm install js-agent
```

See examples below for details on how to implement and run an agent.

## Features

- Agent definition and execution
  - Configurable agent run properties that can be accessed by prompts
  - Observe agent runs (to support console output, UIs, server runs, webapps, etc.)
  - Record all LLM calls of an agent run
  - Calculate the cost of LLM calls and agent runs
  - Stop agent runs when certain criteria are met, e.g. to limit the number of steps
  - Use several different LLM models in one agent
- Supported LLM models
  - OpenAI text completion models (`text-davinci-003` etc.)
  - OpenAI chat completion models (`gpt-4`, `gpt-3.5-turbo`)
- Prompt template creation
  - Create templates for text prompts and chat prompts
  - Utility functions to combine and convert prompts
- Actions and Tools
  - Read and write file
  - Run CLI command
  - Use programmable search engine
  - Summarize website according to topic
  - Ask user for input
  - Optional agent/executor separation (e.g. run the executor in a sandbox environment such as a Docker container)
- Text functions
  - Extract text from webpage
  - Split text into chunks
  - Summarize text recursively
- General utils
  - LLM call retry with exponential backoff

## Examples

**[BabyAGI](https://github.com/lgrammel/js-agent/tree/main/examples/babyagi)**:
JS Agent implementation of [BabyAGI](https://github.com/yoheinakajima/babyagi).

**[JavaScript/TypeScript developer](https://github.com/lgrammel/js-agent/tree/main/examples/javascript-developer)**:
An automated developer agent that works in a docker container.
It can read files, write files and execute commands.
You can use it to document code, write tests, update tests and features, etc.

**[Wikipedia Question-Answering](https://github.com/lgrammel/js-agent/tree/main/examples/wikipedia)**:
An agent that has access to a wikipedia search engine and can read wikipedia articles. You can use it to answer questions about wikipedia content.

## Design Principles

- **typed**: Provide as much typing as possible to support discovery and ensure safety.
- **composable**: The individual pieces should have a good separation of concerns and be easy to combine.
- **extensible**: It should be easy for users to add their own tools, providers, actions, agent steps, etc.
- **use functional programming for object assembly**: All objects that are immutable are assembled using functional programming. Object-orientation is only used for objects that have a changeable state (e.g. `Step` and `AgentRun`).
- **support progressive refinement of agent specifications**: Agent specifications should be easy to write and every building block should provide good defaults. At the same time, it should be possible to easily override the defaults with specific settings, prompts, etc.
- **build for production**: JS Agent will have first-class support for logging, associating LLM calls and cost tracking with agent runs, etc.

## Example Agent

```ts
import * as $ from "js-agent";

export async function runWikipediaAgent({
  wikipediaSearchKey,
  wikipediaSearchCx,
  openAiApiKey,
  task,
}: {
  openAiApiKey: string;
  wikipediaSearchKey: string;
  wikipediaSearchCx: string;
  task: string;
}) {
  type WikipediaAgentRunProperties = { task: string };

  const chatGpt = $.provider.openai.chatModel({
    apiKey: openAiApiKey,
    model: "gpt-3.5-turbo",
  });

  const searchWikipediaAction =
    $.tool.programmableGoogleSearchEngineAction<WikipediaAgentRunProperties>({
      id: "search-wikipedia",
      description:
        "Search wikipedia using a search term. Returns a list of pages.",
      execute: $.tool.executeProgrammableGoogleSearchEngineAction({
        key: wikipediaSearchKey,
        cx: wikipediaSearchCx,
      }),
    });

  const readWikipediaArticleAction =
    $.tool.summarizeWebpage<WikipediaAgentRunProperties>({
      id: "read-wikipedia-article",
      description:
        "Read a wikipedia article and summarize it considering the query.",
      inputExample: {
        url: "https://en.wikipedia.org/wiki/Artificial_intelligence",
        topic: "{query that you are answering}",
      },
      execute: $.tool.executeSummarizeWebpage({
        extractText: $.text.extractWebpageTextFromHtml(),
        summarize: $.text.summarizeRecursively({
          split: $.text.splitRecursivelyAtCharacter({
            maxCharactersPerChunk: 2048 * 4, // needs to fit into a gpt-3.5-turbo prompt
          }),
          summarize: $.text.generate({
            id: "summarize-wikipedia-article-chunk",
            prompt: $.text.SummarizeChatPrompt,
            model: chatGpt,
            processOutput: async (output) => output.trim(),
          }),
        }),
      }),
    });

  return $.runAgent<{ task: string }>({
    properties: { task },
    agent: $.step.generateNextStepLoop({
      actions: [searchWikipediaAction, readWikipediaArticleAction],
      actionFormat: $.action.format.flexibleJson(),
      prompt: $.prompt.concatChatPrompts(
        async ({ runProperties: { task } }) => [
          {
            role: "system",
            content: `## ROLE
You are an knowledge worker that answers questions using Wikipedia content. You speak perfect JSON.

## CONSTRAINTS
Make sure all facts for your answer are from Wikipedia articles that you have read.`,
          },
          {
            role: "user",
            content: `## TASK\n${task}`,
          },
        ],
        $.prompt.availableActionsChatPrompt(),
        $.prompt.recentStepsChatPrompt({ maxSteps: 6 })
      ),
      model: chatGpt,
    }),
    controller: $.agent.controller.maxSteps(20),
    observer: $.agent.observer.combineObservers(
      $.agent.observer.showRunInConsole({ name: "Wikipedia Agent" }),
      {
        async onRunFinished({ run }) {
          const runCostInMillicent = await $.agent.calculateRunCostInMillicent({
            run,
          });

          console.log(
            `Run cost: $${(runCostInMillicent / 1000 / 100).toFixed(2)}`
          );

          console.log(
            `LLM calls: ${
              run.recordedCalls.filter((call) => call.success).length
            }`
          );
        },
      }
    ),
  });
}
```
