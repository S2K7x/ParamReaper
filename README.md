## README.md for ParamReaper

```markdown
# ParamReaper

**ParamReaper** is a powerful, multi-threaded web application parameter enumeration tool built for bug bounty hunters and security researchers. It crawls websites, extracts parameters from URLs, forms, and JavaScript, guesses hidden parameters, and identifies API endpoints—all while respecting scope and rate limits. Whether you're hunting for XSS, SQLi, or SSRF vulnerabilities, ParamReaper is your go-to tool for uncovering the hidden variables that matter.

## Features

- **Comprehensive Parameter Discovery**: Extracts parameters from URLs, HTML forms (GET/POST), and JavaScript files.
- **Parameter Guessing**: Tests common parameter names to find hidden or undocumented inputs.
- **API Endpoint Detection**: Identifies potential API endpoints using patterns and content types.
- **JSON Support**: Handles JSON-based POST requests for modern APIs.
- **Advanced JS Parsing**: Uses `esprima` to analyze JavaScript code for parameters and endpoints.
- **Multithreading**: Speeds up enumeration with configurable concurrent threads.
- **Scope Control**: Limits enumeration to specific domains or paths with the `--scope` option.
- **Rate Limiting**: Adds delays between requests to stay stealthy and respectful.
- **Command-Line Interface**: Flexible configuration via CLI arguments.

## Installation

### Prerequisites
- Python 3.6+
- Required libraries: `requests`, `beautifulsoup4`, `esprima`

### Setup
1. Clone the repository:
   ```bash
   git clone https://github.com/yourusername/ParamReaper.git
   cd ParamReaper
   ```
2. Install dependencies:
   ```bash
   pip install requests beautifulsoup4 esprima
   ```
3. Run the tool:
   ```bash
   python paramreaper.py --help
   ```

## Usage

### Basic Command
Crawl a target website with default settings:
```bash
python paramreaper.py https://example.com
```

### Advanced Examples
- **With Authentication and Scope**:
  ```bash
  python paramreaper.py https://example.com --cookies '{"session": "abc123"}' --headers '{"Authorization": "Bearer xyz"}' --scope "example.com,/api"
  ```
- **Custom Depth and Threads**:
  ```bash
  python paramreaper.py https://target.com --depth 3 --threads 10 --delay 1
  ```

### Command-Line Options
| Option       | Description                              | Default       |
|--------------|------------------------------------------|---------------|
| `url`        | Target URL to enumerate                 | (Required)    |
| `--depth`    | Max crawl depth                         | 2             |
| `--delay`    | Delay between requests (seconds)        | 0.5           |
| `--threads`  | Number of concurrent threads            | 5             |
| `--scope`    | Limit to domains/paths (e.g., "example.com,/api") | None |
| `--cookies`  | Cookies in JSON format                  | "{}"          |
| `--headers`  | Headers in JSON format                  | "{}"          |

### Output
Results are saved to `parameters.txt` in the working directory, categorized as:
- **Extracted Parameters**: Found in URLs, forms, or JavaScript.
- **Guessed Parameters**: Potentially accepted by the server.
- **API Endpoints**: Likely API URLs.

Example `parameters.txt`:
```
Extracted Parameters:
==================================================
Parameter: id
Found in URLs: https://example.com/page?id=123
--------------------------------------------------

Guessed Parameters:
==================================================
Parameter: token
Potentially used in URLs: https://example.com/page
--------------------------------------------------

API Endpoints:
==================================================
https://example.com/api/users
```

## How It Works

1. **Crawling**: Uses a multi-threaded crawler to explore the target up to the specified depth.
2. **Extraction**: Parses URLs, forms (GET/POST), and JavaScript for parameters.
3. **Guessing**: Tests common parameters with unique values, checking for server reflection.
4. **API Detection**: Identifies endpoints via patterns, content types, and JS analysis.
5. **Output**: Aggregates findings into a structured text file.

## Tips for Bug Bounty Hunters

- **Authentication**: Use `--cookies` and `--headers` to access authenticated areas.
- **Scope**: Define `--scope` to stay within program boundaries.
- **Stealth**: Increase `--delay` to avoid detection.
- **APIs**: Focus on `API Endpoints` section for high-value targets.

## Limitations

- JavaScript parsing with `esprima` may miss dynamically constructed parameters.
- JSON POST support assumes API endpoints return text reflections for guessing.
- Multithreading efficiency depends on target server capacity and `--delay`.

## Contributing

Contributions are welcome! Please:
1. Fork the repository.
2. Create a feature branch (`git checkout -b feature/awesome-idea`).
3. Commit your changes (`git commit -m "Add awesome idea"`).
4. Push to the branch (`git push origin feature/awesome-idea`).
5. Open a pull request.

## Legal Notice

ParamReaper is intended for ethical security research and bug bounty programs. Only use it on targets where you have explicit permission. Respect all applicable laws and program policies.

## Acknowledgments

- Built with ❤️ by [Your Name] for the bug bounty community.
- Powered by `requests`, `BeautifulSoup`, and `esprima`.

## License

MIT License - see [LICENSE](LICENSE) for details.
```

---

## Naming Rationale

**"ParamReaper"**:
- **"Param"**: Short for "parameters," the tool’s primary focus.
- **"Reaper"**: Suggests harvesting or collecting with precision and efficiency, aligning with the tool’s purpose of gathering actionable data for security testing.
- Cool factor: It’s edgy, memorable, and fits the hacker aesthetic popular in the bug bounty world.

## README Highlights

- **Clear Structure**: Sections for installation, usage, features, and more.
- **Practical Examples**: Shows basic and advanced commands.
- **Bug Bounty Focus**: Tips and legal notice tailored for hunters.
- **Professional Tone**: Balanced with a touch of personality (e.g., "Built with ❤️").
