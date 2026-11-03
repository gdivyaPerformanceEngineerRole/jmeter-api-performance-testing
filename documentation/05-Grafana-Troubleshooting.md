# Grafana Troubleshooting — JMeter API Performance Testing Project

**Environment:** Windows, Grafana 13.3.0, InfluxDB 1.13.2, JMeter 5.6.3  
**Status:** Resolved; Grafana home page accessible at `http://localhost:3000`.

## Scenario

While setting up Grafana to visualize JMeter metrics stored in InfluxDB, I encountered two issues: the browser initially reported `ERR_CONNECTION_REFUSED`, and then Grafana rejected administrator credentials even after an attempted password reset.

## 1. Connection refused

Browser error: `localhost refused to connect` (`ERR_CONNECTION_REFUSED`).

I checked the service and port using Administrator PowerShell:

```powershell
Get-Service -Name Grafana
Test-NetConnection localhost -Port 3000
```

A later port check showed `TcpTestSucceeded : True`. I restarted the service and checked its status:

```powershell
Restart-Service -Name Grafana
Get-Service -Name Grafana
```

The service reported `Running`, and the login page became accessible. A successful TCP check confirms connectivity to the port at that moment, not successful authentication.

## 2. Login failure

Grafana showed `Login failed: Invalid username or password` for the administrator account. An initial password-reset attempt reported a password-policy error; a later attempt reported success, but login still failed.

### Inspect the Windows service configuration

```powershell
sc.exe qc Grafana
& "C:\Program Files\GrafanaLabs\svc-13.3.0.0\nssm.exe" get Grafana AppDirectory
& "C:\Program Files\GrafanaLabs\svc-13.3.0.0\nssm.exe" get Grafana AppParameters
& "C:\Program Files\GrafanaLabs\svc-13.3.0.0\nssm.exe" get Grafana Application
```

Observed values:

- AppDirectory: `C:\Program Files\GrafanaLabs\grafana\`
- AppParameters: `server`
- Application: `C:\Program Files\GrafanaLabs\grafana\bin\grafana.exe`

### Locate the application database

```powershell
Get-ChildItem "C:\Program Files\GrafanaLabs\grafana\data" -Filter "grafana.db" -Recurse |
  Select-Object FullName, Length, LastWriteTime
```

Located: `C:\Program Files\GrafanaLabs\grafana\data\grafana.db`.

Grafana uses SQLite by default for its application database. The CLI reset should target the same data directory as the service.

## 3. Unsupported flag

An attempted CLI command used `--data` and returned:

```text
Incorrect Usage: flag provided but not defined: -data
```

This Grafana CLI version does not accept `--data` as a flag. I used `--configOverrides` instead.

## 4. Successful password reset

In Administrator PowerShell:

```powershell
& "C:\Program Files\GrafanaLabs\grafana\bin\grafana.exe" cli --homepath "C:\Program Files\GrafanaLabs\grafana" --config "C:\Program Files\GrafanaLabs\grafana\conf\defaults.ini" --configOverrides "cfg:default.paths.data=C:\Program Files\GrafanaLabs\grafana\data" admin reset-admin-password "<NEW_STRONG_PASSWORD>"
```

**Security:** Replace `<NEW_STRONG_PASSWORD>` with a unique strong password. Never include actual passwords in documentation or GitHub commits.

The output confirmed the intended data path and displayed:

```text
Admin password changed successfully
```

I restarted the service:

```powershell
Restart-Service -Name Grafana
```

I then successfully signed in and reached the Grafana home page.

## Root-cause assessment

**Observed:** Password resets had reported success without resolving login failure. The successful sequence explicitly targeted the installation's data directory, restarted the service, and verified login.

**Likely explanation:** Earlier resets may have operated with different effective configuration or database paths. This was not conclusively established from server logs, so it should be described as a likely configuration mismatch rather than a proven root cause.

## Interview questions

**Q: How did you troubleshoot Grafana when `localhost:3000` was unavailable?**  
A: I checked the Grafana Windows service and tested TCP port 3000. I restarted the service, verified it was running, and retried the browser.

**Q: How did you diagnose invalid administrator credentials?**  
A: I inspected the NSSM-managed service configuration and found Grafana's SQLite database path. I reset the administrator password using the CLI with an explicit data-path override, restarted the service, and verified the login.

**Q: What is NSSM?**  
A: Non-Sucking Service Manager, a utility for running applications as Windows services. I used it to inspect the executable, startup parameters, and working directory.

**Q: What does `TcpTestSucceeded : True` prove?**  
A: Only that a TCP connection to the tested port succeeded at that time; it does not verify Grafana login or dashboard health.

**Q: Why did the `--data` command fail?**  
A: The installed Grafana CLI version did not define that flag. I used the supported `--configOverrides` option to select the intended data directory.

**Q: What did you learn?**  
A: Verify the running service's effective configuration and data location, and always test the end-user outcome after a maintenance command reports success.

## Next step

Configure InfluxDB as a Grafana data source and build JMeter response-time, throughput, error-rate, and percentile panels.
