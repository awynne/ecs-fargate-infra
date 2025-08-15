# Documentation

This directory contains all project documentation for the ECS Fargate Infrastructure project.

## Structure

- **[technical-specs/](technical-specs/)** - Technical specifications and feature documentation
- **[research/](research/)** - Research findings, spikes, and evaluation results  
- **[architecture/](architecture/)** - System architecture, diagrams, and design documents
- **[ADR/](ADR/)** - Architecture Decision Records
- **[runbooks/](runbooks/)** - Operational procedures and troubleshooting guides

## Documentation Guidelines

- Use markdown for all documentation
- Link between pages using relative markdown links
- Keep filenames consistent (use hyphens, no spaces)
- Include README.md as index files for directories
- Store images in `assets/images/` subdirectories

## Linking Examples

```markdown
<!-- Link to other pages -->
[Getting Started](technical-specs/getting-started.md)
[Architecture Overview](architecture/README.md)
[Deployment Guide](runbooks/deployment.md)

<!-- Link to sections within pages -->
[Authentication](technical-specs/authentication.md#bearer-tokens)

<!-- Images -->
![Architecture Diagram](assets/images/architecture.png)
```