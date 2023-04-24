---
theme: ./theme
layout: intro
title: "CLI GPT: What is it and how did I make it?"
image: /images/intro.jpg
---

# CLI GPT

What is it and how did I make it?

<div class="absolute bottom-10">
  <span class="font-700">
    Jiří Prokop @ Bayzat
  </span>
</div>

---
layout: bullets
---

# What is cli-gpt

- CLI tool emulating ChatGPT in your terminal
- conversations are persisted per folder
- initial system message and example conversation can be provided
- local files can be included in the conversation
- one-shot and multiline modes
- model's parameters can be configured

<!--
- `cli-gpt` is CLI tool emulating ChatGPT in your terminal
- you can have conversations persisted per folder which is useful when you work on different projects
- it allows to specify initial system message and example conversation (which is recommended to direct GPT's behaviour)
- local files (including their path and name) can be included in the conversation
- it has one-shot and multiline modes
- model's parameters can be configured globally too (temperature aka creativity etc.)
-->

---
layout: intro-image-right
image: /images/vscode.gif
---

# My motivation for creating it

- ChatGPT is cumbersome
- more focused flow; easy attachment of files
- nicely integrates into VSCode's terminal
- quick one-shot questions while browsing in floating terminal window
- can be aliased to `ai` and similar
- learn how to work with [OpenAI's API](https://platform.openai.com/docs/guides/chat)

<!--
- I found ChatGPT cumbersome when using it during development.
- With `cli-gpt` I can stay more focused, I can easily attach files when needed.
- The flow I use in VSCode is having one terminal tab dedicated to conversation with `cli-gpt`.
- For quick one-shot questions while browsing, I can open new floating terminal window and ask it directly.
- The `cli-gpt` command can be aliased to `ai` and similar.
- I also wanted to learn how to work with OpenAI's API when I saw a lot of different other tools.
-->

---
layout: section
---

# How did I make it

---
layout: bullets
---

# How did I make it (part 1)

- [Deno](https://deno.land/), GitHub Actions & OpenAI's API
- Used ChatGPT to create a quick draft of the tool
- Used the draft tool to improve itself

<!--
- I wanted to create it quickly and also learn a bit about Deno, GitHub Actions and AI utilization for creative process.
- I initially used ChatGPT to create a draft of the tool.
- Once I had something working, I started using the tool itself to improve it.
  The advantage was that I could include up-to-date files and ask AI to make changes for me.
  Then it was just a matter of copy and pasting it back.
-->

---
layout: bullets
---

# How did I make it (part 2)

- Modern JavaScript & Deno features
  - [`for await`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for-await...of) and [`yield`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function*)
  - [`fetch`](https://developer.mozilla.org/en-US/docs/Web/API/fetch)
- Response streaming (aka AI typing back)
- Deno's Node.js compatibility layer for prompt
- GitHub Actions for checks & releases

<!--
- I used modern JavaScript and Deno features like:
	- `for await` and `yield` (and async iterators/generators)
	- `fetch`
- This allowed me to easily implement streaming of the response
- I also used Deno's Node.js compatibility layer to work with terminal fuctionality (ie. multiline prompt)
- I used GitHub Actions to check the code and publish the binary
-->

---
layout: bullets
---

# How did I make it (part 3)

- AI helped with [`README.md`](https://github.com/synaptiko/cli-gpt/blob/master/README.md#fun-fact) and [`--help`](https://github.com/synaptiko/cli-gpt/blob/master/src/printHelp.ts)
- Also with [plugin system brainstorming](https://github.com/synaptiko/cli-gpt/blob/master/brainstorming/Plugin%20System.md)

<!--
- AI also helped me with various tasks like creating `README.md` and command documentation for `--help`
- I also used it for initial brainstorming about plugin system
-->

---
layout: section
---

# Project structure

Directories, files, and their organization

---
layout: bullets
---

# Project structure: /

- `README.md`: Contains all important information.
- `src`: Contains all the code.
- `deno.json`: Contains `run` task.
- `bundle.ts`: Contains script to create a single binary for distribution.
  - I couldn't use `deno compile` because it doesn't support `node:` imports yet.
- `.github/workflows`: GHAs

---
layout: bullets
---

# Project structure: /src

- `index.ts`: The main entry point of the CLI tool, handling user inputs and coordinating the application's operation.
- `ChatCompletion.ts`: Responsible for interacting with the OpenAI Chat Completion API to retrieve and process chatbot responses.
- `ConversationPersistance.ts`: Handles message storage and retrieval for the ongoing conversation context.
- `prompt.ts`: Manages user prompts for input, supporting both single-line and multi-line input modes.
- `loadConfig.ts`: Loads and validates configuration parameters from environment variables.
- `parseArgs.ts`: Parses command-line arguments and validates user inputs for the application.
- `printHelp.ts`: Displays help information and usage instructions for the CLI tool.

---
layout: section
---

# Code highlights

Interesting portions of the code

---
layout: section
---

## 1. Stream results of chat completion

---
layout: section
---

```ts
async *complete(): AsyncGenerator<string> {
  // ...
  const response = await fetch('https://api.openai.com/v1/chat/completions', { body: { /* ... */ stream: true } });
  // ...
  for await (const chunk of response.body) {
    const lines = chunk.split('\n');

    for (const line of lines) {
      if (line.startsWith('data: ')) {
        const payload = line.slice(6);

        if (payload !== '[DONE]') {
          const message = JSON.parse(payload);
          const delta = message.choices[0].delta;
          const { content } = delta;

          if (content !== undefined) {
            yield content;
          }
        }
      }
    }
  }
}
```

[`ChatCompletion.ts`](https://github.com/synaptiko/cli-gpt/blob/master/src/ChatCompletion.ts#L28)

---
layout: section
---

## 2. Use Node's modules in Deno for prompt

---
layout: section
---

```ts
import readline from 'node:readline';
import process from 'node:process';

export function prompt(multiline = false): Promise<string> {
  const rl = readline.createInterface({ input: process.stdin, output: process.stdout, prompt: '' });

  return new Promise((resolve) => {
    const text: string[] = [];

    rl.on('line', (line: string) => {
      text.push(line);

      if (!multiline) {
        rl.close();
      }
    });

    rl.on('close', () => {
      resolve(text.join('\n'));
    });
  });
}
```

[`prompt.ts`](https://github.com/synaptiko/cli-gpt/blob/master/src/prompt.ts)

---
layout: section
---

# Practical usage examples

---
layout: bullets
---

## 1. Ask quick questions from your home folder

- Create a `.cli-gpt.initial.md` file with a system message instructing GPT to provide concise answers to generic questions:
  ```bash
  cli-gpt -i -s "You a helpful AI assistant. Your goal is to provide concise, relevant, and accurate"\
   "answers to a wide range of generic questions. Focus on delivering information in a clear and brief manner."\
   "Be as concise as possible."
  ```

- Use one-shot mode to ask quick questions and get useful but brief responses:
  ```bash
  cli-gpt -o "What is user type guard function in TypeScript?"
  ```

---
layout: bullets
---

## Summarize

- Create a folder named `.summarize` and initialize system message:
  ```bash
  cd && mkdir .summarize && cd .summarize
  cli-gpt -i -s "You are an AI assistant skilled in summarizing text."\
   "Your goal is to distill long articles or documents into concise and relevant summaries,"\
   "capturing the main points and essential information. Be as concise as possible."
  ```

- Use one-shot mode and multiline mode to summarize articles:
  ```bash
  cd .summarize && cli-gpt -o -m
  ```
- Paste your article to the prompt, press `<Enter>` and `<Cmd>+d`.

_Tip: create an alias for the command above._

---
layout: bullets
---

## Bootstrap new project(s)

- Create your project folder and initialize system message:
  ```bash
  mkdir Projects && cd Projects
  cli-gpt -i -s "You are a helpful assistant. Your answers are commands for terminal usage only."\
   "You must avoid explanations or comments."
  ```
- Ask it to bootstrap the project:
  ```bash
  cli-gpt -o "Create React and TypeScript project with Vite & Node.js. Project's name is ExtraordinaryMole."
  ```

---
layout: bullets
---

# Future of cli-gpt

- Plugin system
	- intercepting outgoing/incoming messages
	- execute commands, chain other AI APIs/tools

- Inspiration from:
  - [Toolkit: AI plugins](https://www.toolkit.club/)
  - [LangChain](https://js.langchain.com/docs/)
  - [AutoGPT](https://github.com/Significant-Gravitas/Auto-GPT)
  - [BabyAGI](https://github.com/yoheinakajima/babyagi)
  - [`opencommit`](https://github.com/di-sukharev/opencommit)/[`aicommits`](https://github.com/Nutlope/aicommits)

<!--
- I'd like to implement a plugin system
- Plugins can enhance the functionality by intercepting outgoing & incoming messages
  For example: plugin can understand when AI suggests to edit a file and it can update the local file based on the instruction for you;
               or you can create a command which will create a changelog entry for you and similar

- I got inspired by various new tools using GPT, namely
  - AI app frameworks like LangChain and Toolkit which can generate plugins for it
	- Agent systems like AutoGPT, BabyAGI etc.
	- CLI dev tools like `opencommit` and `aicommits`
-->

---
layout: section
---

# Q&A

---
layout: intro-image-right
image: /images/qr-code.png
---

# Thanks for listening👂

Feel free to check **[the repository](https://github.com/synaptiko/cli-gpt)**.
<br/><br/>
If you have any **ideas**, **fork it**!
<br/>
And feel free to **contribute back**.

<br/>
<br/>
<br/>

_Slides at_
<br/>
_[https://synaptiko.github.io/cli-gpt-slides/](https://synaptiko.github.io/cli-gpt-slides/)_