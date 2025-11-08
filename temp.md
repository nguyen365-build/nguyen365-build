Of course. Here are the instructions for getting started with the Gemini CLI.

### 🐧 What is the Gemini CLI?

The **Gemini CLI** is an open-source tool that brings Google's Gemini models directly into your terminal. It's like having a chat with Gemini, but it's deeply integrated with your local development environment.

You can use it to:

  * Ask questions about your code.
  * Generate code, documentation, or commit messages.
  * Debug errors and troubleshoot issues.
  * Summarize or explain files.
  * Interact with your file system and run shell commands.

-----

### 🚀 Getting Started: Installation & Setup

Follow these steps to get it running on your system.

#### 1\. Prerequisites

You must have **Node.js version 20 or higher** installed. You can check your version by opening your terminal and running:

```bash
node -v
```

If you don't have it, you can download it from the official [Node.js website](https://nodejs.org/).

#### 2\. Installation

You have two main options to install the Gemini CLI.

  * **Option 1: Recommended (Global Install)**
    This installs the `gemini` command on your system so you can run it from any directory.

    ```bash
    npm install -g @google/gemini-cli
    ```

  * **Option 2: Run without Installing (npx)**
    This is a great way to try it out without installing it permanently.

    ```bash
    npx @google/gemini-cli
    ```

    *Note: On macOS, you might also be able to install it via Homebrew with `brew install gemini-cli`.*

#### 3\. Authentication

The first time you run the tool, you'll need to log in.

1.  Run the `gemini` command in your terminal:
    ```bash
    gemini
    ```
2.  It will prompt you to authenticate. Choose **"Login with Google"**.
3.  Your web browser will open. Follow the steps to sign in with your Google account and grant the necessary permissions.
4.  Once authorized, you can return to your terminal.

This one-time login gives you access to a generous free tier for personal use.

-----

### 💬 How to Use It

Using the Gemini CLI is as simple as starting a conversation.

#### Basic Chatting

Once you're in the `gemini` prompt (it looks like a `>`), you can type your questions in plain English.

> `> What's the difference between a class and an interface in TypeScript?`

> `> Write a PowerShell script to find all .log files in the current directory and delete any that are older than 30 days.`

> `> I'm getting this error: 'Permission denied'. What are the common causes?`

#### Built-in Commands

The CLI also has special "slash" commands for specific actions. Type `/` to see a list. Here are the most important ones:

  * `/help`: Shows a full list of commands and keyboard shortcuts.
  * `/chat`: Starts a new, empty chat session.
  * `/clear`: Clears your terminal screen.
  * `/quit` or `/exit`: Exits the Gemini CLI.

#### Using File Context

The real power of the CLI comes from making it "aware" of your project.

  * **Start in Your Project Folder:** Before you run the `gemini` command, `cd` into your project's directory. This gives Gemini context of the files in that folder.
  * **Reference Files:** You can "attach" files to your prompt using the `@` symbol.
      * `> Explain the code in @./src/index.js`
      * `> Based on @./package.json, what scripts can I run?`
  * **Project-Level Context (GEMINI.md):** You can create a file named `GEMINI.md` in the root of your project. Put instructions in this file to give Gemini persistent context, such as your coding style, project goals, or preferred technologies.

Here is a short tutorial that walks you through installing and using the Google Gemini CLI.
