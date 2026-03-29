# 01-2 Installation and First Success
> Choose the right installation method, complete setup in five minutes, and let Claude Code scaffold your first TypeScript project.

## Learning Objectives
- Explain the differences between the three installation methods and choose the one that fits your environment
- Complete the OAuth authentication flow and pass the `claude doctor` health check
- Use Claude Code to scaffold a TypeScript project that meets the specification
- Run `npm test` and `npx tsc` and see zero-error success output

---

## Content

### 1. Why the Installation Method Matters

For most tools, the installation method is just a means to an end. But with Claude Code, the installation method directly determines three things: **update mechanism**, **permission model**, and **version stability**.

Installing with `sudo npm install -g` writes the global package to a system directory. Every subsequent update will require sudo, and permission issues will continue to haunt you. Installing via npm is convenient but does not auto-update; you need to track new versions yourself.

The native installer is the official distribution channel maintained by Anthropic. It handles automatic updates and does not depend on your local Node.js environment. This is the most stable choice for day-to-day development.

---

### 2. Three Installation Methods

| Method | Command | Auto-Update | Use Case |
|--------|---------|-------------|----------|
| Native Installer (Recommended) | macOS/Linux: `curl -fsSL https://claude.ai/install.sh \| bash`<br>Windows PowerShell: `irm https://claude.ai/install.ps1 \| iex` | Yes | Day-to-day development |
| npm | `npm install -g @anthropic-ai/claude-code` | No | CI environments that need a pinned version |
| Homebrew | `brew install --cask claude-code` | No | macOS users who prefer managing tools with brew |

**Prerequisites for npm installation**: Your machine needs Node.js 18.x or above. Run `node --version` to check. If the version does not meet the requirement, install the correct version via [nvm](https://github.com/nvm-sh/nvm) (macOS/Linux) or [nvm-windows](https://github.com/coreybutler/nvm-windows). Do not modify the system Node.js directly.

**Do not use sudo with npm install -g**. sudo causes the global package to be written to a root-owned directory, forcing all subsequent npm operations to require elevated privileges. This is an environment pollution problem that is difficult to reverse.

---

### 3. Authentication Flow

After installation, the first time you run `claude`, the terminal will output an authentication URL and automatically open your browser.

The authentication page supports the following account types:
- **Claude Pro / Max**: Personal subscription accounts
- **Teams / Enterprise**: Organization accounts, enabled by an administrator
- **Anthropic Console (API Key)**: Direct API key usage, suitable for automation scenarios

After completing browser authorization, the authentication token is stored in a local configuration file. Subsequent launches of `claude` do not require re-authentication unless the token expires or you manually run `claude logout`.

---

### 4. Verifying the Installation

After installation and authentication are complete, run the following two commands to confirm the environment is working properly:

```bash
claude --version
```

Example output: `claude 1.x.x`, confirming the CLI runs correctly.

```bash
claude doctor
```

`claude doctor` checks authentication status, network connectivity, and whether dependency requirements are met. All items must show green checkmarks to be considered a full pass. If any item fails, `doctor` will provide specific remediation suggestions.

---

### 5. IDE Integration (Optional)

Claude Code is fundamentally a terminal tool and works fully without an IDE. If you prefer using it within an IDE:

- **VS Code**: Search for "Claude Code" in the Extensions Marketplace and install it. Once installed, you can open a Claude Code conversation directly from the editor sidebar.
- **JetBrains (IntelliJ, WebStorm, etc.)**: Search for "Claude Code" in the Plugin Marketplace and install it.

Both integration methods simply provide an embedded terminal interface. Under the hood, it is the same CLI tool with identical behavior. All exercises in this course are terminal-based; IDE integration is a matter of personal preference.

---

## Hands-On Example: Scaffolding the AI Dev Assistant Project
> Continuing the main project: This is the starting point for the AI Dev Assistant project. All subsequent chapters will build on this scaffold.

**Goal**: Use Claude Code to scaffold a TypeScript project that meets the specification, including a proper tsconfig, test configuration, and npm scripts.

**Prerequisites**: Claude Code installation is complete, and all `claude doctor` checks pass.

**Steps**:

1. Create and enter the project directory in your terminal:
   ```bash
   mkdir ai-dev-assistant && cd ai-dev-assistant
   ```

2. Launch Claude Code:
   ```bash
   claude
   ```
   When you see the Claude prompt, you are in interactive mode.

3. Enter the following instruction to have Claude scaffold the entire project:
   ```
   Initialize a TypeScript project:
   - Use npm init to create package.json
   - Install typescript@5, jest@29, ts-jest, @types/jest, eslint@8, prettier@3 as devDependencies
   - Create tsconfig.json (target: ES2022, module: NodeNext, strict: true)
   - Create src/index.ts (export an empty main function)
   - Create tests/index.test.ts (test that the main function exists)
   - Add scripts to package.json: build, test, lint
   ```

4. Observe Claude's execution process. You will see Claude sequentially run shell commands, create files, and write content. At each step, Claude will show what it is doing and why.

5. After Claude finishes, do not close the conversation. First, verify the results in another terminal window:
   ```bash
   npm test
   ```
   Expected output includes `Tests: 1 passed`.

   ```bash
   npx tsc
   ```
   No output means compilation succeeded. Any output indicates type errors that need to be fixed.

6. Confirm the directory structure:
   ```bash
   ls -la
   ```
   You should see `package.json`, `tsconfig.json`, `src/`, `tests/`, `node_modules/`.

**Expected Results**:
- `package.json`'s `devDependencies` includes typescript@5.x, jest@29.x, ts-jest, @types/jest, eslint@8.x, prettier@3.x
- `tsconfig.json` is configured with `target: "ES2022"`, `module: "NodeNext"`, `strict: true`
- `src/index.ts` exists and exports a `main` function
- `tests/index.test.ts` exists and contains at least one test
- `npm test` outputs `1 passed`
- `npx tsc` produces no error output

**Common Errors**:

- **`npm ERR! EACCES permission denied`**: The global Node.js directory is owned by root. The solution is to switch to nvm for managing Node.js versions rather than trying to fix the existing permission issue, because the problem is likely to recur after a fix.

- **`Cannot find module 'ts-jest'`**: Claude sometimes forgets to install ts-jest when configuring Jest. Go back to the Claude Code conversation and type "ts-jest is not installed, please run npm install --save-dev ts-jest", or manually run `npm install --save-dev ts-jest`.

- **`Cannot find type definition file for 'jest'`**: The `compilerOptions.types` array in `tsconfig.json` does not include `"jest"`. Confirm that `@types/jest` is installed and add `"types": ["jest"]` to tsconfig.json.

---

## Key Takeaways
- The native installer is the only method that supports automatic updates; prefer it for day-to-day development
- Do not use sudo to install any npm global package; version conflicts and permission issues are difficult to reverse
- `claude doctor` is the standard tool for verifying environment integrity; run it first when problems arise
- Claude Code does not require an IDE; the terminal is a complete working environment
- After Claude finishes execution, immediately verify results with `npm test` and `npx tsc`; do not assume it is always correct

---

## Self-Check
- Run `claude doctor`. Do all items show as passed? If any item fails, do you know how to fix it based on its suggestions?
- Run `npm test` in the `ai-dev-assistant` directory. Does the output include `1 passed`? If it fails, which file or configuration does the error message point to?
- Run `npx tsc`. Is there zero error output? If there are errors, are they related to `tsconfig.json` settings?
