# Vulnerability Scanner for package.json

Scan for vulnerable package versions in package.json files.

## Input

$ARGUMENTS

## Instructions

Parse the input to extract vulnerability information. The input should contain:
- Package name(s) (e.g., "react", "next")
- Affected versions (e.g., "19.0", "19.1", "19.2", "15.x", "16.x")
- Fixed versions (e.g., "19.0.1", "19.1.2", "15.5.7")
- (Optional) Search path - defaults to current working directory

If no input is provided, ask the user for the vulnerability details.

## Task

1. **Parse vulnerability info**: Extract package names, affected version patterns, and fixed versions from the input.

2. **Search for package.json files**: Search recursively from the specified path or current working directory.

   Exclude:
   - `node_modules` directories
   - `Library` directories (Unity packages, etc.)
   - `.git` directories
   - Build output directories (`dist`, `build`, `.next`, etc.)

3. **Check each package.json**: For each found package.json:
   - Parse the dependencies and devDependencies
   - Check if any match the vulnerable package names
   - Compare versions against the affected version patterns
   - Consider semver ranges (^, ~, >=, etc.)

4. **Generate report**: Output a markdown table with:
   - File path
   - Package name
   - Current version
   - Required fix version
   - Status (Vulnerable / Safe / Unknown)

5. **Provide fix commands**: Suggest appropriate update commands for each package manager (npm, yarn, bun, pnpm).

## Example Input

```
Package: react
Affected: 19.0, 19.1, 19.2
Fixed: 19.0.1, 19.1.2, 19.2.1

Package: next
Affected: 14.3.0-canary, 15.0-15.5.6, 16.0-16.0.6
Fixed: 15.0.5, 15.1.9, 15.2.6, 15.3.6, 15.4.8, 15.5.7, 16.0.7

Path: /path/to/search
```

## Output Format

### Scan Results

| Path | Package | Version | Fix Version | Status |
|------|---------|---------|-------------|--------|
| /path/to/package.json | next | 15.2.3 | 15.2.6 | Vulnerable |

### Recommended Actions

```bash
# For npm
npm update <package>

# For yarn
yarn upgrade <package>

# For bun
bun update <package>

# For pnpm
pnpm update <package>
```
