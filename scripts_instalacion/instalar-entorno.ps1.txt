# ==========================================================
# Configuración UTF-8
# ==========================================================

[Console]::InputEncoding  = New-Object System.Text.UTF8Encoding($false)
[Console]::OutputEncoding = New-Object System.Text.UTF8Encoding($false)
$OutputEncoding = New-Object System.Text.UTF8Encoding($false)

# ==========================================================
# Funciones auxiliares
# ==========================================================

function Write-Section {
    param([string]$Text)

    Write-Host ""
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host " $Text" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Cyan
}

function Refresh-Path {
    $machinePath = [Environment]::GetEnvironmentVariable("Path", "Machine")
    $userPath    = [Environment]::GetEnvironmentVariable("Path", "User")

    $env:Path = "$machinePath;$userPath"
}

function Test-CommandAvailable {
    param([string]$Command)

    return [bool](Get-Command $Command -ErrorAction SilentlyContinue)
}

function Test-VersionCommand {
    param(
        [string]$Name,
        [string]$Command
    )

    Write-Host ""
    Write-Host "$Name :" -ForegroundColor Yellow

    try {
        Invoke-Expression $Command
        Write-Host "[OK] $Name" -ForegroundColor Green
        return $true
    }
    catch {
        Write-Host "[ERROR] No se ha podido comprobar $Name" -ForegroundColor Red
        return $false
    }
}

# ==========================================================
# Comprobar permisos de administrador
# ==========================================================

$principal = New-Object Security.Principal.WindowsPrincipal(
    [Security.Principal.WindowsIdentity]::GetCurrent()
)

$isAdmin = $principal.IsInRole(
    [Security.Principal.WindowsBuiltInRole]::Administrator
)

if (-not $isAdmin) {
    Write-Host ""
    Write-Host "[ERROR] Este script debe ejecutarse como administrador." -ForegroundColor Red
    Write-Host "Abre PowerShell como administrador y vuelve a ejecutarlo."
    exit 1
}

# ==========================================================
# Comprobar WinGet
# ==========================================================

Write-Section "Comprobación inicial"

if (-not (Test-CommandAvailable "winget")) {
    Write-Host "[ERROR] WinGet no está disponible en este equipo." -ForegroundColor Red
    Write-Host ""
    Write-Host "Instala o actualiza 'App Installer' desde Microsoft Store"
    Write-Host "y vuelve a ejecutar el script."
    exit 1
}

Write-Host "[OK] WinGet disponible." -ForegroundColor Green
winget --version

# ==========================================================
# Paquetes
# ==========================================================

$packages = @(
    @{ Name = "JetBrains Toolbox";     Id = "JetBrains.Toolbox" },
    @{ Name = "Python 3.14";           Id = "Python.Python.3.14" },
    @{ Name = "Git";                   Id = "Git.Git" },
    @{ Name = "GitHub CLI";            Id = "GitHub.cli" },
    @{ Name = "Visual Studio Code";     Id = "Microsoft.VisualStudioCode" },
    @{ Name = "Notepad++";             Id = "Notepad++.Notepad++" },
    @{ Name = "Eclipse Temurin JDK 25"; Id = "EclipseAdoptium.Temurin.25.JDK" }
)

$packageResults = @()

Write-Section "Instalación del entorno de desarrollo"

