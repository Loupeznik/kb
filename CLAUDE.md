# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a personal knowledge base containing markdown documentation across multiple domains: cloud platforms, databases, Docker, infrastructure, Kubernetes, penetration testing, and general development practices. The repository is organized by technology category with nested subdirectories for specific platforms or tools.

## Repository Structure

The knowledge base documentation is organized within the Docusaurus website:

- `website/` - Docusaurus website for hosting the knowledge base
  - `website/docs/Cloud/` - Cloud provider documentation (Azure, GCP, OCI)
  - `website/docs/DB/` - Database documentation (PostgreSQL, MSSQL)
  - `website/docs/DevOps/` - DevOps practices and CI/CD configurations
  - `website/docs/Docker/` - Docker and containerization documentation
  - `website/docs/DotNet/` - .NET framework documentation
  - `website/docs/Git/` - Git workflow and configuration
  - `website/docs/Infra/` - Infrastructure documentation
    - `website/docs/Infra/Linux/` - Linux system administration (SELinux, firewalld, networking, storage, services)
    - `website/docs/Infra/Proxmox/` - Proxmox virtualization platform (authentication, cloud-init, setup)
  - `website/docs/k8s/` - Kubernetes documentation and troubleshooting
  - `website/docs/Pentesting/` - Penetration testing techniques
  - `website/docs/Powershell/` - PowerShell scripting

## Docusaurus Website

The knowledge base is published as a static website using Docusaurus v3 with TypeScript. The website is automatically deployed to GitHub Pages at `https://loupeznik.github.io/kb/` when changes are pushed to the `master` branch.

### Website Structure

- `website/docs/` - All markdown documentation (copied from root-level category folders)
- `website/src/` - React components and pages
- `website/static/` - Static assets (images, favicon, etc.)
- `website/docusaurus.config.ts` - Main configuration file
- `website/sidebars.ts` - Sidebar navigation configuration
- `website/biome.json` - Code quality and formatting configuration

### Prerequisites

- Node.js 22.0 or higher (preferably 24.0+)
- npm package manager

### Local Development

```bash
# Start development server
npm start --prefix website

# Build for production
npm run build --prefix website

# Serve production build locally
npm run serve --prefix website
```

The development server runs at `http://localhost:3000/kb/`

### Code Quality Checks

The website project uses TypeScript and Biome for code quality and formatting.

#### Run TypeScript type checking

```bash
npm run typecheck --prefix website
```

#### Run Biome formatting

```bash
npm run format --prefix website
```

#### Run Biome linting

```bash
npm run lint --prefix website
```

#### Run all Biome checks (format + lint)

```bash
npm run check --prefix website
```

### Biome Configuration

The project uses the following Biome settings:
- **Indentation**: Tabs
- **Line width**: 100 characters
- **Quote style**: Single quotes
- **Trailing commas**: Always

### Adding Documentation to Website

When adding new documentation:

1. Add markdown files directly to the appropriate category in `website/docs/` directory (e.g., `website/docs/Cloud/`, `website/docs/DB/`, etc.)
2. Add front matter if needed for ordering:
   ```markdown
   ---
   title: Document Title
   sidebar_position: 1
   ---
   ```
3. For new categories, create a `_category_.json` file:
   ```json
   {
     "label": "Category Name",
     "position": 1,
     "collapsed": false
   }
   ```
4. The sidebar navigation will automatically update based on the file structure

### Deployment

The website is automatically deployed via GitHub Actions when changes are pushed to `master`:

1. Workflow file: `.github/workflows/deploy-docs.yml`
2. Triggered on push to `master` when `website/**` files change
3. Builds static site and deploys to GitHub Pages
4. Requires GitHub Pages source to be set to "GitHub Actions" in repository settings

### MDX Compatibility

Docusaurus uses MDX for markdown processing. Be aware that:
- Angle brackets (`<` and `>`) in text are interpreted as JSX/HTML tags
- Use backticks for placeholders: `` `<PLACEHOLDER>` `` instead of `<PLACEHOLDER>`
- Wrap angle brackets in code blocks when showing command examples

## Working with Documentation

### Adding New Documentation

- Create markdown files in the appropriate category directory within `website/docs/`
- Use descriptive filenames that match the content topic
- Follow existing documentation structure and formatting
- Include code examples with language identifiers for syntax highlighting
- Add practical command examples where applicable
- Focus on solutions to specific problems rather than general overviews
- Ensure placeholders with angle brackets are wrapped in backticks for MDX compatibility

### Documentation Style

- Use clear section headers with markdown heading syntax
- Include symptoms/problem descriptions before solutions
- Provide complete, runnable command examples
- Use code blocks with appropriate language tags (```bash, ```sql, ```yaml, etc.)
- Include context about when to use specific solutions
- Document edge cases and troubleshooting steps

### Common Documentation Patterns

1. **Troubleshooting guides**: Problem description → Symptoms → Solution → Verification
2. **Configuration guides**: Context → Configuration steps → Verification
3. **Command references**: Description → Example commands → Options/variations

## Version Control

This repository uses git with `master` as the default branch.
