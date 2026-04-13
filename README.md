# LegendaryBatchUpdateLauncher
For use with legendary launcher on windows. Check for updates to a game, if update found, update it, then launch the game. Currently legendary fails to launch and drops out if the game needs an update. Copy and paste the below into a bat or cmd file replacing gameID with the game ID you want to launch.

@echo off
setlocal

:: ── CONFIG ──────────────────────────────────────────────────────────────────
set GAME=gameID
:: Replace gameID with your game's legendary App Name/ID.
:: Find it by running:  legendary list-installed
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

echo Launching %GAME%...
legendary launch %GAME%
