# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a collection of reusable AI agent skills, packaged as a Claude Code plugin (`.claude-plugin/plugin.json`). Each skill is a standalone Markdown file (`SKILL.md`) that teaches an AI agent how to perform a specific task via structured instructions.

## Architecture

- `skills/<skill-name>/SKILL.md` - Each skill lives in its own directory with a `SKILL.md` file containing YAML frontmatter (`name`, `description`) and the full skill instructions
- `.claude-plugin/plugin.json` - Plugin manifest declaring the package for Claude Code

## Writing Skills

Use the `skill-creator` skill to write or modify any skills