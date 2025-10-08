# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a personal knowledge base containing markdown documentation across multiple domains: cloud platforms, databases, Docker, infrastructure, Kubernetes, penetration testing, and general development practices. The repository is organized by technology category with nested subdirectories for specific platforms or tools.

## Repository Structure

The knowledge base is organized into the following top-level categories:

- `Cloud/` - Cloud provider documentation (Azure, GCP, OCI)
- `DB/` - Database documentation (PostgreSQL, MSSQL)
- `Docker/` - Docker and containerization documentation
- `DotNet/` - .NET framework documentation
- `Git/` - Git workflow and configuration
- `Infra/` - Infrastructure documentation
  - `Linux/` - Linux system administration (SELinux, firewalld, networking, storage, services)
  - `Proxmox/` - Proxmox virtualization platform (authentication, cloud-init, setup)
- `k8s/` - Kubernetes documentation and troubleshooting
- `Pentesting/` - Penetration testing techniques
- `Powershell/` - PowerShell scripting

## Working with Documentation

### Adding New Documentation

- Create markdown files in the appropriate category directory
- Use descriptive filenames that match the content topic
- Follow existing documentation structure and formatting
- Include code examples with language identifiers for syntax highlighting
- Add practical command examples where applicable
- Focus on solutions to specific problems rather than general overviews

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
