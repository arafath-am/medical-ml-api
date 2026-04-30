# Contributing to Medical ML API

Thank you for your interest in contributing to this project! While this is primarily a personal portfolio project, contributions, suggestions, and feedback are welcome.

## How to Contribute

### Reporting Issues

If you find a bug or have a feature suggestion:

1. **Check existing issues** to avoid duplicates
2. **Create a new issue** with a clear title and description
3. Include:
   - Your environment (OS, Python version, GPU model)
   - Steps to reproduce (for bugs)
   - Expected vs. actual behavior
   - Relevant logs or error messages

### Suggesting Enhancements

Enhancement suggestions are welcome! Please:
- Explain the use case
- Describe the proposed solution
- Consider backward compatibility
- Mention any alternatives you've considered

### Pull Requests

If you'd like to submit code changes:

1. **Fork the repository**
2. **Create a feature branch** (`git checkout -b feature/your-feature-name`)
3. **Make your changes** following the code style below
4. **Test your changes** thoroughly
5. **Commit with clear messages** (see commit guidelines below)
6. **Push to your fork** (`git push origin feature/your-feature-name`)
7. **Open a Pull Request** with a clear description

## Development Setup

Follow the [SETUP.md](SETUP.md) guide to get the development environment running.

Additional dev dependencies:
```bash
pip install black flake8 pytest
```

## Code Style

### Python
- Follow [PEP 8](https://pep8.org/)
- Use `black` for formatting: `black *.py`
- Use type hints where applicable
- Maximum line length: 120 characters
- Docstrings for all public functions/classes

Example:
```python
from typing import Optional

def process_audio(
    audio_path: str, 
    max_tokens: int = 256
) -> Optional[str]:
    """
    Process audio file and return transcription.
    
    Args:
        audio_path: Path to audio file
        max_tokens: Maximum tokens for response
        
    Returns:
        Transcribed text or None if error
    """
    # Implementation here
    pass
```

### Naming Conventions
- Variables/functions: `snake_case`
- Classes: `PascalCase`
- Constants: `UPPER_SNAKE_CASE`
- Private methods: `_leading_underscore`

### Comments
- Use comments to explain **why**, not **what**
- Keep comments up-to-date with code changes
- Prefer self-documenting code over comments when possible

## Commit Message Guidelines

Use clear, descriptive commit messages:

```
type: brief description (50 chars or less)

More detailed explanation if needed. Explain what and why,
not how (the code shows how).

- Bullet points for multiple changes
- Reference issues: Fixes #123
```

**Types:**
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation changes
- `refactor`: Code refactoring
- `perf`: Performance improvements
- `test`: Adding/updating tests
- `chore`: Maintenance tasks

**Examples:**
```
feat: add support for WebSocket streaming

fix: resolve GPU memory leak in model cleanup
Fixes #45

docs: update installation instructions for Ubuntu 24.04

refactor: simplify authentication middleware
```

## Testing

Before submitting a PR:

1. **Test manually** with sample audio files
2. **Verify GPU memory management** works correctly
3. **Check error handling** for edge cases
4. **Test with different audio formats**
5. **Ensure API documentation updates** if endpoints changed

### Running Tests
```bash
# Run all tests
pytest

# Run specific test file
pytest test_model_manager.py

# With coverage
pytest --cov=.
```

## Documentation

If your changes affect user-facing features:

- Update `README.md`
- Add examples to `EXAMPLES.md`
- Update `SETUP.md` if installation/deployment changes
- Update API docs (FastAPI auto-generates, but verify)

## Performance Considerations

When adding features:
- **Monitor GPU memory usage** - profile with `nvidia-smi`
- **Test with large files** - ensure chunking works
- **Consider concurrent requests** - check for race conditions
- **Measure latency** - new features shouldn't significantly slow requests

## Security Considerations

- **Never commit API keys** or credentials
- **Validate all user inputs** to prevent injection attacks
- **Sanitize file uploads** to prevent malicious files
- **Log security events** for audit trails
- **Follow OWASP guidelines** for web API security

## Questions?

If you have questions about contributing:
- Open an issue with the `question` label
- Reach out via [GitHub Discussions](https://github.com/arafath-am/medical-ml-api/discussions)
- Check existing issues and discussions first

## License

By contributing, you agree that your contributions will be licensed under the MIT License.

---

Thank you for contributing to Medical ML API! 🎉
