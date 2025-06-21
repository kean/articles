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
- `tags`: Array of tags (e.g., ["ios", "swift"])
- `permalink`: URL path (e.g., "/post/nuke-docs")
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