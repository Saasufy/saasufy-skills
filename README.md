# Saasufy Skills
Saasufy AI skills for Claude Code, Claude (web and Desktop) and pi.

Saasufy is a no-code/low-code, real-time data store and application-creation platform, packaged as a tool for your AI agent. Optimized for vibe-coding security and maintainability. Learn more about Saasufy at [saasufy.com](https://saasufy.com/).

## Installing with pi

```bash
pi install git:github.com/Saasufy/saasufy-skills
```

Once installed, pi will discover the skill from `skills/saasufy/SKILL.md` automatically.

## Installing with Claude Code

After launching Claude Code, run these two commands:

```text
/plugin marketplace add Saasufy/saasufy-skills
/plugin install saasufy-skills@saasufy-skills
```

## Installing with Claude (web)

In [claude.ai](https://claude.ai), add this GitHub repository as a plugin marketplace. The skill itself is free; plugins are a Claude feature which requires a paid Claude plan (Pro or above).

1. Open **Customize** in the left sidebar and go to the **Plugins** tab.
2. Click **Add**, then select **Add marketplace**.
3. Choose **Add from a repository** and enter `Saasufy/saasufy-skills`.
4. Go to the **Discover** tab, select **saasufy-skills** and click **Add**.

## Installing with Claude Desktop

In the Claude Desktop app, the steps are the same as on the web:

1. Open **Customize** in the left sidebar and go to the **Plugins** tab.
2. Click **Add**, then select **Add marketplace**.
3. Choose **Add from a repository** and enter `Saasufy/saasufy-skills`.
4. Go to the **Discover** tab, select **saasufy-skills** and click **Add**.

Plugins are saved to your Claude account, so if you have already installed it on the web, it will be available in Claude Desktop too (and vice versa).

## Use the skills

How you get started depends on whether your AI agent is working inside a project directory.

### With a coding harness (Claude Code or pi)

Launch your agent from inside your project directory (an empty directory is fine for a new app), then ask it to set up your Saasufy app:

```text
Please set up this Saasufy app.
```

The agent will walk you through creating your Saasufy API credential, save it in your project directory and perform the first deployment.

### Without a project directory (Claude web or Claude Desktop)

In a regular chat there is no project directory, so there is no existing app for Claude to set up. Instead, start a new chat and describe the app you want to build:

```text
Please use the Saasufy skill to help me build an app. I don't have a project directory, so please guide me through the setup step by step.

Here is what I want to build: <describe your app>
```

Claude will guide you through creating a Saasufy account and API credential, then build the app with you in the chat.

## License

[MIT](LICENSE)
