# PS скрипт - чистит папки темп всех пользователей
Скрипт запускается от админа, рядом с ним должен быть файл computers.txt, в котором должен быть список устройств. Каждое устройство с новой строки. <br>
Лог выполнения создаётся рядом со скриптом
```
# Cleans Temp folder for ALL users (including active profiles)
# Ignores locked files, continues cleaning
# Run as domain admin

$ScriptPath = Split-Path -Parent $MyInvocation.MyCommand.Path
$ComputerList = Join-Path $ScriptPath "computers.txt"
$LogFile = Join-Path $ScriptPath "clean_temp_log.txt"

function Write-Log {
    param([string]$Message)
    $TimeStamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    "$TimeStamp - $Message" | Out-File -FilePath $LogFile -Append -Encoding UTF8
}

if (-not (Test-Path $ComputerList)) {
    Write-Log "ERROR: File $ComputerList not found!"
    exit 1
}

$Computers = Get-Content $ComputerList | Where-Object { $_.Trim() -ne "" }

foreach ($Computer in $Computers) {
    $Computer = $Computer.Trim()
    Write-Log "Processing: $Computer"

    if (-not (Test-Connection -ComputerName $Computer -Count 1 -Quiet)) {
        Write-Log "  [ERROR] $Computer is offline"
        continue
    }

    try {
        $Profiles = Get-WmiObject -ComputerName $Computer -Class Win32_UserProfile | Where-Object { $_.Special -eq $false }

        foreach ($Profile in $Profiles) {
            $UserPath = $Profile.LocalPath
            $TempPath = Join-Path $UserPath "AppData\Local\Temp"
            $Sid = $Profile.SID
            $Loaded = $Profile.Loaded
            $UncTempPath = "\\$Computer\C$\$($TempPath.Substring(3))".Replace(':', '$')

            if ($Loaded) {
                Write-Log "  [ACTIVE] $Sid -> $TempPath (via Invoke-Command)"
                try {
                    $Result = Invoke-Command -ComputerName $Computer -ScriptBlock {
                        param($Path)
                        if (Test-Path $Path) {
                            $Items = Get-ChildItem $Path -Force -ErrorAction SilentlyContinue
                            $Deleted = 0
                            foreach ($Item in $Items) {
                                try {
                                    Remove-Item $Item.FullName -Force -Recurse -ErrorAction Stop
                                    $Deleted++
                                } catch {
                                    # Skip locked files
                                }
                            }
                            if ($Deleted -gt 0) {
                                return "Deleted $Deleted items (skipped locked)"
                            } else {
                                return "Empty or all locked"
                            }
                        } else {
                            return "Folder not found"
                        }
                    } -ArgumentList $TempPath -ErrorAction Stop

                    Write-Log "  [SUCCESS] $Result"
                } catch {
                    Write-Log "  [ERROR] Invoke-Command failed: $($_.Exception.Message)"
                }
            } else {
                if (Test-Path $UncTempPath) {
                    try {
                        $Items = Get-ChildItem $UncTempPath -Force -ErrorAction SilentlyContinue
                        $Deleted = 0
                        foreach ($Item in $Items) {
                            try {
                                Remove-Item $Item.FullName -Force -Recurse -ErrorAction Stop
                                $Deleted++
                            } catch {
                                # Skip locked
                            }
                        }
                        if ($Deleted -gt 0) {
                            Write-Log "  [SUCCESS] Deleted $Deleted items: $UncTempPath (skipped locked)"
                        } else {
                            Write-Log "  [INFO] Empty or all locked: $UncTempPath"
                        }
                    } catch {
                        Write-Log "  [ERROR] Deletion failed: $($_.Exception.Message)"
                    }
                } else {
                    Write-Log "  [WARNING] Not found: $UncTempPath"
                }
            }
        }
    } catch {
        Write-Log "  [ERROR] WMI query failed: $($_.Exception.Message)"
    }
}

Write-Log "Cleanup completed."
```
