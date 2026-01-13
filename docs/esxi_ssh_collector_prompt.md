# ESXi SSH Data Collection Script

Create a PowerShell 7+ script that collects hardware and configuration data from ESXi hosts via SSH.

## Input

- A newline-separated text file containing ESXi hostnames or IP addresses (one per line)
- Credentials prompted at runtime (single username/password for all hosts)

## Workflow per host

1. Connect to the ESXi host via PowerCLI (`Connect-VIServer`)
2. Record whether SSH service is currently enabled
3. Enable the SSH service via `Get-VMHostService` / `Start-VMHostService`
4. Execute the following commands via the Windows built-in SSH client (`ssh.exe`):
   - `vmware -vl`
   - `esxcli network nic list`
   - `lspci -v |grep -i eth`
   - `lspci -v |grep -i net`
   - `lspci -p |grep -i qedentv`
   - `vsish -e get /hardware/bios/dmiInfo`
   - `vsish -e get /hardware/cpu/cpuModelName`
   - `vsish -e get /hardware/cpu/cpuInfo`
   - `esxcfg-scsidevs -a`
   - `esxcfg-scsidevs -A`
   - `esxcfg-scsidevs -c`
5. Disable SSH service (default behavior)
6. Disconnect from the host

## Output

- Single CSV file with proper RFC 4180 quoting (handles embedded commas, newlines, quotes)
- Column headers: `Hostname` followed by each command as its own column header (the literal command string)
- Each row contains one host's data with full command output per cell
- Filename auto-generated with timestamp (e.g., `esxi_inventory_20250113_143022.csv`)

## Logging

- Log all failures (connection errors, auth failures, command failures, timeouts) to both console and a separate timestamped log file
- Include hostname in all log entries

## Parallelization

- Use `ForEach-Object -Parallel` with a thread-safe approach for collecting results
- Default throttle limit: 10 concurrent hosts

## Command Line Parameters

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `-HostFile` | Yes | - | Path to the input file |
| `-ThrottleLimit` | No | 10 | Number of concurrent hosts |
| `-Timeout` | No | 30 | Seconds before giving up on SSH command |
| `-Retries` | No | 2 | Number of retry attempts for failed hosts |
| `-OutputFile` | No | auto-generated | Custom output CSV filename |
| `-PreserveSSHState` | No (switch) | $false | If set, restore SSH to its original state instead of always disabling |

## Error Handling

- Retry failed hosts up to the configured retry count before logging as failed
- Continue processing remaining hosts on any failure
- Log authentication failures with hostname
- Attempt all hosts regardless of maintenance mode or connection state
