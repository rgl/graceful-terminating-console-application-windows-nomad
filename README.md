# About

Tests how a task is gracefully terminated by [nomad](https://www.nomadproject.io/).

Nomad will send a CTRL+BREAK signal to the running application and will wait for the [task `kill_timeout` property](https://www.nomadproject.io/docs/job-specification/task#kill_timeout) time to expire (defaults to `5s`) before killing it.

**NB** Be aware that `kill_timeout` is capped by the [nomad client `max_kill_timeout` property](https://www.nomadproject.io/docs/configuration/client#max_kill_timeout) (defaults to `30s`).

This uses the [graceful-terminating-console-application-windows](https://github.com/rgl/graceful-terminating-console-application-windows) application to test the nomad behavior.

## Usage

Create the `tmp` directory:

```powershell
mkdir tmp
cd tmp
```

Install nomad:

```powershell
$url = 'https://releases.hashicorp.com/nomad/1.0.0-beta3/nomad_1.0.0-beta3_windows_amd64.zip'
$zipPath = Split-Path -Leaf $url
$exePath = "$PWD/nomad.exe"
(New-Object Net.WebClient).DownloadFile($url, $zipPath)
Expand-Archive $zipPath
Move-Item "$([IO.Path]::GetFileNameWithoutExtension($zipPath))/nomad.exe" .
```

Create an example configuration file:

```powershell
Set-Content -Encoding Ascii config.nomad @"
# see https://www.nomadproject.io/docs/configuration

disable_anonymous_signature = true
disable_update_check = true

telemetry {
  publish_allocation_metrics = true
  publish_node_metrics = true
}

client {
  enabled = true
  network_speed = 1000
}

plugin "raw_exec" {
  config {
    enabled = true
  }
}
"@
```

Start nomad in development mode:

```powershell
.\nomad.exe agent -dev -config config.nomad
```

Access its Web UI:

    http://localhost:4646

Create an example job:

```powershell
$url = 'https://github.com/rgl/graceful-terminating-console-application-windows/releases/download/v0.5.0/graceful-terminating-console-application-windows.zip'
$zipPath = Split-Path -Leaf $url
$exePath = "$PWD/graceful-terminating-console-application-windows.exe"
(New-Object Net.WebClient).DownloadFile($url, $zipPath)
Expand-Archive $zipPath
Move-Item "$([IO.Path]::GetFileNameWithoutExtension($zipPath))/*.exe" .
Set-Content -Encoding Ascii graceful-stop.nomad @"
# see https://www.nomadproject.io/docs/job-specification

job "graceful-stop" {
  datacenters = ["dc1"]
  update {
    stagger = "30s"
    max_parallel = 1
  }
  group "graceful-stop" {
    task "graceful-stop" {
      driver = "raw_exec"
      kill_timeout = "15s"
      config {
        command = "$($exePath -replace '\\','/')"
        args = [
          "10"
        ]
      }
    }
  }
}
"@
./nomad.exe run graceful-stop.nomad
```
