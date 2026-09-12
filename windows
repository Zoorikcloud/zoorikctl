<#
.SYNOPSIS
    zoorikctl installer for Windows.

.DESCRIPTION
    Served at https://get.zoorik.com/windows.

        irm https://get.zoorik.com/windows | iex

    A PowerShell script and NOT a bash script behind a WSL prerequisite. A Windows engineer who has
    to install a Linux subsystem before they can install a CLI has been handed a project, not a
    command, and that is the step an onboarding dies on.

    ENVIRONMENT
        ZOORIKCTL_VERSION      install this release instead of the latest (e.g. v1.2.3)
        ZOORIKCTL_INSTALL_DIR  install here instead of %LOCALAPPDATA%\Programs\zoorikctl
        ZOORIKCTL_BASE_URL     download from here instead of GitHub releases (mirrors, air-gapped
                            copies, and this repository's own installer tests)
#>

# Stop on the first error rather than carrying on past a failed download into an install.
$ErrorActionPreference = 'Stop'
Set-StrictMode -Version Latest

# 🔴 TWO REPOSITORIES, AND THEY ARE NOT THE SAME ONE. See the matching note in install.sh.
#
# $Repo is where the release ASSETS live -- public, because a customer's machine is anonymous and
# GitHub will not serve release assets from a private repository to an unauthenticated caller.
#
# $BuildRepo is where the workflow that SIGNED them ran, and it is the identity recorded in the
# Fulcio certificate cosign checks below. Collapsing the two makes every signature check fail
# closed on a correct release, and the message blames the signature rather than the constant.
$Repo = 'storagetax/zoorikctl'
$BuildRepo = 'storagetax/storagetax-k8s-optimizer' 
$Binary = 'zoorikctl.exe'
$ChecksumFile = 'checksums.txt'

function Fail {
    param([string] $What, [string] $Next)
    Write-Host ''
    Write-Host "  $What" -ForegroundColor Red
    if ($Next) {
        Write-Host ''
        Write-Host "  What to do: $Next"
    }
    Write-Host ''
    exit 1
}

function Get-Architecture {
    # PROCESSOR_ARCHITECTURE is the running process's view and lies under WOW64; the OS
    # architecture is the one that decides which binary can run.
    switch ([System.Runtime.InteropServices.RuntimeInformation]::OSArchitecture) {
        'X64'   { return 'amd64' }
        'Arm64' { return 'arm64' }
        default {
            Fail "zoorikctl does not ship a Windows build for $_." `
                 "Open an issue at https://github.com/$Repo/issues naming this architecture."
        }
    }
}

function Get-LatestVersion {
    if ($env:ZOORIKCTL_VERSION) { return $env:ZOORIKCTL_VERSION }
    try {
        # The redirect the /releases/latest URL performs IS the answer, so no JSON parsing.
        $response = Invoke-WebRequest -Uri "https://github.com/$Repo/releases/latest" `
            -MaximumRedirection 0 -ErrorAction SilentlyContinue -UseBasicParsing
        $location = $response.Headers.Location
    } catch {
        $location = $_.Exception.Response.Headers.Location
    }
    if (-not $location) {
        Fail "Could not work out which release of zoorikctl is the latest." `
             "Set ZOORIKCTL_VERSION to a release tag from https://github.com/$Repo/releases."
    }
    return ([string] $location).Split('/')[-1]
}

<#
    🔴 THE CHECKSUM CHECK IS THE POINT OF THIS FILE.

    Everything else is convenience. An installer that writes an unverified download onto PATH is a
    supply-chain hole shipped to every customer, and it is the one defect in this lane that cannot
    be fixed after the fact: a compromised mirror, a hijacked CDN edge or a corporate proxy that
    rewrites bodies all present as a normal, successful install.

    Get-FileHash is in Windows PowerShell 4.0 and every PowerShell 7, so this needs nothing
    installed.
#>
function Assert-Checksum {
    param([string] $Archive, [string] $Asset, [string] $Sums)

    $line = Get-Content -LiteralPath $Sums |
        Where-Object { $_ -match "\s\*?$([regex]::Escape($Asset))\s*$" } |
        Select-Object -First 1
    if (-not $line) {
        Fail "The release's $ChecksumFile does not list $Asset." `
             "Nothing has been installed. The release is incomplete; report it at https://github.com/$Repo/issues."
    }
    $expected = ($line -split '\s+')[0].ToLowerInvariant()
    $actual = (Get-FileHash -LiteralPath $Archive -Algorithm SHA256).Hash.ToLowerInvariant()

    if ($expected -ne $actual) {
        Remove-Item -LiteralPath $Archive -Force -ErrorAction SilentlyContinue
        Write-Host ''
        Write-Host '  The downloaded file is not the file this release published.' -ForegroundColor Red
        Write-Host "    expected  $expected"
        Write-Host "    actually  $actual"
        Write-Host ''
        Write-Host '  What to do: nothing has been installed and the download has been deleted.'
        Write-Host '  Retry once - a truncated download looks exactly like this. If it repeats,'
        Write-Host '  something between you and GitHub is altering the file, and you should not'
        Write-Host '  install it.'
        Write-Host ''
        exit 1
    }
    Write-Host "  checksum  ok ($actual)"
}

function Assert-Signature {
    param([string] $Sums, [string] $Signature, [string] $Certificate)

    if (-not (Get-Command cosign -ErrorAction SilentlyContinue)) {
        Write-Host '  signature not checked (cosign is not installed)'
        return
    }
    if (-not (Test-Path $Signature) -or -not (Test-Path $Certificate)) {
        Write-Host '  signature not checked (this release published none)'
        return
    }
    & cosign verify-blob --certificate $Certificate --signature $Signature `
        --certificate-identity-regexp "^https://github.com/$BuildRepo/" `
        --certificate-oidc-issuer 'https://token.actions.githubusercontent.com' $Sums 2>$null
    if ($LASTEXITCODE -ne 0) {
        Fail "The release's checksum file is not signed by this repository's build." `
             "Nothing has been installed. Do not install this download."
    }
    Write-Host "  signature ok (keyless, built by $BuildRepo)"
}

function Install-Stxctl {
    $arch = Get-Architecture
    $version = Get-LatestVersion
    $asset = "zoorikctl_${version}_windows_${arch}.zip"
    $base = if ($env:ZOORIKCTL_BASE_URL) {
        $env:ZOORIKCTL_BASE_URL
    } else {
        "https://github.com/$Repo/releases/download/$version"
    }

    Write-Host ''
    Write-Host "Installing zoorikctl $version (windows/$arch)"

    $work = Join-Path ([System.IO.Path]::GetTempPath()) ("zoorikctl-" + [guid]::NewGuid())
    New-Item -ItemType Directory -Path $work -Force | Out-Null
    try {
        # TLS 1.2 explicitly: Windows PowerShell 5.1 still defaults to SSL3/TLS1 on unpatched
        # hosts, and GitHub refuses those - which surfaces as an unhelpful "could not create SSL/TLS
        # secure channel" rather than as anything about protocol versions.
        [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12

        $archive = Join-Path $work $asset
        $sums = Join-Path $work $ChecksumFile
        Invoke-WebRequest -Uri "$base/$asset" -OutFile $archive -UseBasicParsing
        Invoke-WebRequest -Uri "$base/$ChecksumFile" -OutFile $sums -UseBasicParsing
        foreach ($extra in @("$ChecksumFile.sig", "$ChecksumFile.pem")) {
            try {
                Invoke-WebRequest -Uri "$base/$extra" -OutFile (Join-Path $work $extra) `
                    -UseBasicParsing -ErrorAction SilentlyContinue
            } catch { }
        }

        Assert-Checksum -Archive $archive -Asset $asset -Sums $sums
        Assert-Signature -Sums $sums -Signature (Join-Path $work "$ChecksumFile.sig") `
            -Certificate (Join-Path $work "$ChecksumFile.pem")

        Expand-Archive -LiteralPath $archive -DestinationPath $work -Force
        $extracted = Join-Path $work $Binary
        if (-not (Test-Path $extracted)) {
            Fail "The archive does not contain $Binary." `
                 "Nothing has been installed; report this at https://github.com/$Repo/issues."
        }

        $target = if ($env:ZOORIKCTL_INSTALL_DIR) {
            $env:ZOORIKCTL_INSTALL_DIR
        } else {
            Join-Path $env:LOCALAPPDATA 'Programs\zoorikctl'
        }
        # Per-user by default and never Program Files: this must install on a locked-down corporate
        # laptop without a UAC prompt, and an installer that silently needs administrator is an
        # installer that fails for the customer most likely to be evaluating us.
        New-Item -ItemType Directory -Path $target -Force | Out-Null
        Move-Item -LiteralPath $extracted -Destination (Join-Path $target $Binary) -Force

        Write-Host "  installed $(Join-Path $target $Binary)"
        Add-ToUserPath -Directory $target
        Write-Host ''
        Write-Host '  Next: paste the connect command from the Storagetax console.'
        Write-Host ''
    } finally {
        Remove-Item -LiteralPath $work -Recurse -Force -ErrorAction SilentlyContinue
    }
}

function Add-ToUserPath {
    param([string] $Directory)

    $userPath = [Environment]::GetEnvironmentVariable('Path', 'User')
    if ($userPath -and ($userPath -split ';' | Where-Object { $_ -eq $Directory })) {
        return
    }
    $updated = if ($userPath) { "$userPath;$Directory" } else { $Directory }
    [Environment]::SetEnvironmentVariable('Path', $updated, 'User')
    # The current session does not inherit the change, and telling somebody to "reopen your
    # terminal" after they have just pasted one line is how the next step fails for a reason that
    # has nothing to do with us.
    $env:Path = "$env:Path;$Directory"
    Write-Host "  added $Directory to your user PATH (this session included)"
}

Install-Stxctl
