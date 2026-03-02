# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a personal resume repository for 陈志达. It contains a Markdown-based resume that is automatically converted to HTML via GitHub Actions. The project generates a polished HTML resume from a simple Markdown source, making it easy to maintain while providing a professional output.

## Architecture

The system consists of three main components:

1. **Source Content**: `chenzhida/resume.md` - The Markdown source of the resume
2. **Template**: `chenzhida/template.html` - HTML template with placeholders for dynamic content
3. **Styles**: `chenzhida/style.css` - Custom styling for the resume
4. **Generator**: GitHub Action (`.github/workflows/update-chenzhida.yml`) - Automatically converts Markdown to HTML

## Build System

The project uses GitHub Actions for automated HTML generation:

- **Trigger**: Automatic on push to `chenzhida/resume.md`, `chenzhida/template.html`, or `chenzhida/style.css`
- **Manual Trigger**: Can be manually run via GitHub Actions UI
- **Process**:
  1. Checks out the repository
  2. Installs Pandoc
  3. Converts `resume.md` to HTML using Pandoc
  4. Injects the converted content into the HTML template
  5. Replaces template placeholders ({{NAME}}, {{TITLE}}, {{EMAIL}}, etc.)
  6. Commits the generated `index.html` back to the repository
  7. Creates a build summary with links to the output

## Key Files

- `chenzhida/resume.md` - The source resume content in Chinese
- `chenzhida/template.html` - HTML template with placeholders for personal information
- `chenzhida/style.css` - Custom CSS styling for the resume layout
- `chenzhida/index.html` - Generated HTML output (committed automatically)

## Template Placeholders

The HTML template uses the following placeholders that get automatically replaced:
- `{{NAME}}` - Full name
- `{{TITLE}}` - Professional title
- `{{EMAIL}}` - Email address
- `{{GITHUB}}` - GitHub profile URL
- `{{GITHUB_USER}}` - GitHub username
- `{{LOCATION}}` - Location
- `{{PHONE}}` - Phone number
- `{{UPDATE_TIME}}` - Last update timestamp
- `{{RESUME_CONTENT}}` - The main resume content from the converted Markdown

## Development Workflow

1. Edit `chenzhida/resume.md` for content updates
2. The GitHub Action automatically handles HTML generation and deployment
3. No manual build or deployment needed
4. Output is committed back to the repository and viewable on GitHub Pages

## Notes

- The project follows a simple "write in Markdown, get HTML" workflow
- All styling is handled by `style.css` plus GitHub Markdown CSS
- The automated system ensures the resume is always up-to-date
- Generated HTML includes metadata for better search engine optimization