foreach ($package in $packages) {

    $name = $package.Name
    $id   = $package.Id

    Write-Host ""
    Write-Host "Procesando $name..." -ForegroundColor Yellow

    winget list `
        --id $id `
        --exact `
        --accept-source-agreements | Out-Null

    $installed = ($LASTEXITCODE -eq 0)

    if ($installed) {

        Write-Host "Ya está instalado. Comprobando actualizaciones..." -ForegroundColor DarkGray

        winget upgrade `
            --id $id `
            --exact `
            --silent `
            --disable-interactivity `
            --accept-package-agreements `
            --accept-source-agreements

        $exitCode = $LASTEXITCODE

        if ($exitCode -eq 0) {
            Write-Host "[OK] $name actualizado correctamente." -ForegroundColor Green
            $packageResults += [PSCustomObject]@{
                Name   = $name
                Status = "OK"
            }
        }
        elseif ($exitCode -eq -1978335189) {
            Write-Host "[OK] $name ya está actualizado." -ForegroundColor Green
            $packageResults += [PSCustomObject]@{
                Name   = $name
                Status = "OK"
            }
        }
        else {
            Write-Host "[REVISAR] $name (código $exitCode)" -ForegroundColor Red
            $packageResults += [PSCustomObject]@{
                Name   = $name
                Status = "REVISAR"
            }
        }

    }
    else {

        Write-Host "No está instalado. Instalando..." -ForegroundColor DarkGray

        winget install `
            --id $id `
            --exact `
            --silent `
            --disable-interactivity `
            --accept-package-agreements `
            --accept-source-agreements

        $exitCode = $LASTEXITCODE

        if ($exitCode -eq 0) {
            Write-Host "[OK] $name instalado correctamente." -ForegroundColor Green
            $packageResults += [PSCustomObject]@{
                Name   = $name
                Status = "OK"
            }
        }
        else {
            Write-Host "[REVISAR] $name (código $exitCode)" -ForegroundColor Red
            $packageResults += [PSCustomObject]@{
                Name   = $name
                Status = "REVISAR"
            }
        }
    }
}

# ==========================================================
# Recargar PATH
# ==========================================================

Write-Section "Actualización del PATH"

Refresh-Path

Write-Host "[OK] PATH recargado para esta sesión." -ForegroundColor Green

# ==========================================================
# Actualizar pip
# ==========================================================

Write-Section "Actualización de pip"

if (Test-CommandAvailable "python") {

    python -m pip install --upgrade pip

    if ($LASTEXITCODE -eq 0) {
        Write-Host "[OK] pip actualizado/comprobado." -ForegroundColor Green
    }
    else {
        Write-Host "[REVISAR] No se ha podido actualizar pip." -ForegroundColor Red
    }

}
else {

    Write-Host "[REVISAR] Python no está disponible todavía en el PATH." -ForegroundColor Red
}

# ==========================================================
# Verificación
# ==========================================================

Write-Section "Verificación del entorno"

$verificationResults = @()

$verificationResults += [PSCustomObject]@{
    Name   = "Python"
    Status = if (Test-VersionCommand "Python" "python --version") { "OK" } else { "ERROR" }
}

$verificationResults += [PSCustomObject]@{
    Name   = "pip"
    Status = if (Test-VersionCommand "pip" "python -m pip --version") { "OK" } else { "ERROR" }
}

$verificationResults += [PSCustomObject]@{
    Name   = "Git"
    Status = if (Test-VersionCommand "Git" "git --version") { "OK" } else { "ERROR" }
}

$verificationResults += [PSCustomObject]@{
    Name   = "GitHub CLI"
    Status = if (Test-VersionCommand "GitHub CLI" "gh --version") { "OK" } else { "ERROR" }
}

$verificationResults += [PSCustomObject]@{
    Name   = "Visual Studio Code"
    Status = if (Test-VersionCommand "Visual Studio Code" "code --version") { "OK" } else { "ERROR" }
}

$verificationResults += [PSCustomObject]@{
    Name   = "Java"
    Status = if (Test-VersionCommand "Java" "java --version") { "OK" } else { "ERROR" }
}

$verificationResults += [PSCustomObject]@{
    Name   = "Java Compiler"
    Status = if (Test-VersionCommand "Java Compiler" "javac --version") { "OK" } else { "ERROR" }
}

# ==========================================================
# Resumen
# ==========================================================

Write-Section "Resumen"

foreach ($result in $packageResults) {

    if ($result.Status -eq "OK") {
        Write-Host "[OK] $($result.Name)" -ForegroundColor Green
    }
    else {
        Write-Host "[REVISAR] $($result.Name)" -ForegroundColor Red
    }
}

Write-Host ""

foreach ($result in $verificationResults) {

    if ($result.Status -eq "OK") {
        Write-Host "[OK] $($result.Name)" -ForegroundColor Green
    }
    else {
        Write-Host "[ERROR] $($result.Name)" -ForegroundColor Red
    }
}

$errors =
    ($packageResults | Where-Object { $_.Status -ne "OK" }).Count +
    ($verificationResults | Where-Object { $_.Status -ne "OK" }).Count

Write-Host ""

if ($errors -eq 0) {
    Write-Host "Entorno instalado y verificado correctamente." -ForegroundColor Green
}
else {
    Write-Host "Proceso finalizado con $errors elemento(s) a revisar." -ForegroundColor Yellow
}

Write-Host ""
