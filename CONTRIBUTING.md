# Contributing

Thank you for your interest in this project! This is a personal infrastructure repository shared for educational purposes.

## Ways to Contribute

### Reporting Issues
- Use GitHub Issues for bugs or questions
- Include relevant configuration (sanitized of secrets)
- Describe expected vs. actual behavior

### Suggesting Improvements
- Open an issue to discuss before submitting PRs
- Focus on security, reliability, or documentation improvements

### Pull Requests
1. Fork the repository
2. Create a feature branch (`git checkout -b feature/improvement`)
3. Make your changes
4. Ensure no secrets are included
5. Submit a PR with a clear description

## Code Style

- Use consistent YAML formatting (2-space indentation)
- Include comments explaining non-obvious configurations
- Follow existing patterns for Traefik labels and service structure

## Security

- **Never** commit secrets, passwords, or API keys
- Use `.env.example` files for documentation
- Test configurations locally before submitting
