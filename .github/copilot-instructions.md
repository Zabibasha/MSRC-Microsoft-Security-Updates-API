# Copilot Instructions for MSRC Security Updates API

## Project Overview
This is a **PowerShell module** (v1.9.9) for accessing Microsoft Security Response Center (MSRC) security update details via the CVRF (Common Vulnerability Reporting Format) API. The module is published on PowerShell Gallery and provides cmdlets for querying security updates, CVEs, and generating HTML reports.

## Architecture & Key Patterns

### Module Structure
- **Root**: [MsrcSecurityUpdates.psm1](src/MsrcSecurityUpdates/MsrcSecurityUpdates.psm1) - dot-sources all Public and Private functions
- **Public/**: User-facing cmdlets that export via `Export-ModuleMember`
- **Private/**: Internal helper functions not exposed to module consumers
- **Manifest**: [MsrcSecurityUpdates.psd1](src/MsrcSecurityUpdates/MsrcSecurityUpdates.psd1) - PowerShell v5.1+ required

### Global Variables Pattern
The module uses **shared global variables** initialized by `Set-GlobalVariables.ps1`:
- `$global:msrcApiUrl` - API endpoint (default: `https://api.msrc.microsoft.com/cvrf/v3.0`)
- `$global:msrcApiVersion` - Version header (default: `api-version=2023-11-01`)
- `$global:msrcProxy` / `$global:msrcProxyCredential` - Optional proxy configuration

These globals are set at module load and can be reconfigured via `Set-MSRCApiKey -APIVersion` to switch between API v2.0 and v3.0.

### API Integration
- No API key required for public endpoints
- Uses REST method with dynamic Accept headers (JSON/XML)
- All functions fetch from `https://api.msrc.microsoft.com/cvrf/` endpoints
- **See**: [Get-MsrcCvrfDocument.ps1](src/MsrcSecurityUpdates/Public/Get-MsrcCvrfDocument.ps1) for REST pattern

### Dynamic Parameters Pattern
Functions like `Get-MsrcCvrfDocument` use **DynamicParam** blocks to:
- Fetch valid CVRF IDs from API at runtime via `Get-CVRFID`
- Auto-validate user input against live list (e.g., '2017-Apr', '2017-Mar')
- Throw error if API is unreachable

## Key Workflows

### Testing
- **File**: [MsrcSecurityUpdates.tests.ps1](src/MsrcSecurityUpdates/MsrcSecurityUpdates.tests.ps1)
- Tests use **Pester framework** (DescribeIt-Based)
- Tests dynamically call both API v2.0 and v3.0 to verify compatibility
- Run via: `Invoke-Pester MsrcSecurityUpdates.tests.ps1`
- No external test runner; PowerShell native testing

### Building
- Use `msbuild /t:build` for project compilation (see workspace task)
- Module distribution via PowerShell Gallery (publish step not in repo)

## Project-Specific Conventions

### Function Naming
- Public: `Get-Msrc*`, `Set-Msrc*` (verb-noun with MSRC prefix)
- Private: Helper functions without export prefix convention
- **Example**: `Get-MsrcCvrfDocument` (public), `Get-CVRFID` (private)

### Error Handling
- Functions use `-ErrorAction Stop` in REST calls
- Throw custom messages for API connectivity issues
- No try-catch blocks visible; rely on caller error handling

### Data Format Transformation
- CVRF documents can be returned as JSON (default) or XML (`-AsXml` flag)
- Private functions like `Get-MsrcCvrfAffectedSoftware` transform CVRF data for reporting
- JSON/XML parsing handled transparently via Accept headers

### JSON Data Files
- [CVSS-Descriptions.json](src/MsrcSecurityUpdates/Public/CVSS-Descriptions.json) - Metadata for CVSS severity levels
- [CVSS-Metrics.json](src/MsrcSecurityUpdates/Public/CVSS-Metrics.json) - CVSS scoring reference
- Used by functions that generate HTML reports

## Integration Points

### External Dependencies
- MSRC CVRF API (public, no credentials required)
- PowerShell 5.1+ runtime
- Optional: Proxy support (credentials stored in global variables)

### Cross-Component Communication
- All public functions depend on global variables set by module initialization
- Private functions (`Get-CVRFID`, threat status helpers) called by public cmdlets
- Data flows: API → JSON/XML parsing → Object transformation → HTML report generation

## Development Guidelines

1. **Respect the global variable pattern** - Don't hardcode API URLs; reference `$global:msrcApiUrl` and `$global:msrcApiVersion`
2. **Proxy support** - Always check for `$global:msrcProxy` and `$global:msrcProxyCredential` before REST calls (see pattern in Get-MsrcCvrfDocument)
3. **API version compatibility** - Ensure changes work with both v2.0 and v3.0 API versions; tests validate both
4. **Dynamic parameter validation** - Use `Get-CVRFID` for parameter validation against live API data
5. **Test both code paths** - Update tests to cover new v2.0 and v3.0 scenarios if adding API version branching logic
