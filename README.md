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
5. Does a game save cloud sync after the game process ends

## ⚙️ Requirements

- Windows (CMD / Batch environment)
- Legendary CLI installed and available in your system `PATH`
- A valid installed game via Legendary

## 🔍 Finding Your Game ID

To get the correct game ID (App Name), run:

```cmd
legendary list-installed
```

Look for the **App Name** column — that's what you'll use in the script.

## 🚀 Usage

1. Create a `.bat` or `.cmd` file
2. Copy and paste the script below
3. Replace `gameID` with your actual game ID
4. Replace `YourGame.exe` with the actual game process name
5. Run the script

```bat
@echo off
setlocal
:: ── CONFIG ──────────────────────────────────────────────────────────────────
set GAME=gameID
:: Replace gameID with your game's legendary App Name/ID.
:: Find it by running:  legendary list-installed

set GAME_EXE=YourGame.exe
:: Replace with the actual game process name (Task Manager > Details tab
:: while the game is running will show you the exact .exe name).
:: ─────────────────────────────────────────────────────────────────────────────

echo Checking for updates to %GAME%...
:: list-installed --check-updates --tsv prints a line for each game needing an update.
:: If our game appears in that output, an update is available.
legendary list-installed --check-updates --tsv 2>nul | findstr /I "%GAME%" >nul
if %ERRORLEVEL% EQU 0 (
    echo Update found. Updating %GAME%...
    legendary update %GAME% -y
    if %ERRORLEVEL% NEQ 0 (
        echo Update failed. Aborting launch.
        pause
        exit /b 1
    )
    echo Update complete.
) else (
    echo No update needed.
)

echo Syncing cloud saves for %GAME%...
legendary sync-saves %GAME% -y
if %ERRORLEVEL% NEQ 0 (
    echo Warning: save sync before launch failed. Continuing anyway...
)

echo Launching %GAME%...
start "" legendary launch %GAME%

:: Give the wrapper/launcher a few seconds to actually spawn the real game exe
timeout /t 15 /nobreak >nul

echo Waiting for %GAME_EXE% to start...
:WAIT_FOR_START
tasklist /FI "IMAGENAME eq %GAME_EXE%" 2>nul | findstr /I "%GAME_EXE%" >nul
if %ERRORLEVEL% NEQ 0 (
    timeout /t 5 /nobreak >nul
    goto WAIT_FOR_START
)

echo %GAME_EXE% is running. Waiting for it to close...
:WAIT_FOR_EXIT
tasklist /FI "IMAGENAME eq %GAME_EXE%" 2>nul | findstr /I "%GAME_EXE%" >nul
if %ERRORLEVEL% EQU 0 (
    timeout /t 5 /nobreak >nul
    goto WAIT_FOR_EXIT
)

echo %GAME_EXE% has closed. Syncing cloud saves...
legendary sync-saves %GAME% -y
if %ERRORLEVEL% NEQ 0 (
    echo Warning: save sync after launch failed. Your local save may not be backed up to the cloud.
    pause
)

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
