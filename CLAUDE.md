# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a collection of blog articles for kean.blog, written in Markdown format. Each article file follows the naming convention `YYYY-MM-DD-article-slug.markdown`.

## Article Structure

All articles use Jekyll-style YAML front matter with the following fields:
- `layout: post` (standard for all posts)
- `title`: The article title
- `subtitle`: Optional subtitle
- `description`: Meta description for SEO
- `date`: Publication date in ISO format (e.g., "2025-01-10 09:00:00 +0900")
- `category`: Primary category (e.g., "programming")
- `tags`: YAML list of 1–4 topics (e.g., `[swiftui, macos]`), never a comma-separated string, which Jekyll splits on spaces. Each tag must match a file in the site repo's `_topics/`: `ai`, `architecture`, `computer-science`, `concurrency`, `ios`, `macos`, `networking`, `nuke`, `performance`, `persistence`, `pulse`, `rxswift`, `swift`, `swiftui`, `testing`, `tools`, `uikit`
- `series`: Optional, for parts of a multi-part series (`regex`, `vpn`, `pulse`), defined in the site repo's `_data/series.yml`. Parts are ordered by date
- `permalink`: URL path (e.g., "/post/nuke-docs")
- `slug`: Only when the filename slug differs from the last part of the `permalink`; set it to that last part (e.g., `learn-ios` for `/post/learn-ios`). The feed builds each entry's ID from the slug, so a mismatch gives the entry an ID that isn't its URL
- `uuid`: Unique identifier
- `mastodon`: Optional Mastodon post URL

## Common Tasks

### Creating a New Article
1. Create a new file with the naming pattern: `YYYY-MM-DD-article-slug.markdown`
2. Add the required front matter
3. Write content in Markdown format

### Working with Articles
- Articles are standalone Markdown files with no build process in this repository
- The files are likely processed by an external static site generator for deployment
- Focus on content editing and maintaining consistent front matter structure

## Git Workflow
- Current branch: main
- Commit messages should be concise and describe the content changes (e.g., "Add new post about X", "Update Y article", "Fix typo in Z")