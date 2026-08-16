# LegendaryBatchUpdateLauncher

A simple Windows batch script for use with the **Legendary launcher** that ensures your game is always up to date before launching.

## ✨ Overview

`LegendaryBatchUpdateLauncher` solves a common issue where **Legendary fails to launch a game if an update is required**,
causing it to exit prematurely.

This script:

1. Checks if a game has an available update
2. Automatically installs the update (if found)
3. Checks for cloud saves, syncs if exists
4. Launches the game
5. Watches the actual game executable for process end
6. Does a game save cloud sync after the game process ends
7. Logs the run session of the game launch/cloud save sync etc.

## ⚙️ Requirements

- Windows (CMD / Batch environment)
- Legendary CLI installed and available in your system `PATH`
- A valid installed game via Legendary
- Better when the .bat is converted to a hidden 64bit command executable (bat to exe etc).

## 🔍 Finding Your Game ID

To get the correct game ID (App Name), run:

```cmd
legendary list-installed
```

Look for the **App Name** column — that's what you'll use in the script.

## 🔍 Finding Your Game EXE
- Look in task manager after game is launched, sort by cpu usage will usually put the game at the top
- Below script in powershell will list recent apps from the time you launch the script. May help to narrow it down.
- Check if the game exe has a child process, for example sifu.exe will launch as parent and the child process to actually use in the .bat file is Sifu-Win64-Shipping.exe 

```cmd
$before = Get-Process | Select-Object -ExpandProperty ProcessName
# launch the game here
Start-Sleep -Seconds 20
$after = Get-Process | Select-Object -ExpandProperty ProcessName
Compare-Object $before $after | Where-Object SideIndicator -eq "=>"
```
- A valid installed game via Legendary  

## 🚀 Usage

1. Create a `.bat` or `.cmd` file
2. Copy and paste the script below
3. Replace `gameID` with your actual game ID
4. Replace `YourGame.exe` with the actual game process name
5. Run the script

Full script:

```bat
@echo off
setlocal
:: ── CONFIG ──────────────────────────────────────────────────────────────────
set GAME=GAME_ID
:: Use the internal App Name/ID (not the display title) - find it with:
::   legendary list-installed

set GAME_EXE=GAME.EXE
:: This is the REAL game process (Unreal Engine shipping exe), not Sifu.exe
:: (which is just a small bootstrapper that exits almost immediately).

set LOGFILE=%~dp0legendary_launch_log.txt
:: All output is logged here since the console is hidden (compiled to an
:: invisible-console exe) - nothing printed to screen is ever seen.
:: ─────────────────────────────────────────────────────────────────────────────

echo ============================================== >> "%LOGFILE%"
echo %DATE% %TIME% - Starting launch for %GAME% >> "%LOGFILE%"

echo Updating %GAME% (no-op if already current)... >> "%LOGFILE%"
legendary update %GAME% -y >> "%LOGFILE%" 2>&1
if %ERRORLEVEL% NEQ 0 (
    echo Update failed. Aborting launch. >> "%LOGFILE%"
    mshta "javascript:new ActiveXObject('WScript.Shell').Popup('Legendary update failed for %GAME%. Launch aborted. See legendary_launch_log.txt for details.',0,'Launch Error',48);close();"
    exit /b 1
)
echo Update check complete. >> "%LOGFILE%"

echo Syncing cloud saves for %GAME%... >> "%LOGFILE%"
legendary sync-saves %GAME% -y >> "%LOGFILE%" 2>&1
if %ERRORLEVEL% NEQ 0 (
    echo Warning: save sync before launch failed. Continuing anyway... >> "%LOGFILE%"
)

echo Launching %GAME%... >> "%LOGFILE%"
powershell -NoProfile -WindowStyle Hidden -Command "Start-Process -FilePath 'legendary' -ArgumentList 'launch','%GAME%' -WindowStyle Hidden" >> "%LOGFILE%" 2>&1

:: Give the wrapper/launcher a few seconds to actually spawn the real game exe
timeout /t 15 /nobreak >nul

echo Waiting for %GAME_EXE% to start... >> "%LOGFILE%"
:WAIT_FOR_START
tasklist /FI "IMAGENAME eq %GAME_EXE%" 2>nul | findstr /I "%GAME_EXE%" >nul
if %ERRORLEVEL% NEQ 0 (
    timeout /t 5 /nobreak >nul
    goto WAIT_FOR_START
)

echo %GAME_EXE% is running. Waiting for it to close... >> "%LOGFILE%"
:WAIT_FOR_EXIT
tasklist /FI "IMAGENAME eq %GAME_EXE%" 2>nul | findstr /I "%GAME_EXE%" >nul
if %ERRORLEVEL% EQU 0 (
    timeout /t 5 /nobreak >nul
    goto WAIT_FOR_EXIT
)

echo %GAME_EXE% has closed. Syncing cloud saves... >> "%LOGFILE%"
legendary sync-saves %GAME% -y >> "%LOGFILE%" 2>&1
if %ERRORLEVEL% NEQ 0 (
    echo Warning: save sync after launch failed. Your local save may not be backed up to the cloud. >> "%LOGFILE%"
    mshta "javascript:new ActiveXObject('WScript.Shell').Popup('Cloud save sync failed after playing %GAME%. Your save may not be backed up. See legendary_launch_log.txt for details.',0,'Save Sync Warning',48);close();"
)

echo %DATE% %TIME% - Finished. >> "%LOGFILE%"
endlocal
```

## 🧠 How It Works

- Uses `legendary list-installed --check-updates --tsv` to detect updates
- Pipes output to `findstr` to match your game
- Checks `%ERRORLEVEL%` to determine if an update is required
- Runs `legendary update` only when needed
- Safely aborts if the update fails
- Launches the game via `start` so the script isn't blocked by wrapper/bootstrapper processes that exit early
- Polls for the real game process (`GAME_EXE`) to start, then waits for it to close
- Runs `legendary sync-saves` both before and after the play session to keep cloud saves up to date

## ⚠️ Notes

- The script assumes `legendary` is accessible from your command line
- `GAME_EXE` must be the **actual game process**, not a launcher/bootstrapper stub — many Epic games (e.g. those built on Unreal Engine) use a small launcher exe that hands off to a separate shipping exe and exits almost immediately. Check Task Manager's Details tab while playing to confirm the right process name.
- If Legendary behavior changes, the update detection or sync-saves commands may need adjustment
