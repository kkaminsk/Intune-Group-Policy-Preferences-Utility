# Intune Group Policy Preferences Utility

## Overview
This utility enables you to export Group Policy Preferences (GPP) from Active Directory Group Policy Objects and apply them on devices that don't have traditional GPO access, such as:
- Azure AD/Entra-joined devices managed by Intune
- Workgroup computers
- Standalone systems outside your domain

The tool converts GPP settings into JSON format, which can then be processed locally using PowerShell to replicate the original GPP behavior.

## How It Works
1. **Export Phase**: Run `Convert-GPPtoJSON.ps1` on a domain-joined machine with Group Policy access to export GPP settings to JSON files
2. **Deploy Phase**: Distribute the JSON rules files to target devices (e.g., via Intune, SCCM, or other deployment tools)
3. **Apply Phase**: Run `Process-RegistryPreferenceRules.ps1` on target devices to apply the settings locally

## Prerequisites

### For Export (Convert-GPPtoJSON.ps1)
- Windows PowerShell 5.1 or PowerShell 7+
- **GroupPolicy PowerShell module** (included in RSAT tools)
- Domain-joined computer with access to Group Policy
- Permissions to read the target GPO(s)
- Network connectivity to domain controller

### For Processing (Process-RegistryPreferenceRules.ps1)
- Windows PowerShell 5.1 or PowerShell 7+
- **Administrator privileges** (required to modify registry)
- Access to the JSON rules file(s)

## Supported Preference Types

### Currently Supported (for processing)
- **Registry** preferences (fully supported with Create, Update, Replace, and Delete actions)

### Exported but Not Yet Processed
The export script can extract the following preference types, but the processing script currently only applies Registry preferences:
- Folder, File, Group, Environment Variable, NT Service, Shortcut, and Scheduled Task preferences

## Usage

### Step 1: Export GPP to JSON

Run this on a **domain-joined machine** with Group Policy access:

```powershell
# Export a single GPO
.\Admin\Convert-GPPtoJSON.ps1 -GPOName \ My GPO\ -ExportPath \.\Rules\

# Export multiple GPOs
.\Admin\Convert-GPPtoJSON.ps1 -GPOName \GPO1\,\GPO2\,\GPO3\ -ExportPath \.\Rules\

# Using default values (exports \Test GPP\ to .\Rules)
.\Admin\Convert-GPPtoJSON.ps1
```

**Parameters:**
- `GPOName`: Name of the GPO(s) to export (default: \Test GPP\)
- `ExportPath`: Directory where JSON files will be created (default: \.\Rules\)

**Output:**
- Creates one JSON file per GPO in the specified export path
- File naming: `<GPOName>.JSON`
- The export directory will be created if it doesn't exist

### Step 2: Deploy JSON Files

Transfer the generated JSON file(s) from the `Rules` folder to your target devices. Common deployment methods:
- **Intune**: Package as Win32 app or use Proactive Remediations
- **SCCM/Configuration Manager**: Deploy as application or package
- **File share**: Copy to accessible network location
- **Manual**: Copy directly to each device

### Step 3: Process Rules on Target Device

Run this with **administrator privileges** on the target device:

```powershell
# Process a specific rules file
.\Process-RegistryPreferenceRules.ps1 -GPPLogKeyPath \HKLM:\SOFTWARE\MyCompany\GPP\ -Path \C:\Rules\ -RulesFileName \My GPO.JSON\

# Process all JSON files in a directory
.\Process-RegistryPreferenceRules.ps1 -GPPLogKeyPath \HKLM:\SOFTWARE\MyCompany\GPP\ -Path \C:\Rules\

# Using default values
.\Process-RegistryPreferenceRules.ps1
```

**Parameters:**
- `GPPLogKeyPath`: Registry path for tracking applied rules (default: \HKLM:\SOFTWARE\ASD\GPP\)
- `Path`: Directory containing the JSON rules files (default: \.\Rules\)
- `RulesFileName`: Specific JSON file(s) to process (optional - if omitted, all .JSON files in Path are processed)

**What It Does:**
- Reads JSON rules file(s) from the specified path
- Applies registry preferences according to the action specified (Create, Update, Replace, Delete)
- Tracks applied rules in the registry log path to support RunOnce and RemovePolicy features
- Handles error suppression based on BypassErrors flag

## Understanding Registry Actions

The utility supports four action types for registry preferences:

