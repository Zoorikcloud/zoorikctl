& {
    $ErrorActionPreference = 'Stop'
    $ProgressPreference = 'SilentlyContinue'
    Set-StrictMode -Version Latest

    $Repo = 'Zoorikcloud/zoorikctl'
    $BuildRepo = 'Zoorikcloud/k8spilot'
    $Binary = 'zoorikctl.exe'
    $ChecksumFile = 'checksums.txt'
    $StopMarker = 'zoorikctl-install-stopped'

    function Say {
        param([string] $Message)
        Write-Host "[zoorikctl] $Message"
    }

    function Fail {
        param([string] $What, [string] $Next)
        Write-Host "[zoorikctl] Error: $What" -ForegroundColor Red
        if ($Next) {
            Write-Host "[zoorikctl] $Next"
        }
        throw $StopMarker
    }

    function Get-Architecture {
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
            Write-Host '[zoorikctl] Error: Checksum mismatch. The downloaded file is not the file this release published.' -ForegroundColor Red
            Say "  expected  $expected"
            Say "  actually  $actual"
            Say 'Nothing has been installed and the download has been deleted. Retry once: a truncated download looks exactly like this. If it repeats, something between you and GitHub is altering the file, and you should not install it.'
            throw $StopMarker
        }
        Say 'Checksum verified.'
    }

    function Assert-Signature {
        param([string] $Sums, [string] $Signature, [string] $Certificate)

        if (-not (Get-Command cosign -ErrorAction SilentlyContinue)) {
            return
        }
        if (-not (Test-Path $Signature) -or -not (Test-Path $Certificate)) {
            Say 'Signature not checked (this release published none).'
            return
        }
        Say 'Verifying signature...'
        & cosign verify-blob --certificate $Certificate --signature $Signature `
            --certificate-identity-regexp "^https://github.com/$BuildRepo/" `
            --certificate-oidc-issuer 'https://token.actions.githubusercontent.com' $Sums 2>$null
        if ($LASTEXITCODE -ne 0) {
            Fail "The release's checksum file is not signed by this repository's build." `
                 "Nothing has been installed. Do not install this download."
        }
        Say "Signature verified (keyless, built by $BuildRepo)."
    }

    function Install-Zoorikctl {
        $arch = Get-Architecture
        if (-not $env:ZOORIKCTL_VERSION) { Say 'Resolving latest version...' }
        $version = Get-LatestVersion
        Say "Version: $version"
        $asset = "zoorikctl_${version}_windows_${arch}.zip"
        $base = if ($env:ZOORIKCTL_BASE_URL) {
            $env:ZOORIKCTL_BASE_URL
        } else {
            "https://github.com/$Repo/releases/download/$version"
        }

        $work = Join-Path ([System.IO.Path]::GetTempPath()) ("zoorikctl-" + [guid]::NewGuid())
        New-Item -ItemType Directory -Path $work -Force | Out-Null
        try {
            [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12

            $archive = Join-Path $work $asset
            $sums = Join-Path $work $ChecksumFile
            Say "Downloading $base/$asset..."
            Invoke-WebRequest -Uri "$base/$asset" -OutFile $archive -UseBasicParsing
            Invoke-WebRequest -Uri "$base/$ChecksumFile" -OutFile $sums -UseBasicParsing
            foreach ($extra in @("$ChecksumFile.sig", "$ChecksumFile.pem")) {
                try {
                    Invoke-WebRequest -Uri "$base/$extra" -OutFile (Join-Path $work $extra) `
                        -UseBasicParsing -ErrorAction SilentlyContinue
                } catch { }
            }

            Say 'Verifying checksum...'
            Assert-Checksum -Archive $archive -Asset $asset -Sums $sums
            Assert-Signature -Sums $sums -Signature (Join-Path $work "$ChecksumFile.sig") `
                -Certificate (Join-Path $work "$ChecksumFile.pem")

            Say 'Extracting...'
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
            New-Item -ItemType Directory -Path $target -Force | Out-Null
            Move-Item -LiteralPath $extracted -Destination (Join-Path $target $Binary) -Force

            Say "zoorikctl $version installed to $(Join-Path $target $Binary)"
            Add-ToUserPath -Directory $target
            Say "Run 'zoorikctl --help' to get started, or paste the connect command from the Zoorik console."
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
        $env:Path = "$env:Path;$Directory"
        Say "Added $Directory to your user PATH (this session included)."
    }

    try {
        Install-Zoorikctl
    } catch {
        if ("$_" -ne $StopMarker) { throw }
        if ($PSCommandPath) { exit 1 }
    }
}
