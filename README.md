name: Windows Cloud RDP

on:
  workflow_dispatch:

jobs:
  build:
    runs-on: windows-latest
    # حداکثر زمان مجاز گیت‌هاب برای یک Job برابر با 360 دقیقه است
    timeout-minutes: 360

    steps:
      - name: 1. Enable Remote Desktop & Firewall
        run: |
          Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0
          Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
          Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name "UserAuthentication" -Value 0

      - name: 2. Create Admin Account
        run: |
          # ساخت اکانت با پسورد مشخص
          # توجه: برای امنیت بیشتر، بهتر است پسورد را هم در Secrets قرار دهید
          $Password = ConvertTo-SecureString "Pass123456!" -AsPlainText -Force
          New-LocalUser -Name "NvDAdmin" -Password $Password -FullName "RDP Admin" -Description "GitHub RDP User"
          Add-LocalGroupMember -Group "Administrators" -Member "NvDAdmin"
          Write-Host "Admin account 'NvDAdmin' created successfully."

      - name: 3. Install & Connect Tailscale VPN
        env:
          TAILSCALE_AUTHKEY: ${{ secrets.TAILSCALE_AUTHKEY }}
        run: |
          # دانلود و نصب Tailscale به صورت بی‌صدا (Silent)
          Invoke-WebRequest "https://pkgs.tailscale.com/stable/tailscale-setup-latest-amd64.msi" -OutFile "C:\tailscale.msi"
          Start-Process msiexec -ArgumentList "/i C:\tailscale.msi /quiet /norestart" -Wait
          
          # انتظار برای نصب کامل سرویس
          Start-Sleep -Seconds 20
          
          # اتصال به شبکه Tailscale
          & "C:\Program Files\Tailscale\tailscale.exe" up --authkey="${{ secrets.TAILSCALE_AUTHKEY }}" --hostname="github-rdp-server"
          
          # دریافت IP اختصاصی Tailscale
          Start-Sleep -Seconds 10
          $ts_ip = & "C:\Program Files\Tailscale\tailscale.exe" ip -4
          
          Write-Host ""
          Write-Host "==================================================" -ForegroundColor Green
          Write-Host "  >>> RDP SERVER IS READY! <<<" -ForegroundColor Cyan
          Write-Host "  Tailscale IP : $ts_ip" -ForegroundColor Yellow
          Write-Host "  Username     : NvDAdmin" -ForegroundColor Yellow
          Write-Host "  Password     : Pass123456!" -ForegroundColor Yellow
          Write-Host "  >> Connect via RDP to: $ts_ip" -ForegroundColor Magenta
          Write-Host "==================================================" -ForegroundColor Green

      - name: 4. Keep Server Alive (Max 6 Hours)
        run: |
          Write-Host "Starting keep-alive loop to maintain connection..."
          $startTime = Get-Date
          # 6 ساعت برابر است با 21600 ثانیه
          $maxSeconds = 21600 
          
          while ($true) {
              $elapsed = (Get-Date) - $startTime
              if ($elapsed.TotalSeconds -ge $maxSeconds) {
                  Write-Host "Reached 6-hour limit. Closing workflow."
                  break
              }
              
              Write-Host "Server running... Uptime: $([Math]::Round($elapsed.TotalMinutes)) minutes. Remaining: $([Math]::Round(($maxSeconds - $elapsed.TotalSeconds) / 60)) minutes."
              
              # هر 5 دقیقه یک بار وضعیت را چاپ کن تا Workflow غیرفعال نشود
              Start-Sleep -Seconds 300
          }