- **C (Create)**: Creates a new registry key or value if it doesn't exist
- **U (Update)**: Updates an existing registry value (creates if doesn't exist)
- **R (Replace)**: Deletes existing value/key and recreates it
- **D (Delete)**: Removes the specified registry key or value

## Key Features

### RunOnce Flag
When `runOnce` is set to `1` in the JSON, the rule will only be applied once. Subsequent executions will skip this rule if it was already successfully applied.

### RemovePolicy Flag
When `removePolicy` is set to `1` in the JSON:
- The rule settings are logged to the registry
- If the rule is later removed from the JSON file, the processing script will automatically reverse/delete the changes
- This mimics GPO behavior where settings are removed when the policy no longer applies

### Bypass Errors
When `bypassErrors` is set to `1`, errors during processing of that specific rule will be suppressed and won't stop the overall processing.

## Logging and Tracking

The utility creates a registry key at the path specified by `GPPLogKeyPath` (default: `HKLM:\SOFTWARE\ASD\GPP`) to track:
- **Applied Rules**: Timestamps for when each rule (identified by GUID) was last applied
- **RemovePolicy Blobs**: Serialized rule information for rules that need to be reversed when no longer applied

This tracking enables:
- RunOnce functionality (rules applied only once)
- Automatic cleanup when policies are removed
- Audit trail of when rules were applied

## Troubleshooting

### Export Issues

**Problem**: \Cannot find GPO\ error when running Convert-GPPtoJSON
- **Solution**: Verify the GPO name is correct (case-sensitive)
- **Solution**: Ensure you're running on a domain-joined machine
- **Solution**: Confirm you have permissions to read the GPO
- **Solution**: Check network connectivity to the domain controller

**Problem**: No JSON file created after export
- **Solution**: Verify the GPO contains Group Policy Preferences (not just regular policies)
- **Solution**: Check that the export path is writable
- **Solution**: Ensure the GroupPolicy PowerShell module is installed (`Get-Module -ListAvailable GroupPolicy`)

**Problem**: JSON file is created but empty or contains only brackets
- **Solution**: The GPO may not contain any supported preference types (currently only Registry preferences are fully supported for processing)

### Processing Issues

**Problem**: \Access Denied\ errors when processing rules
- **Solution**: Run PowerShell as Administrator
- **Solution**: Verify you have permissions to modify the target registry hive (usually requires admin rights for HKLM)

**Problem**: Rules don't seem to apply
- **Solution**: Check if RunOnce is enabled and rule was already applied previously
- **Solution**: Verify the JSON file path is correct and accessible
- **Solution**: Check the registry log path (`GPPLogKeyPath`) for applied rule timestamps
- **Solution**: Manually inspect the target registry location to confirm current state

**Problem**: \Cannot find path\ errors
- **Solution**: Ensure the `Path` parameter points to a valid directory containing JSON files
- **Solution**: If specifying `RulesFileName`, verify the file exists in the specified path
- **Solution**: Check for typos in file names (case-sensitive on some systems)

**Problem**: Changes get reversed unexpectedly
- **Solution**: This is expected behavior when `removePolicy` is set to 1 and the rule is removed from the JSON
- **Solution**: To keep changes permanent, ensure `removePolicy` is set to 0

**Problem**: Script runs but registry values don't match expected values
- **Solution**: Verify the registry value type in the JSON matches the intended type (REG_SZ, REG_DWORD, etc.)
- **Solution**: For REG_DWORD, ensure the value is in the correct format (e.g., \00000001\ for 1)
- **Solution**: Check the action type - Replace (R) behaves differently than Update (U)

### General Troubleshooting Steps

1. **Enable Verbose Output**: Run scripts with `-Verbose` parameter for detailed execution information
2. **Check Event Logs**: Review Windows Event Logs for any related errors
3. **Test with Simple Rule**: Create a minimal JSON with one simple registry entry to isolate issues
4. **Verify JSON Format**: Ensure JSON is valid (use a JSON validator tool)
5. **Review Permissions**: Confirm script execution policy allows running PowerShell scripts (`Get-ExecutionPolicy`)

## Example Workflow

Here's a complete example of using this utility:

```powershell
# 1. On domain-joined machine: Export GPO preferences
.\Admin\Convert-GPPtoJSON.ps1 -GPOName \Corporate Settings\ -ExportPath \C:\GPPExport\

# 2. Copy the generated JSON file to target devices
# (e.g., via Intune package deployment to C:\ProgramData\GPPRules)

# 3. On target device: Apply the preferences (run as Administrator)
.\Process-RegistryPreferenceRules.ps1 -GPPLogKeyPath \HKLM:\SOFTWARE\Company\GPP\ -Path \C:\ProgramData\GPPRules\ -RulesFileName \Corporate Settings.JSON\

# 4. Verify the changes in registry
# Check HKLM:\SOFTWARE paths defined in your JSON rules
```

## Intune Deployment Example

For deploying via Intune as a Win32 app:

1. **Package contents**:
   - `Process-RegistryPreferenceRules.ps1`
   - Your JSON rules file(s)
   - Install script (`install.ps1`):
   
```powershell
# install.ps1
Start-Process powershell.exe -ArgumentList \-ExecutionPolicy Bypass -File .\Process-RegistryPreferenceRules.ps1 -GPPLogKeyPath HKLM:\SOFTWARE\Company\GPP -Path . -RulesFileName MyRules.JSON \ -Wait -NoNewWindow
```

2. **Detection script** (`detect.ps1`):

```powershell
# detect.ps1
 = \HKLM:\SOFTWARE\Company\GPP\
if (Test-Path ) {
     = Get-ItemProperty -Path  -ErrorAction SilentlyContinue
    if () {
        Write-Output \GPP Rules Applied\
        exit 0
    }
}
exit 1
```

3. Package using Intune Win32 Content Prep Tool and deploy

## Best Practices

1. **Test First**: Always test JSON rules on a non-production device before wide deployment
2. **Use Descriptive Names**: Use clear GPO and rule names for easier troubleshooting
3. **Version Control**: Keep JSON files in version control to track changes
4. **Separate Concerns**: Create separate GPO exports for different types of settings
5. **Document Changes**: Use the `desc` field in JSON to document what each rule does
6. **Regular Audits**: Periodically review the log registry path to ensure rules are applying correctly
7. **Cleanup Old Rules**: Remove unused rules from JSON to prevent bloat
8. **Backup**: Keep backups of working JSON configurations
9. **Use Custom Log Paths**: Use company-specific registry paths for logging to avoid conflicts

## Limitations

- Currently only processes **Registry** preferences (other types are exported but not yet processed)
- Filters from GPP (like OS version, organizational unit, etc.) are exported but not currently evaluated during processing
- Scheduled Task triggers are exported but not fully implemented
- Requires manual deployment of JSON files to target devices
- No automatic update mechanism - changes require redeployment

## Contributing

This is a community-driven utility. Contributions to extend support for additional preference types (Files, Folders, Services, etc.) are welcome.

## Credits

Original concept and implementation by the Microsoft/Intune community for managing Group Policy Preferences on non-domain-joined devices.
