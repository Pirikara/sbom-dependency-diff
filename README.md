# SBOM Dependency Diff

A GitHub Action that generates SBOMs (Software Bill of Materials) for base and head refs using [Syft](https://github.com/anchore/syft) and outputs dependency differences (added/removed/changed) as JSON.

## Features

- **SBOM generation with Syft** - Eliminates the need for ecosystem-specific handling
- **CycloneDX JSON format** - Uses CycloneDX as the default SBOM format
- **Composite Action** - No Dockerfile required, runs directly on GitHub Actions runners
- **JSON outputs** - Easy to consume in subsequent workflow steps using `jq` or scripts
- **Focused responsibility** - Only detects and returns dependency differences; vulnerability scanning, license checks, etc. can be implemented in subsequent workflow steps

## What This Action Does

This action:

1. Generates SBOMs for both base and head refs using Syft
2. Compares the SBOMs using CycloneDX CLI
3. Outputs the differences as JSON:
   - `added` - Array of newly added dependencies
   - `removed` - Array of removed dependencies
   - `changed` - Array of dependencies with version changes
4. Provides a `has_diff` boolean flag
5. Optionally fails the job if differences are detected (via `fail-on-diff` input)

## What This Action Does NOT Do

This action is focused solely on **detecting and returning dependency differences**. It does NOT:

- Perform vulnerability scanning (e.g., OSV)
- Check package release dates
- Validate license policies
- Output dependency tree structures

These features should be implemented in subsequent workflow steps using the JSON outputs from this action.

## Usage

### Basic Example

```yaml
name: dependency-diff

on:
  pull_request:

jobs:
  diff:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Required for comparing refs

      - name: SBOM dependency diff
        id: diff
        uses: tomoyayamashita/sbom-dependency-diff@v1
        with:
          path: .
          sbom-format: cyclonedx-json

      - name: Show added dependencies
        run: |
          echo '${{ steps.diff.outputs.added }}' | jq .
```

### Fail on Any Dependency Changes

```yaml
      - name: SBOM dependency diff
        id: diff
        uses: tomoyayamashita/sbom-dependency-diff@v1
        with:
          path: .
          sbom-format: cyclonedx-json
          fail-on-diff: "true"
```

### Advanced Example with Vulnerability Scanning

```yaml
      - name: SBOM dependency diff
        id: diff
        uses: tomoyayamashita/sbom-dependency-diff@v1

      - name: Check added dependencies for vulnerabilities
        if: steps.diff.outputs.has_diff == 'true'
        run: |
          # Example: Use OSV scanner or other tools on added dependencies
          echo '${{ steps.diff.outputs.added }}' | jq -r '.[].purl' | while read purl; do
            echo "Checking $purl for vulnerabilities..."
            # Your vulnerability scanning logic here
          done
```

## Inputs

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `base-ref` | string | No | `GITHUB_BASE_REF` or `origin/main` | Base git ref to compare (e.g., PR base branch) |
| `head-ref` | string | No | `GITHUB_HEAD_REF` or `HEAD` | Head git ref to compare (e.g., PR head branch) |
| `path` | string | No | `"."` | Target path to scan for SBOM generation (useful for monorepos) |
| `sbom-format` | string | No | `"cyclonedx-json"` | Syft SBOM output format (currently only CycloneDX JSON is supported) |
| `syft-args` | string | No | `""` | Additional arguments to pass to Syft (e.g., for image scanning) |
| `working-directory` | string | No | `"."` | Working directory for git operations and Syft execution |
| `fail-on-diff` | string | No | `"false"` | If `"true"`, the action will fail when any dependency diff is detected |

### Default Behavior for Refs

- `base-ref` (if not specified):
  - Uses `GITHUB_BASE_REF` environment variable (in PR context)
  - Falls back to `origin/main` if not in PR context

- `head-ref` (if not specified):
  - Uses `GITHUB_HEAD_REF` environment variable (in PR context)
  - Falls back to `HEAD` if not in PR context

## Outputs

All outputs are **JSON strings**.

| Name | Type | Description |
|------|------|-------------|
| `has_diff` | boolean (as string) | `"true"` if there are any dependency differences, `"false"` otherwise |
| `added` | JSON array (as string) | Array of added dependency components |
| `removed` | JSON array (as string) | Array of removed dependency components |
| `changed` | JSON array (as string) | Array of dependency components with version changes |

### Output Format Examples

#### `added` / `removed` format

```json
[
  {
    "name": "lodash",
    "version": "4.17.21",
    "purl": "pkg:npm/lodash@4.17.21",
    "bom_ref": "pkg:npm/lodash@4.17.21",
    "type": "library"
  }
]
```

#### `changed` format

```json
[
  {
    "name": "lodash",
    "from_version": "4.17.20",
    "to_version": "4.17.21",
    "from_purl": "pkg:npm/lodash@4.17.20",
    "to_purl": "pkg:npm/lodash@4.17.21",
    "bom_ref": "pkg:npm/lodash@4.17.21"
  }
]
```

## Important Notes

### Checkout Requirements

This action requires access to git history to compare different refs. Therefore:

- **Always use `actions/checkout@v4` with `fetch-depth: 0`**

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0  # Required!
```

### CycloneDX CLI Dependency

The JSON normalization logic depends on the output structure of the CycloneDX CLI `diff` command. The current implementation assumes a specific structure, but this may need adjustment based on the actual CycloneDX CLI version and output format.

If you encounter issues with the JSON parsing, please check:
1. The CycloneDX CLI version being installed
2. The actual structure of the diff JSON output
3. The `jq` processing logic in the "Normalize diff JSON" step of `action.yml`

## Development

### Testing

To test this action locally or in a real repository:

1. Create a PR with dependency changes:
   - Add a new dependency
   - Remove an existing dependency
   - Update a dependency version

2. Run the action in the PR workflow

3. Verify the outputs:
   - Check `added`, `removed`, `changed` arrays
   - Verify `has_diff` is set correctly

### TODO

- [ ] Verify the actual CycloneDX CLI diff JSON output structure
- [ ] Adjust `jq` processing in the normalize step to match real output
- [ ] Add support for additional SBOM formats if needed
- [ ] Consider adding examples for common ecosystems (npm, Go, Python, etc.)

## License

MIT License - See [LICENSE](LICENSE) file for details

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## Related Projects

- [Syft](https://github.com/anchore/syft) - CLI tool for generating SBOMs
- [CycloneDX CLI](https://github.com/CycloneDX/cyclonedx-cli) - CycloneDX SBOM manipulation tool
- [CycloneDX](https://cyclonedx.org/) - SBOM standard

## Author

Pirikara
