# CyberArk Privilege Cloud Lab - Complete Installation Guide

## Document Information

| Property | Value |
|----------|-------|
| Version | 1.0 |
| Classification | Technical Implementation Guide |
| Audience | Solutions Architect, Systems Engineer, Security Administrator |
| Estimated Duration | 2-4 days (depending on environment readiness) |

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Prerequisites and Planning](#2-prerequisites-and-planning)
3. [Infrastructure Preparation](#3-infrastructure-preparation)
4. [Phase 1: Active Directory Configuration](#4-phase-1-active-directory-configuration)
5. [Phase 2: CyberArk Identity Setup](#5-phase-2-cyberark-identity-setup)
6. [Phase 3: User Management](#6-phase-3-user-management)
7. [Phase 4: Multi-Factor Authentication](#7-phase-4-multi-factor-authentication)
8. [Phase 5: PSM High Availability Deployment](#8-phase-5-psm-high-availability-deployment)
9. [Phase 6: SIA Configuration](#9-phase-6-sia-configuration)
10. [Phase 7: Naming Conventions and Standards](#10-phase-7-naming-conventions-and-standards)
11. [Phase 8: OKTA Integration](#11-phase-8-okta-integration)
12. [Validation and Testing](#12-validation-and-testing)
13. [Troubleshooting](#13-troubleshooting)
14. [Appendices](#14-appendices)

---

## 1. Introduction

### 1.1 Purpose

This document provides a comprehensive, step-by-step installation guide for deploying a CyberArk Privilege Cloud lab environment. The guide is designed for solutions architects and systems engineers who need to implement a fully functional PAM environment with high availability, multi-factor authentication, and enterprise directory integration.

### 1.2 Scope

This installation covers:

- Active Directory as Identity Provider (IdP)
- TIER model directory structure implementation
- User account management and cleanup
- Multi-factor authentication (Email OTP, SMS OTP, Mobile App)
- PSM (Privileged Session Manager) in High Availability configuration
- SIA (Secure Infrastructure Access) connector deployment
- Safe and Platform naming conventions
- OKTA SAML federation (optional)

### 1.3 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           CYBERARK PRIVILEGE CLOUD                              │
│                    (*.privilegecloud.cyberark.cloud)                            │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       │ HTTPS/443
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              ON-PREMISES ZONE                                   │
│                            (indramind.cyblab)                                   │
│                                                                                 │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐             │
│  │      DC01       │    │   Connector01   │    │   Connector02   │             │
│  │                 │    │                 │    │                 │             │
│  │  Active         │    │  PSM Component  │    │  PSM Component  │             │
│  │  Directory      │    │  SIA Connector  │    │  SIA Connector  │             │
│  │                 │    │                 │    │                 │             │
│  │  10.10.3.151    │    │  10.10.3.8      │    │  10.10.3.70     │             │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘             │
│           │                      │                      │                       │
│           └──────────────────────┼──────────────────────┘                       │
│                                  │                                              │
│                    ┌─────────────┴─────────────┐                                │
│                    │   psm.indramind.cyblab    │                                │
│                    │   (DNS Round Robin HA)    │                                │
│                    └───────────────────────────┘                                │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 1.4 Component Roles

| Server | Role | Components | Notes |
|--------|------|------------|-------|
| DC01 | Domain Controller | AD DS, DNS, LDAPS | Identity source |
| Connector01 | Session Manager (Primary) | PSM, SIA | No CPM - dedicated PSM |
| Connector02 | Session Manager (Secondary) | PSM, SIA | No CPM - dedicated PSM |

> **Important:** The PSM connectors are dedicated session manager servers. CPM (Central Policy Manager) is managed by CyberArk in the Privilege Cloud SaaS offering.

---

## 2. Prerequisites and Planning

### 2.1 CyberArk Privilege Cloud Tenant

Before beginning installation, ensure you have:

| Requirement | Details | Status |
|-------------|---------|--------|
| Privilege Cloud Tenant | Active subscription with tenant URL | ☐ |
| CyberArk Identity Tenant | Associated Identity tenant | ☐ |
| Installation User | Account with Vault Admin permissions | ☐ |
| Connector Package | Downloaded from Privilege Cloud portal | ☐ |
| License Allocation | Sufficient user licenses | ☐ |

**Obtain Tenant URLs:**
```
Privilege Cloud:  https://[tenant-id].privilegecloud.cyberark.cloud
Identity Admin:   https://[tenant-id].id.cyberark.cloud/admin
Identity Portal:  https://[tenant-id].id.cyberark.cloud
```

### 2.2 Network Requirements

#### 2.2.1 Firewall Rules (Outbound from On-Premises)

| Source | Destination | Port | Protocol | Purpose |
|--------|-------------|------|----------|---------|
| Connector01, Connector02 | *.cyberark.cloud | 443 | HTTPS | Cloud connectivity |
| Connector01, Connector02 | *.amazonaws.com | 443 | HTTPS | AWS backend services |
| Connector01, Connector02 | DC01 | 389 | LDAP | Directory queries |
| Connector01, Connector02 | DC01 | 636 | LDAPS | Secure directory queries |
| Connector01, Connector02 | DC01 | 88 | Kerberos | Authentication |
| Connector01, Connector02 | DC01 | 445 | SMB | File sharing |
| Connector01, Connector02 | Target Systems | 22, 3389, etc. | Various | Session connections |

#### 2.2.2 Firewall Rules (Internal)

| Source | Destination | Port | Protocol | Purpose |
|--------|-------------|------|----------|---------|
| Admin Workstations | Connector01, Connector02 | 3389 | RDP | Server management |
| Connector01 | Connector02 | All | Any | HA communication |
| Connector02 | Connector01 | All | Any | HA communication |

#### 2.2.3 DNS Requirements

| Record Type | Name | Value | Purpose |
|-------------|------|-------|---------|
| A | psm.indramind.cyblab | 10.10.3.8 | PSM HA (Connector01) |
| A | psm.indramind.cyblab | 10.10.3.70 | PSM HA (Connector02) |
| A | connector01.indramind.cyblab | 10.10.3.8 | Direct access |
| A | connector02.indramind.cyblab | 10.10.3.70 | Direct access |

### 2.3 Server Requirements

#### 2.3.1 Domain Controller (DC01)

| Specification | Minimum | Recommended |
|---------------|---------|-------------|
| Operating System | Windows Server 2019 | Windows Server 2022 |
| vCPU | 2 | 4 |
| RAM | 4 GB | 8 GB |
| Disk (OS) | 60 GB | 100 GB |
| Network | 1 Gbps | 1 Gbps |

**Roles Required:**
- Active Directory Domain Services (AD DS)
- DNS Server
- (Optional) Active Directory Certificate Services for LDAPS

#### 2.3.2 PSM/SIA Connector Servers (Connector01, Connector02)

| Specification | Minimum | Recommended |
|---------------|---------|-------------|
| Operating System | Windows Server 2019 | Windows Server 2022 |
| vCPU | 4 | 8 |
| RAM | 8 GB | 16 GB |
| Disk (OS) | 80 GB | 120 GB |
| Disk (Recordings) | 200 GB | 500 GB |
| Network | 1 Gbps | 10 Gbps |

**Pre-installed Roles:**
- Remote Desktop Session Host (RDS)
- .NET Framework 4.8

### 2.4 Active Directory Requirements

| Requirement | Details |
|-------------|---------|
| Domain Functional Level | Windows Server 2012 R2 or higher |
| Forest Functional Level | Windows Server 2012 R2 or higher |
| LDAP Connectivity | Port 389 (LDAP) or 636 (LDAPS) |
| Service Account | Read permissions on user/group objects |

### 2.5 Account Requirements

| Account | Purpose | Permissions Required |
|---------|---------|---------------------|
| Installation Admin | Privilege Cloud setup | Vault Admin in Privilege Cloud |
| AD Service Account | CyberArk Identity connector | Read access to OU=PAM |
| PSM Service Account | PSM Windows service | Local admin on PSM servers |
| Domain Admin | AD configuration | Domain Admins (temporary) |

### 2.6 Pre-Installation Checklist

Complete all items before proceeding:

```
INFRASTRUCTURE
☐ All servers provisioned and accessible
☐ Windows Server 2022 installed on all servers
☐ Servers joined to indramind.cyblab domain
☐ Windows Updates applied
☐ Antivirus exclusions configured for CyberArk paths
☐ Time synchronization configured (NTP)

NETWORK
☐ Firewall rules implemented
☐ DNS records created
☐ Network connectivity verified between all servers
☐ Cloud connectivity verified (443 to *.cyberark.cloud)

ACTIVE DIRECTORY
☐ Domain functional level verified
☐ Domain Admin credentials available
☐ LDAP/LDAPS connectivity confirmed

CYBERARK CLOUD
☐ Tenant URLs documented
☐ Installation admin credentials available
☐ Connector package downloaded
☐ License count verified

DOCUMENTATION
☐ IP addressing scheme documented
☐ Server naming convention confirmed
☐ Change management approval obtained
☐ Rollback plan prepared
```

---

## 3. Infrastructure Preparation

### 3.1 Verify Server Connectivity

Execute on each server to validate network readiness.

#### 3.1.1 Test Cloud Connectivity (Connector01 & Connector02)

```powershell
# Test CyberArk Cloud connectivity
$CloudEndpoints = @(
    "connector.privilegecloud.cyberark.cloud",
    "console.privilegecloud.cyberark.cloud",
    "webaccess.privilegecloud.cyberark.cloud"
)

Write-Host "Testing CyberArk Cloud Connectivity" -ForegroundColor Cyan
Write-Host "=====================================" -ForegroundColor Cyan

foreach ($Endpoint in $CloudEndpoints) {
    $Result = Test-NetConnection -ComputerName $Endpoint -Port 443 -WarningAction SilentlyContinue
    if ($Result.TcpTestSucceeded) {
        Write-Host "[PASS] $Endpoint :443" -ForegroundColor Green
    } else {
        Write-Host "[FAIL] $Endpoint :443" -ForegroundColor Red
    }
}
```

**Expected Output:**
```
Testing CyberArk Cloud Connectivity
=====================================
[PASS] connector.privilegecloud.cyberark.cloud :443
[PASS] console.privilegecloud.cyberark.cloud :443
[PASS] webaccess.privilegecloud.cyberark.cloud :443
```

> **If any test fails:** Check firewall rules, proxy settings, and DNS resolution before proceeding.

#### 3.1.2 Test Domain Controller Connectivity

```powershell
# Test DC connectivity from Connector servers
$DomainController = "DC01.indramind.cyblab"
$Ports = @(
    @{ Port = 389; Service = "LDAP" },
    @{ Port = 636; Service = "LDAPS" },
    @{ Port = 88; Service = "Kerberos" },
    @{ Port = 445; Service = "SMB" },
    @{ Port = 53; Service = "DNS" }
)

Write-Host "`nTesting Domain Controller Connectivity: $DomainController" -ForegroundColor Cyan
Write-Host "=========================================================" -ForegroundColor Cyan

foreach ($PortInfo in $Ports) {
    $Result = Test-NetConnection -ComputerName $DomainController -Port $PortInfo.Port -WarningAction SilentlyContinue
    if ($Result.TcpTestSucceeded) {
        Write-Host "[PASS] $($PortInfo.Service) :$($PortInfo.Port)" -ForegroundColor Green
    } else {
        Write-Host "[FAIL] $($PortInfo.Service) :$($PortInfo.Port)" -ForegroundColor Red
    }
}
```

#### 3.1.3 Verify Domain Membership

```powershell
# Verify server is domain-joined
$ComputerSystem = Get-WmiObject -Class Win32_ComputerSystem

Write-Host "`nDomain Membership Status" -ForegroundColor Cyan
Write-Host "========================" -ForegroundColor Cyan
Write-Host "Computer Name:    $($ComputerSystem.Name)"
Write-Host "Domain:           $($ComputerSystem.Domain)"
Write-Host "Part of Domain:   $($ComputerSystem.PartOfDomain)"

if (-not $ComputerSystem.PartOfDomain) {
    Write-Host "`n[ERROR] Server is not domain-joined. Join to indramind.cyblab before proceeding." -ForegroundColor Red
    exit 1
}

if ($ComputerSystem.Domain -ne "indramind.cyblab") {
    Write-Host "`n[WARNING] Server is joined to $($ComputerSystem.Domain), expected indramind.cyblab" -ForegroundColor Yellow
}
```

### 3.2 Prepare Connector Servers

Execute on **Connector01** and **Connector02**.

#### 3.2.1 Install Required Windows Features

```powershell
# Install Remote Desktop Session Host and required features
Write-Host "Installing Windows Features for PSM..." -ForegroundColor Cyan

$Features = @(
    "RDS-RD-Server",           # Remote Desktop Session Host
    "NET-Framework-45-Core",   # .NET Framework 4.5
    "RSAT-AD-PowerShell"       # AD PowerShell module
)

foreach ($Feature in $Features) {
    $Installed = Get-WindowsFeature -Name $Feature
    if ($Installed.Installed) {
        Write-Host "[EXISTS] $Feature" -ForegroundColor Yellow
    } else {
        Write-Host "[INSTALLING] $Feature..." -ForegroundColor Cyan
        Install-WindowsFeature -Name $Feature -IncludeManagementTools
        Write-Host "[INSTALLED] $Feature" -ForegroundColor Green
    }
}

# Check if reboot is required
$RebootRequired = (Get-WindowsFeature -Name $Features[0]).InstallState -eq "InstallPending"
if ($RebootRequired) {
    Write-Host "`n[ACTION REQUIRED] Server reboot required. Reboot and re-run this section." -ForegroundColor Yellow
}
```

#### 3.2.2 Configure Remote Desktop Settings

```powershell
# Configure RDP settings for PSM
Write-Host "`nConfiguring Remote Desktop Settings..." -ForegroundColor Cyan

# Enable Remote Desktop
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0

# Enable Network Level Authentication
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name "UserAuthentication" -Value 1

# Configure RDP Port (default 3389)
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name "PortNumber" -Value 3389

# Enable firewall rule
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"

Write-Host "[CONFIGURED] Remote Desktop settings applied" -ForegroundColor Green
```

#### 3.2.3 Create Recording Directory

```powershell
# Create directory for PSM session recordings
$RecordingPath = "D:\PSMRecordings"

if (-not (Test-Path $RecordingPath)) {
    New-Item -Path $RecordingPath -ItemType Directory -Force | Out-Null
    Write-Host "[CREATED] $RecordingPath" -ForegroundColor Green
} else {
    Write-Host "[EXISTS] $RecordingPath" -ForegroundColor Yellow
}

# Set NTFS permissions (PSM service account will need access)
$Acl = Get-Acl $RecordingPath
Write-Host "[INFO] Current permissions on $RecordingPath :"
$Acl.Access | Format-Table IdentityReference, FileSystemRights, AccessControlType -AutoSize
```

#### 3.2.4 Configure Windows Defender Exclusions

```powershell
# Add Windows Defender exclusions for CyberArk paths
Write-Host "`nConfiguring Windows Defender Exclusions..." -ForegroundColor Cyan

$ExclusionPaths = @(
    "C:\Program Files (x86)\CyberArk",
    "C:\Program Files\CyberArk",
    "D:\PSMRecordings"
)

$ExclusionProcesses = @(
    "PSMConsole.exe",
    "PSMDriver.exe",
    "CyberArk.TransparentConnection.BL.exe"
)

foreach ($Path in $ExclusionPaths) {
    try {
        Add-MpPreference -ExclusionPath $Path -ErrorAction Stop
        Write-Host "[ADDED] Path exclusion: $Path" -ForegroundColor Green
    } catch {
        Write-Host "[WARNING] Could not add exclusion for $Path : $($_.Exception.Message)" -ForegroundColor Yellow
    }
}

foreach ($Process in $ExclusionProcesses) {
    try {
        Add-MpPreference -ExclusionProcess $Process -ErrorAction Stop
        Write-Host "[ADDED] Process exclusion: $Process" -ForegroundColor Green
    } catch {
        Write-Host "[WARNING] Could not add exclusion for $Process : $($_.Exception.Message)" -ForegroundColor Yellow
    }
}
```

#### 3.2.5 Verify Server Preparation

```powershell
# Final verification script
Write-Host "`n" -ForegroundColor Cyan
Write-Host "╔════════════════════════════════════════════════════════════════╗" -ForegroundColor Cyan
Write-Host "║           CONNECTOR SERVER PREPARATION SUMMARY                 ║" -ForegroundColor Cyan
Write-Host "╚════════════════════════════════════════════════════════════════╝" -ForegroundColor Cyan

# Check Windows Features
Write-Host "`n[1] Windows Features:" -ForegroundColor White
$RDSInstalled = (Get-WindowsFeature -Name RDS-RD-Server).Installed
$NETInstalled = (Get-WindowsFeature -Name NET-Framework-45-Core).Installed
Write-Host "    RDS Session Host:     $(if($RDSInstalled){'[OK]'}else{'[MISSING]'})" -ForegroundColor $(if($RDSInstalled){'Green'}else{'Red'})
Write-Host "    .NET Framework:       $(if($NETInstalled){'[OK]'}else{'[MISSING]'})" -ForegroundColor $(if($NETInstalled){'Green'}else{'Red'})

# Check RDP
Write-Host "`n[2] Remote Desktop:" -ForegroundColor White
$RDPEnabled = (Get-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server').fDenyTSConnections -eq 0
Write-Host "    RDP Enabled:          $(if($RDPEnabled){'[OK]'}else{'[DISABLED]'})" -ForegroundColor $(if($RDPEnabled){'Green'}else{'Red'})

# Check Recording Directory
Write-Host "`n[3] Recording Directory:" -ForegroundColor White
$RecordingExists = Test-Path "D:\PSMRecordings"
Write-Host "    D:\PSMRecordings:     $(if($RecordingExists){'[OK]'}else{'[MISSING]'})" -ForegroundColor $(if($RecordingExists){'Green'}else{'Red'})

# Check Domain
Write-Host "`n[4] Domain Membership:" -ForegroundColor White
$Domain = (Get-WmiObject Win32_ComputerSystem).Domain
Write-Host "    Domain:               $Domain" -ForegroundColor $(if($Domain -eq 'indramind.cyblab'){'Green'}else{'Yellow'})

# Check Cloud Connectivity
Write-Host "`n[5] Cloud Connectivity:" -ForegroundColor White
$CloudTest = Test-NetConnection -ComputerName "connector.privilegecloud.cyberark.cloud" -Port 443 -WarningAction SilentlyContinue
Write-Host "    CyberArk Cloud:       $(if($CloudTest.TcpTestSucceeded){'[OK]'}else{'[FAILED]'})" -ForegroundColor $(if($CloudTest.TcpTestSucceeded){'Green'}else{'Red'})

Write-Host "`n════════════════════════════════════════════════════════════════" -ForegroundColor Cyan
```

### 3.3 Create DNS Records for PSM HA

Execute on **DC01** (DNS Server).

```powershell
# Create DNS A records for PSM High Availability
$ZoneName = "indramind.cyblab"

Write-Host "Creating DNS Records for PSM HA..." -ForegroundColor Cyan
Write-Host "====================================" -ForegroundColor Cyan

# PSM HA records (Round Robin)
$PSMRecords = @(
    @{ Name = "psm"; IP = "10.10.3.8"; Comment = "Connector01" },
    @{ Name = "psm"; IP = "10.10.3.70"; Comment = "Connector02" }
)

foreach ($Record in $PSMRecords) {
    try {
        # Check if record already exists
        $Existing = Get-DnsServerResourceRecord -ZoneName $ZoneName -Name $Record.Name -RRType A -ErrorAction SilentlyContinue |
                    Where-Object { $_.RecordData.IPv4Address.ToString() -eq $Record.IP }

        if ($Existing) {
            Write-Host "[EXISTS]  $($Record.Name).$ZoneName -> $($Record.IP) ($($Record.Comment))" -ForegroundColor Yellow
        } else {
            Add-DnsServerResourceRecordA -ZoneName $ZoneName -Name $Record.Name -IPv4Address $Record.IP -TimeToLive 00:05:00
            Write-Host "[CREATED] $($Record.Name).$ZoneName -> $($Record.IP) ($($Record.Comment))" -ForegroundColor Green
        }
    } catch {
        Write-Host "[ERROR]   Failed to create $($Record.Name) -> $($Record.IP): $($_.Exception.Message)" -ForegroundColor Red
    }
}

# Individual server records
$ServerRecords = @(
    @{ Name = "connector01"; IP = "10.10.3.8" },
    @{ Name = "connector02"; IP = "10.10.3.70" }
)

foreach ($Record in $ServerRecords) {
    try {
        $Existing = Get-DnsServerResourceRecord -ZoneName $ZoneName -Name $Record.Name -RRType A -ErrorAction SilentlyContinue

        if ($Existing) {
            Write-Host "[EXISTS]  $($Record.Name).$ZoneName -> $($Record.IP)" -ForegroundColor Yellow
        } else {
            Add-DnsServerResourceRecordA -ZoneName $ZoneName -Name $Record.Name -IPv4Address $Record.IP
            Write-Host "[CREATED] $($Record.Name).$ZoneName -> $($Record.IP)" -ForegroundColor Green
        }
    } catch {
        Write-Host "[ERROR]   Failed to create $($Record.Name): $($_.Exception.Message)" -ForegroundColor Red
    }
}

# Verify DNS resolution
Write-Host "`nVerifying DNS Resolution..." -ForegroundColor Cyan
Write-Host "============================" -ForegroundColor Cyan

$TestNames = @("psm.indramind.cyblab", "connector01.indramind.cyblab", "connector02.indramind.cyblab")

foreach ($Name in $TestNames) {
    try {
        $Resolution = Resolve-DnsName -Name $Name -Type A -ErrorAction Stop
        foreach ($R in $Resolution) {
            Write-Host "[OK] $Name -> $($R.IPAddress)" -ForegroundColor Green
        }
    } catch {
        Write-Host "[FAIL] Cannot resolve $Name" -ForegroundColor Red
    }
}
```

**Verification - DNS Round Robin:**

```powershell
# Test DNS round robin by resolving multiple times
Write-Host "`nTesting DNS Round Robin (psm.indramind.cyblab):" -ForegroundColor Cyan
1..6 | ForEach-Object {
    $Result = Resolve-DnsName -Name "psm.indramind.cyblab" -Type A -DnsOnly
    Write-Host "Query $_`: $($Result.IPAddress -join ', ')"
}
```

---

## 4. Phase 1: Active Directory Configuration

### 4.1 Overview

This phase creates the complete TIER model OU structure, security groups, and service accounts required for CyberArk PAM operations.

**Execute all commands in this section on DC01 as Domain Administrator.**

### 4.2 Create PAM Organizational Unit Structure

#### 4.2.1 Complete TIER Model Creation Script

Save this script as `C:\Scripts\Create-TierStructure.ps1` on DC01:

```powershell
#Requires -Modules ActiveDirectory
#Requires -RunAsAdministrator

<#
.SYNOPSIS
    Creates the complete CyberArk PAM TIER model OU structure.

.DESCRIPTION
    This script creates:
    - PAM root OU with Tier0, Tier1, Tier2 sub-OUs
    - Security groups for each tier with appropriate naming
    - Service accounts for CyberArk components
    - Admin groups for CyberArk roles (Vault Admins, Safe Managers, etc.)
    - User OUs for active and disabled accounts

.PARAMETER DomainDN
    The domain distinguished name (e.g., "DC=indramind,DC=cyblab")

.PARAMETER Verbose
    Enable verbose output

.EXAMPLE
    .\Create-TierStructure.ps1 -DomainDN "DC=indramind,DC=cyblab"

.NOTES
    Author: Lab Team
    Version: 2.0
    Requires: ActiveDirectory PowerShell module, Domain Admin rights
#>

[CmdletBinding()]
param(
    [Parameter(Mandatory = $true, HelpMessage = "Enter the domain DN (e.g., DC=indramind,DC=cyblab)")]
    [ValidatePattern("^DC=.+")]
    [string]$DomainDN
)

# Script configuration
$ErrorActionPreference = "Stop"
$InformationPreference = "Continue"

# Logging function
function Write-Log {
    param(
        [string]$Message,
        [ValidateSet("INFO", "SUCCESS", "WARNING", "ERROR")]
        [string]$Level = "INFO"
    )

    $Timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $Color = switch ($Level) {
        "INFO"    { "White" }
        "SUCCESS" { "Green" }
        "WARNING" { "Yellow" }
        "ERROR"   { "Red" }
    }

    $Prefix = switch ($Level) {
        "INFO"    { "[INFO]   " }
        "SUCCESS" { "[OK]     " }
        "WARNING" { "[WARN]   " }
        "ERROR"   { "[ERROR]  " }
    }

    Write-Host "$Timestamp $Prefix $Message" -ForegroundColor $Color
}

# Banner
Write-Host ""
Write-Host "╔══════════════════════════════════════════════════════════════════════╗" -ForegroundColor Cyan
Write-Host "║                                                                      ║" -ForegroundColor Cyan
Write-Host "║           CYBERARK PAM - TIER MODEL STRUCTURE CREATION               ║" -ForegroundColor Cyan
Write-Host "║                                                                      ║" -ForegroundColor Cyan
Write-Host "╚══════════════════════════════════════════════════════════════════════╝" -ForegroundColor Cyan
Write-Host ""
Write-Log "Target Domain: $DomainDN" "INFO"
Write-Host ""

# ═══════════════════════════════════════════════════════════════════════════════
# SECTION 1: ORGANIZATIONAL UNITS
# ═══════════════════════════════════════════════════════════════════════════════

Write-Host "┌──────────────────────────────────────────────────────────────────────┐" -ForegroundColor Cyan
Write-Host "│ SECTION 1: Creating Organizational Units                             │" -ForegroundColor Cyan
Write-Host "└──────────────────────────────────────────────────────────────────────┘" -ForegroundColor Cyan

$OUs = @(
    # Root PAM OU
    @{ Name = "PAM"; Path = $DomainDN; Description = "CyberArk Privileged Access Management" }

    # Tier 0 - Domain Controllers and Forest Admins
    @{ Name = "Tier0"; Path = "OU=PAM,$DomainDN"; Description = "Tier 0 - Domain Controllers and Forest Administrators" }
    @{ Name = "Groups"; Path = "OU=Tier0,OU=PAM,$DomainDN"; Description = "Tier 0 Security Groups" }
    @{ Name = "Accounts"; Path = "OU=Tier0,OU=PAM,$DomainDN"; Description = "Tier 0 Privileged Accounts" }

    # Tier 1 - Enterprise Servers
    @{ Name = "Tier1"; Path = "OU=PAM,$DomainDN"; Description = "Tier 1 - Enterprise Servers" }
    @{ Name = "Groups"; Path = "OU=Tier1,OU=PAM,$DomainDN"; Description = "Tier 1 Security Groups" }
    @{ Name = "Accounts"; Path = "OU=Tier1,OU=PAM,$DomainDN"; Description = "Tier 1 Privileged Accounts" }

    # Tier 2 - Workstations
    @{ Name = "Tier2"; Path = "OU=PAM,$DomainDN"; Description = "Tier 2 - Workstations" }
    @{ Name = "Groups"; Path = "OU=Tier2,OU=PAM,$DomainDN"; Description = "Tier 2 Security Groups" }
    @{ Name = "Accounts"; Path = "OU=Tier2,OU=PAM,$DomainDN"; Description = "Tier 2 Privileged Accounts" }

    # Service Accounts
    @{ Name = "ServiceAccounts"; Path = "OU=PAM,$DomainDN"; Description = "CyberArk Service Accounts" }

    # Admin Groups
    @{ Name = "AdminGroups"; Path = "OU=PAM,$DomainDN"; Description = "CyberArk Administrative Groups" }

    # Users
    @{ Name = "Users"; Path = "OU=PAM,$DomainDN"; Description = "Lab Team User Accounts" }
    @{ Name = "Active"; Path = "OU=Users,OU=PAM,$DomainDN"; Description = "Active Team Members" }
    @{ Name = "Disabled"; Path = "OU=Users,OU=PAM,$DomainDN"; Description = "Disabled/Departed Accounts" }
)

$OUCreated = 0
$OUExisted = 0
$OUFailed = 0

foreach ($OU in $OUs) {
    $FullDN = "OU=$($OU.Name),$($OU.Path)"
    try {
        $Existing = Get-ADOrganizationalUnit -Filter "DistinguishedName -eq '$FullDN'" -ErrorAction SilentlyContinue

        if ($Existing) {
            Write-Log "OU exists: $FullDN" "WARNING"
            $OUExisted++
        } else {
            New-ADOrganizationalUnit -Name $OU.Name `
                                      -Path $OU.Path `
                                      -Description $OU.Description `
                                      -ProtectedFromAccidentalDeletion $true
            Write-Log "Created OU: $FullDN" "SUCCESS"
            $OUCreated++
        }
    } catch {
        Write-Log "Failed to create OU $FullDN : $($_.Exception.Message)" "ERROR"
        $OUFailed++
    }
}

Write-Host ""
Write-Log "OUs Summary: $OUCreated created, $OUExisted existed, $OUFailed failed" "INFO"
Write-Host ""

# ═══════════════════════════════════════════════════════════════════════════════
# SECTION 2: SECURITY GROUPS
# ═══════════════════════════════════════════════════════════════════════════════

Write-Host "┌──────────────────────────────────────────────────────────────────────┐" -ForegroundColor Cyan
Write-Host "│ SECTION 2: Creating Security Groups                                  │" -ForegroundColor Cyan
Write-Host "└──────────────────────────────────────────────────────────────────────┘" -ForegroundColor Cyan

$Groups = @(
    # Tier 0 Groups
    @{
        Name = "PAM-T0-Admins"
        Path = "OU=Groups,OU=Tier0,OU=PAM,$DomainDN"
        Description = "Tier 0 Administrators - Full Domain Controller access"
        Scope = "Global"
        Category = "Security"
    }
    @{
        Name = "PAM-T0-Operators"
        Path = "OU=Groups,OU=Tier0,OU=PAM,$DomainDN"
        Description = "Tier 0 Operators - Limited Domain Controller operations"
        Scope = "Global"
        Category = "Security"
    }
    @{
        Name = "PAM-T0-Auditors"
        Path = "OU=Groups,OU=Tier0,OU=PAM,$DomainDN"
        Description = "Tier 0 Auditors - Read-only Domain Controller access"
        Scope = "Global"
        Category = "Security"
    }

    # Tier 1 Groups
    @{
        Name = "PAM-T1-Admins"
        Path = "OU=Groups,OU=Tier1,OU=PAM,$DomainDN"
        Description = "Tier 1 Administrators - Enterprise server administration"
        Scope = "Global"
        Category = "Security"
    }
    @{
        Name = "PAM-T1-WindowsAdmins"
        Path = "OU=Groups,OU=Tier1,OU=PAM,$DomainDN"
        Description = "Tier 1 Windows Server Administrators"
        Scope = "Global"
        Category = "Security"
    }
    @{
        Name = "PAM-T1-LinuxAdmins"
        Path = "OU=Groups,OU=Tier1,OU=PAM,$DomainDN"
        Description = "Tier 1 Linux/Unix Server Administrators"
        Scope = "Global"
        Category = "Security"
    }
    @{
        Name = "PAM-T1-DatabaseAdmins"
        Path = "OU=Groups,OU=Tier1,OU=PAM,$DomainDN"
        Description = "Tier 1 Database Administrators (DBA)"
        Scope = "Global"
        Category = "Security"
    }
    @{
        Name = "PAM-T1-Operators"
        Path = "OU=Groups,OU=Tier1,OU=PAM,$DomainDN"
        Description = "Tier 1 Operators - Limited server access"
        Scope = "Global"
        Category = "Security"
    }

    # Tier 2 Groups
    @{
        Name = "PAM-T2-Admins"
        Path = "OU=Groups,OU=Tier2,OU=PAM,$DomainDN"
        Description = "Tier 2 Administrators - Workstation administration"
        Scope = "Global"
        Category = "Security"
    }
    @{
        Name = "PAM-T2-HelpDesk"
        Path = "OU=Groups,OU=Tier2,OU=PAM,$DomainDN"
        Description = "Tier 2 Help Desk - Limited workstation access"
        Scope = "Global"
        Category = "Security"
    }

    # CyberArk Admin Groups
    @{
        Name = "PAM-Vault-Admins"
        Path = "OU=AdminGroups,OU=PAM,$DomainDN"
        Description = "CyberArk Vault Administrators - Full vault access"
        Scope = "Global"
        Category = "Security"
    }
    @{
        Name = "PAM-Safe-Managers"
        Path = "OU=AdminGroups,OU=PAM,$DomainDN"
        Description = "CyberArk Safe Managers - Safe creation and management"
        Scope = "Global"
        Category = "Security"
    }
    @{
        Name = "PAM-Auditors"
        Path = "OU=AdminGroups,OU=PAM,$DomainDN"
        Description = "CyberArk Auditors - Compliance and audit access"
        Scope = "Global"
        Category = "Security"
    }
    @{
        Name = "PAM-Users"
        Path = "OU=AdminGroups,OU=PAM,$DomainDN"
        Description = "CyberArk Standard Users - Basic privileged access"
        Scope = "Global"
        Category = "Security"
    }
    @{
        Name = "PAM-Approvers"
        Path = "OU=AdminGroups,OU=PAM,$DomainDN"
        Description = "CyberArk Access Request Approvers"
        Scope = "Global"
        Category = "Security"
    }
)

$GroupCreated = 0
$GroupExisted = 0
$GroupFailed = 0

foreach ($Group in $Groups) {
    try {
        $Existing = Get-ADGroup -Filter "Name -eq '$($Group.Name)'" -ErrorAction SilentlyContinue

        if ($Existing) {
            Write-Log "Group exists: $($Group.Name)" "WARNING"
            $GroupExisted++
        } else {
            New-ADGroup -Name $Group.Name `
                        -Path $Group.Path `
                        -GroupScope $Group.Scope `
                        -GroupCategory $Group.Category `
                        -Description $Group.Description
            Write-Log "Created group: $($Group.Name)" "SUCCESS"
            $GroupCreated++
        }
    } catch {
        Write-Log "Failed to create group $($Group.Name) : $($_.Exception.Message)" "ERROR"
        $GroupFailed++
    }
}

Write-Host ""
Write-Log "Groups Summary: $GroupCreated created, $GroupExisted existed, $GroupFailed failed" "INFO"
Write-Host ""

# ═══════════════════════════════════════════════════════════════════════════════
# SECTION 3: SERVICE ACCOUNTS
# ═══════════════════════════════════════════════════════════════════════════════

Write-Host "┌──────────────────────────────────────────────────────────────────────┐" -ForegroundColor Cyan
Write-Host "│ SECTION 3: Creating Service Accounts                                 │" -ForegroundColor Cyan
Write-Host "└──────────────────────────────────────────────────────────────────────┘" -ForegroundColor Cyan

# Extract domain name for UPN suffix
$DomainName = ($DomainDN -replace "DC=", "" -replace ",", ".").ToLower()

$ServiceAccounts = @(
    @{
        Name = "svc-CyberArkIdentity"
        Description = "CyberArk Identity AD Connector Service Account"
        Purpose = "LDAP/LDAPS queries for user/group synchronization"
    }
    @{
        Name = "svc-CyberArkPSM"
        Description = "CyberArk PSM Service Account"
        Purpose = "Privileged Session Manager Windows service"
    }
    @{
        Name = "svc-CyberArkScan"
        Description = "CyberArk Scanner Service Account"
        Purpose = "Account discovery and scanning operations"
    }
    @{
        Name = "svc-CyberArkReconcile"
        Description = "CyberArk Reconciliation Service Account"
        Purpose = "Password reconciliation and verification"
    }
)

$ServiceAccountPath = "OU=ServiceAccounts,OU=PAM,$DomainDN"
$SvcCreated = 0
$SvcExisted = 0
$SvcFailed = 0
$GeneratedPasswords = @()

foreach ($Svc in $ServiceAccounts) {
    try {
        $Existing = Get-ADUser -Filter "SamAccountName -eq '$($Svc.Name)'" -ErrorAction SilentlyContinue

        if ($Existing) {
            Write-Log "Service account exists: $($Svc.Name)" "WARNING"
            $SvcExisted++
        } else {
            # Generate secure password (32 chars + special requirements)
            $Password = -join (
                (65..90 | Get-Random -Count 8 | ForEach-Object { [char]$_ }) +   # Uppercase
                (97..122 | Get-Random -Count 8 | ForEach-Object { [char]$_ }) +  # Lowercase
                (48..57 | Get-Random -Count 8 | ForEach-Object { [char]$_ }) +   # Numbers
                ('!@#$%^&*()'.ToCharArray() | Get-Random -Count 4)                # Special
            ) | Sort-Object { Get-Random } | Join-String

            $SecurePassword = ConvertTo-SecureString -String $Password -AsPlainText -Force

            New-ADUser -Name $Svc.Name `
                       -SamAccountName $Svc.Name `
                       -UserPrincipalName "$($Svc.Name)@$DomainName" `
                       -Path $ServiceAccountPath `
                       -AccountPassword $SecurePassword `
                       -Enabled $true `
                       -PasswordNeverExpires $true `
                       -CannotChangePassword $true `
                       -Description $Svc.Description

            Write-Log "Created service account: $($Svc.Name)" "SUCCESS"
            $GeneratedPasswords += @{
                Account = $Svc.Name
                Password = $Password
                Purpose = $Svc.Purpose
            }
            $SvcCreated++
        }
    } catch {
        Write-Log "Failed to create service account $($Svc.Name) : $($_.Exception.Message)" "ERROR"
        $SvcFailed++
    }
}

Write-Host ""
Write-Log "Service Accounts Summary: $SvcCreated created, $SvcExisted existed, $SvcFailed failed" "INFO"

# Output generated passwords
if ($GeneratedPasswords.Count -gt 0) {
    Write-Host ""
    Write-Host "╔══════════════════════════════════════════════════════════════════════╗" -ForegroundColor Yellow
    Write-Host "║  IMPORTANT: SAVE THESE PASSWORDS SECURELY - SHOWN ONLY ONCE          ║" -ForegroundColor Yellow
    Write-Host "╚══════════════════════════════════════════════════════════════════════╝" -ForegroundColor Yellow
    Write-Host ""

    foreach ($Cred in $GeneratedPasswords) {
        Write-Host "Account:  $($Cred.Account)@$DomainName" -ForegroundColor Cyan
        Write-Host "Password: $($Cred.Password)" -ForegroundColor White
        Write-Host "Purpose:  $($Cred.Purpose)" -ForegroundColor Gray
        Write-Host ""
    }

    # Export to secure file (optional)
    $ExportPath = "C:\Scripts\ServiceAccountCredentials_$(Get-Date -Format 'yyyyMMdd_HHmmss').txt"
    $GeneratedPasswords | ForEach-Object {
        "$($_.Account)@$DomainName`t$($_.Password)`t$($_.Purpose)"
    } | Out-File -FilePath $ExportPath -Encoding UTF8
    Write-Log "Credentials exported to: $ExportPath" "INFO"
    Write-Host ""
    Write-Host "DELETE THIS FILE AFTER STORING PASSWORDS IN A SECURE LOCATION!" -ForegroundColor Red
}

# ═══════════════════════════════════════════════════════════════════════════════
# FINAL SUMMARY
# ═══════════════════════════════════════════════════════════════════════════════

Write-Host ""
Write-Host "╔══════════════════════════════════════════════════════════════════════╗" -ForegroundColor Green
Write-Host "║           TIER MODEL STRUCTURE CREATION COMPLETED                    ║" -ForegroundColor Green
Write-Host "╚══════════════════════════════════════════════════════════════════════╝" -ForegroundColor Green
Write-Host ""
Write-Host "Summary:" -ForegroundColor Cyan
Write-Host "  Organizational Units: $OUCreated created, $OUExisted existed, $OUFailed failed"
Write-Host "  Security Groups:      $GroupCreated created, $GroupExisted existed, $GroupFailed failed"
Write-Host "  Service Accounts:     $SvcCreated created, $SvcExisted existed, $SvcFailed failed"
Write-Host ""
Write-Host "Next Steps:" -ForegroundColor Cyan
Write-Host "  1. Store service account passwords in CyberArk or secure password manager"
Write-Host "  2. Configure CyberArk Identity AD connector using svc-CyberArkIdentity"
Write-Host "  3. Configure PSM service using svc-CyberArkPSM"
Write-Host "  4. Add lab team members to appropriate PAM groups"
Write-Host ""
```

#### 4.2.2 Execute the Script

```powershell
# Navigate to scripts directory
Set-Location C:\Scripts

# Execute with domain DN
.\Create-TierStructure.ps1 -DomainDN "DC=indramind,DC=cyblab"
```

### 4.3 Verify AD Structure

```powershell
# Comprehensive verification of AD structure
Write-Host "`nVerifying PAM OU Structure..." -ForegroundColor Cyan
Write-Host "==============================" -ForegroundColor Cyan

# List all OUs
Write-Host "`n[1] Organizational Units:" -ForegroundColor White
Get-ADOrganizationalUnit -Filter * -SearchBase "OU=PAM,DC=indramind,DC=cyblab" -Properties Description |
    Select-Object Name, @{N='Path';E={($_.DistinguishedName -split ',',2)[1]}}, Description |
    Format-Table -AutoSize

# List all Groups
Write-Host "`n[2] Security Groups:" -ForegroundColor White
Get-ADGroup -Filter * -SearchBase "OU=PAM,DC=indramind,DC=cyblab" -Properties Description |
    Select-Object Name, GroupScope, @{N='Location';E={($_.DistinguishedName -split ',',2)[1] -replace ',DC=indramind,DC=cyblab',''}} |
    Format-Table -AutoSize

# List Service Accounts
Write-Host "`n[3] Service Accounts:" -ForegroundColor White
Get-ADUser -Filter * -SearchBase "OU=ServiceAccounts,OU=PAM,DC=indramind,DC=cyblab" -Properties Description, Enabled |
    Select-Object Name, Enabled, Description |
    Format-Table -AutoSize
```

### 4.4 Grant AD Connector Permissions

The CyberArk Identity service account needs read permissions on the PAM OU.

```powershell
# Grant read permissions to the Identity connector service account
$ServiceAccount = "svc-CyberArkIdentity"
$TargetOU = "OU=PAM,DC=indramind,DC=cyblab"

Write-Host "Granting read permissions for $ServiceAccount on $TargetOU" -ForegroundColor Cyan

# Get the service account SID
$User = Get-ADUser -Identity $ServiceAccount
$UserSID = New-Object System.Security.Principal.SecurityIdentifier($User.SID)

# Get the OU
$OU = Get-ADOrganizationalUnit -Identity $TargetOU

# Get current ACL
$ACL = Get-Acl -Path "AD:\$($OU.DistinguishedName)"

# Create access rule for Read access
$AccessRule = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
    $UserSID,
    "GenericRead",
    "Allow",
    "Descendents"
)

# Add the rule
$ACL.AddAccessRule($AccessRule)

# Apply the ACL
Set-Acl -Path "AD:\$($OU.DistinguishedName)" -AclObject $ACL

Write-Host "[OK] Read permissions granted to $ServiceAccount" -ForegroundColor Green
```

---

## 5. Phase 2: CyberArk Identity Setup

### 5.1 Overview

This phase configures CyberArk Identity with Active Directory as the primary identity provider.

### 5.2 Access CyberArk Identity Admin Portal

1. **Open Browser** (Chrome or Edge recommended)

2. **Navigate to Admin Portal:**
   ```
   URL: https://[your-tenant-id].id.cyberark.cloud/admin
   ```

3. **Login with Installation Admin Credentials**

4. **Verify Access:**
   - You should see the Identity Administration dashboard
   - Confirm you have "Admin" role in top-right user menu

### 5.3 Configure AD Directory Service

#### 5.3.1 Navigate to Directory Services

```
Settings → Core Services → Directory Services
```

#### 5.3.2 Add Active Directory

Click **"Add Directory"** and select **"Active Directory"**

#### 5.3.3 Configure Connection Settings

| Setting | Value | Notes |
|---------|-------|-------|
| **Directory Name** | LAB-AD | Descriptive name for this directory |
| **Forest** | indramind.cyblab | AD forest name |

**Connection Details:**

| Setting | Value |
|---------|-------|
| **Server Address** | DC01.indramind.cyblab |
| **Port** | 636 |
| **Use SSL** | Yes (checked) |
| **Base DN** | DC=indramind,DC=cyblab |

**Bind Credentials:**

| Setting | Value |
|---------|-------|
| **Bind DN** | CN=svc-CyberArkIdentity,OU=ServiceAccounts,OU=PAM,DC=indramind,DC=cyblab |
| **Password** | [Password from script output] |

#### 5.3.4 Configure Sync Settings

| Setting | Value | Notes |
|---------|-------|-------|
| **User Container** | OU=PAM,DC=indramind,DC=cyblab | Sync users from PAM OU |
| **Group Container** | OU=PAM,DC=indramind,DC=cyblab | Sync groups from PAM OU |
| **Sync Interval** | 15 minutes | Automatic sync frequency |
| **Sync Nested Groups** | Yes | Include nested group memberships |

#### 5.3.5 Configure User Sync Filter (Recommended)

Click **"Advanced"** and add LDAP filter:

```ldap
(&(objectClass=user)(objectCategory=person)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))
```

This filter:
- Syncs only user objects (`objectClass=user`)
- Excludes computer accounts (`objectCategory=person`)
- Excludes disabled accounts (`userAccountControl` bit check)

#### 5.3.6 Test and Save Configuration

1. Click **"Test Connection"**

   **Expected Result:**
   ```
   Connection Successful
   Found X users and Y groups
   ```

2. If successful, click **"Save"**

3. Click **"Sync Now"** to perform initial synchronization

### 5.4 Verify Directory Synchronization

#### 5.4.1 Check Sync Status

```
Settings → Core Services → Directory Services → LAB-AD
```

Verify:
- **Status:** Connected
- **Last Sync:** [Recent timestamp]
- **Users Synced:** [Number > 0]
- **Groups Synced:** [Number > 0]

#### 5.4.2 Verify Users Imported

```
Users → All Users → Filter: Directory = "LAB-AD"
```

Confirm that service accounts and any test users appear.

#### 5.4.3 Verify Groups Imported

```
Roles → Directory Groups
```

Confirm PAM groups are visible:
- PAM-Vault-Admins
- PAM-Safe-Managers
- PAM-Auditors
- PAM-Users
- PAM-Approvers
- PAM-T0-*, PAM-T1-*, PAM-T2-* groups

### 5.5 Configure Authentication Profile

#### 5.5.1 Navigate to Authentication Profiles

```
Settings → Authentication → Authentication Profiles
```

#### 5.5.2 Create AD Authentication Profile

1. Click **"Add Profile"**

2. Configure:

| Setting | Value |
|---------|-------|
| **Profile Name** | LAB-AD-Auth |
| **Description** | Active Directory authentication for lab users |
| **Directory** | LAB-AD |
| **Pass-through Authentication** | Yes |

3. **User Mapping:**

| Setting | Value |
|---------|-------|
| **Username Format** | sAMAccountName |
| **Allow UPN Login** | Yes |
| **UPN Suffix** | @indramind.cyblab |

4. Click **"Save"**

### 5.6 Test AD Authentication

#### 5.6.1 Create Test User in AD

On DC01:

```powershell
# Create test user
$TestPassword = ConvertTo-SecureString "TestP@ssw0rd123!" -AsPlainText -Force

New-ADUser -Name "Test PAMUser" `
    -SamAccountName "test.pamuser" `
    -UserPrincipalName "test.pamuser@indramind.cyblab" `
    -Path "OU=Active,OU=Users,OU=PAM,DC=indramind,DC=cyblab" `
    -AccountPassword $TestPassword `
    -Enabled $true `
    -ChangePasswordAtLogon $false `
    -Description "Test user for CyberArk Identity validation"

# Add to PAM-Users group
Add-ADGroupMember -Identity "PAM-Users" -Members "test.pamuser"

Write-Host "[CREATED] test.pamuser@indramind.cyblab" -ForegroundColor Green
```

#### 5.6.2 Force Directory Sync

In CyberArk Identity:
```
Settings → Core Services → Directory Services → LAB-AD → Sync Now
```

Wait 1-2 minutes for sync to complete.

#### 5.6.3 Test Login

1. Open **incognito/private browser window**

2. Navigate to:
   ```
   https://[your-tenant-id].id.cyberark.cloud
   ```

3. Login with:
   ```
   Username: test.pamuser@indramind.cyblab
   Password: TestP@ssw0rd123!
   ```

4. **Expected Result:** Successful login to CyberArk Identity user portal

#### 5.6.4 Verify User Details

After login, verify:
- User name displays correctly
- Group memberships visible (PAM-Users)
- Directory shows "LAB-AD"

---

## 6. Phase 3: User Management

### 6.1 Overview

This phase covers user account cleanup, email standardization, and proper group assignments.

### 6.2 Audit Existing Users

#### 6.2.1 Export Users from CyberArk Identity

1. Navigate to:
   ```
   Users → All Users
   ```

2. Click **"Export"** → Select **"CSV"**

3. Save file as `identity_users_audit_[date].csv`

#### 6.2.2 Review and Categorize Users

Create categorization spreadsheet:

| Username | Email | Source | Status | Action | Notes |
|----------|-------|--------|--------|--------|-------|
| admin | admin@tenant | Local | Active | KEEP | System admin |
| user1 | old@email | Local | Active | UPDATE | Update email |
| yago | yago@old | Local | Inactive | DELETE | Departed |
| olmedo | olmedo@old | Local | Inactive | DELETE | Departed |
| ana | ana@old | Local | Inactive | DELETE | Departed |
| test.pamuser | n/a | LAB-AD | Active | KEEP | Test account |

### 6.3 Create Domain Accounts for Team Members

On DC01:

```powershell
# Define team members to create
$TeamMembers = @(
    @{
        FirstName = "Juan"
        LastName = "Garcia"
        Email = "juan.garcia@company.com"
        Title = "Security Engineer"
        Groups = @("PAM-Users", "PAM-T1-WindowsAdmins")
    },
    @{
        FirstName = "Maria"
        LastName = "Lopez"
        Email = "maria.lopez@company.com"
        Title = "PAM Administrator"
        Groups = @("PAM-Vault-Admins", "PAM-Safe-Managers")
    },
    @{
        FirstName = "Carlos"
        LastName = "Rodriguez"
        Email = "carlos.rodriguez@company.com"
        Title = "Security Analyst"
        Groups = @("PAM-Users", "PAM-Auditors")
    }
    # Add more team members as needed
)

$DomainDN = "DC=indramind,DC=cyblab"
$UserPath = "OU=Active,OU=Users,OU=PAM,$DomainDN"

Write-Host "Creating Team Member Accounts..." -ForegroundColor Cyan
Write-Host "=================================" -ForegroundColor Cyan

foreach ($Member in $TeamMembers) {
    $Username = "$($Member.FirstName.ToLower()).$($Member.LastName.ToLower())"
    $DisplayName = "$($Member.FirstName) $($Member.LastName)"

    try {
        # Check if user exists
        $Existing = Get-ADUser -Filter "SamAccountName -eq '$Username'" -ErrorAction SilentlyContinue

        if ($Existing) {
            Write-Host "[EXISTS] $Username" -ForegroundColor Yellow
            continue
        }

        # Generate temporary password
        $TempPassword = "Welcome123!" + (Get-Random -Minimum 100 -Maximum 999)
        $SecurePassword = ConvertTo-SecureString $TempPassword -AsPlainText -Force

        # Create user
        New-ADUser -Name $DisplayName `
            -GivenName $Member.FirstName `
            -Surname $Member.LastName `
            -SamAccountName $Username `
            -UserPrincipalName "$Username@indramind.cyblab" `
            -EmailAddress $Member.Email `
            -Title $Member.Title `
            -Path $UserPath `
            -AccountPassword $SecurePassword `
            -Enabled $true `
            -ChangePasswordAtLogon $true

        # Add to groups
        foreach ($Group in $Member.Groups) {
            try {
                Add-ADGroupMember -Identity $Group -Members $Username
            } catch {
                Write-Host "  [WARN] Could not add to $Group" -ForegroundColor Yellow
            }
        }

        Write-Host "[CREATED] $Username ($DisplayName)" -ForegroundColor Green
        Write-Host "  Temp Password: $TempPassword" -ForegroundColor Gray
        Write-Host "  Groups: $($Member.Groups -join ', ')" -ForegroundColor Gray

    } catch {
        Write-Host "[ERROR] Failed to create $Username : $($_.Exception.Message)" -ForegroundColor Red
    }
}

Write-Host "`n[COMPLETE] User creation finished. Trigger AD sync in CyberArk Identity." -ForegroundColor Cyan
```

### 6.4 Disable Departed Users

```powershell
# List of departed team members to disable
$DepartedUsers = @(
    "yago",
    "olmedo",
    "ana"
    # Add other departed usernames
)

$DisabledOU = "OU=Disabled,OU=Users,OU=PAM,DC=indramind,DC=cyblab"

Write-Host "Disabling Departed User Accounts..." -ForegroundColor Cyan
Write-Host "====================================" -ForegroundColor Cyan

foreach ($Username in $DepartedUsers) {
    try {
        $User = Get-ADUser -Filter "SamAccountName -eq '$Username'" -ErrorAction SilentlyContinue

        if (-not $User) {
            Write-Host "[NOT FOUND] $Username" -ForegroundColor Yellow
            continue
        }

        # Disable account
        Disable-ADAccount -Identity $User

        # Add description noting when disabled
        Set-ADUser -Identity $User -Description "Disabled $(Get-Date -Format 'yyyy-MM-dd') - Departed"

        # Move to Disabled OU
        Move-ADObject -Identity $User.DistinguishedName -TargetPath $DisabledOU

        Write-Host "[DISABLED] $Username - Moved to Disabled OU" -ForegroundColor Green

    } catch {
        Write-Host "[ERROR] Failed to disable $Username : $($_.Exception.Message)" -ForegroundColor Red
    }
}

Write-Host "`n[COMPLETE] Departed users disabled." -ForegroundColor Cyan
```

### 6.5 Delete Local Users in CyberArk Identity

For users that were created locally in CyberArk Identity and should be removed:

1. Navigate to:
   ```
   Users → All Users
   ```

2. Filter by **Source: "CyberArk Cloud Directory"** (local users)

3. For each user to delete:
   - Click username
   - Click **"Delete User"** (trash icon)
   - Confirm deletion

> **Note:** Domain users synced from AD should be managed in AD, not deleted from Identity.

### 6.6 Sync and Verify

1. **Trigger AD Sync:**
   ```
   Settings → Core Services → Directory Services → LAB-AD → Sync Now
   ```

2. **Verify User List:**
   - New team members appear
   - Departed users no longer appear (if disabled/moved outside sync scope)
   - Group memberships correct

---

## 7. Phase 4: Multi-Factor Authentication

### 7.1 Overview

This phase configures MFA policies for two user categories:
- **Standard Users (PAM-Users):** Email OTP, SMS OTP, or Mobile App
- **High Security Users (PAM-Vault-Admins, PAM-T0-Admins):** Mobile App only

### 7.2 Configure Email OTP

#### 7.2.1 Navigate to Email OTP Settings

```
Settings → Authentication → Multi Factor Authentication → Email OTP
```

#### 7.2.2 Configure Email OTP

| Setting | Value |
|---------|-------|
| **Enable Email OTP** | Yes |
| **OTP Length** | 6 digits |
| **OTP Validity** | 5 minutes |
| **Max Attempts** | 3 |
| **Lockout Duration** | 15 minutes |

**Email Template Settings:**

| Setting | Value |
|---------|-------|
| **Sender Name** | CyberArk Lab |
| **Subject** | Your verification code |
| **Template** | Default |

Click **"Save"**

### 7.3 Configure SMS OTP

#### 7.3.1 Navigate to SMS Settings

```
Settings → Authentication → Multi Factor Authentication → SMS OTP
```

#### 7.3.2 Configure SMS OTP

| Setting | Value |
|---------|-------|
| **Enable SMS OTP** | Yes |
| **OTP Length** | 6 digits |
| **OTP Validity** | 5 minutes |
| **Message Format** | "Your CyberArk code: {OTP}" |
| **Allow International** | Yes |

Click **"Save"**

### 7.4 Configure Mobile Authenticator

#### 7.4.1 Navigate to Mobile Settings

```
Settings → Authentication → Multi Factor Authentication → Mobile Authenticator
```

#### 7.4.2 Configure Mobile Authenticator

| Setting | Value |
|---------|-------|
| **Enable Mobile Authenticator** | Yes |
| **Allow Push Notifications** | Yes |
| **Allow Offline OTP** | Yes |
| **Biometric Option** | Enabled |
| **Max Devices per User** | 2 |

Click **"Save"**

### 7.5 Create Standard MFA Policy

#### 7.5.1 Navigate to Authentication Policies

```
Access → Policies → Authentication Policies
```

#### 7.5.2 Create New Policy Set

Click **"Add Policy Set"**

#### 7.5.3 Configure LAB-2FA-Standard

**General Settings:**

| Setting | Value |
|---------|-------|
| **Policy Name** | LAB-2FA-Standard |
| **Description** | Standard 2FA policy for all lab users |
| **Status** | Active |
| **Priority** | 100 |

**Policy Scope:**

| Setting | Value |
|---------|-------|
| **Apply to** | Specific Groups |
| **Groups** | PAM-Users |

**Authentication Rules:**

| Setting | Value |
|---------|-------|
| **Primary Authentication** | Password |
| **Secondary Authentication (MFA)** | Required |

**Allowed MFA Methods:**

- [x] Email OTP
- [x] SMS OTP
- [x] Mobile Authenticator

**Challenge Settings:**

| Setting | Value |
|---------|-------|
| **Challenge Frequency** | Every login |
| **Remember Device** | Optional (7 days) |

Click **"Save"**

### 7.6 Create High Security MFA Policy

#### 7.6.1 Create New Policy Set

Click **"Add Policy Set"**

#### 7.6.2 Configure LAB-2FA-HighSecurity

**General Settings:**

| Setting | Value |
|---------|-------|
| **Policy Name** | LAB-2FA-HighSecurity |
| **Description** | High security policy for vault and T0 admins |
| **Status** | Active |
| **Priority** | 50 |

> **Important:** Lower priority number = higher precedence. This policy will take effect for qualifying users before the Standard policy.

**Policy Scope:**

| Setting | Value |
|---------|-------|
| **Apply to** | Specific Groups |
| **Groups** | PAM-Vault-Admins, PAM-T0-Admins |

**Authentication Rules:**

| Setting | Value |
|---------|-------|
| **Primary Authentication** | Password |
| **Secondary Authentication (MFA)** | Required |

**Allowed MFA Methods:**

- [ ] Email OTP (disabled)
- [ ] SMS OTP (disabled)
- [x] Mobile Authenticator ONLY

**Additional Settings:**

| Setting | Value |
|---------|-------|
| **Push Notification** | Required |
| **Biometric Verification** | Required |
| **Session Duration** | 4 hours maximum |
| **Re-authenticate on Sensitive Operations** | Yes |

Click **"Save"**

### 7.7 Verify Policy Order

Navigate to:
```
Access → Policies → Authentication Policies
```

Verify policy order:

| Priority | Policy Name | Target Groups |
|----------|-------------|---------------|
| 50 | LAB-2FA-HighSecurity | PAM-Vault-Admins, PAM-T0-Admins |
| 100 | LAB-2FA-Standard | PAM-Users |

### 7.8 Test MFA Policies

#### 7.8.1 Test Standard MFA

1. Login as a user in PAM-Users group (not in Vault-Admins or T0-Admins)
2. After password entry, verify MFA options appear:
   - Email OTP
   - SMS OTP
   - Mobile Authenticator
3. Complete MFA with any method
4. Verify successful login

#### 7.8.2 Test High Security MFA

1. Login as a user in PAM-Vault-Admins group
2. After password entry, verify:
   - **ONLY** Mobile Authenticator option appears
   - Email and SMS options are NOT available
3. Complete MFA with Mobile App
4. Verify successful login

### 7.9 Mobile App Enrollment

#### 7.9.1 User Self-Enrollment Process

Instruct users to follow these steps:

1. **Download CyberArk Identity App**
   - iOS: App Store → "CyberArk Identity"
   - Android: Play Store → "CyberArk Identity"

2. **Login to User Portal**
   ```
   https://[tenant-id].id.cyberark.cloud
   ```

3. **Navigate to Security Settings**
   ```
   User Menu (top right) → Security Settings → MFA Devices
   ```

4. **Add Mobile Device**
   - Click "Add Device"
   - Select "Mobile Authenticator"
   - QR code appears on screen

5. **Scan QR Code**
   - Open CyberArk Identity app
   - Tap "Add Account" → "Scan QR Code"
   - Point camera at QR code

6. **Verify Setup**
   - App sends test push notification
   - Approve on phone
   - "Device Added Successfully" message appears

#### 7.9.2 Admin-Assisted Enrollment

For users needing assistance:

1. Navigate to:
   ```
   Users → [Select User] → Security → MFA Devices
   ```

2. Click **"Send Enrollment Link"**

3. User receives email with enrollment instructions

---

## 8. Phase 5: PSM High Availability Deployment

### 8.1 Overview

This phase deploys PSM (Privileged Session Manager) on two dedicated connector servers in a high availability configuration.

**Architecture:**
```
                    ┌─────────────────────────────────┐
                    │    psm.indramind.cyblab         │
                    │    (DNS Round Robin)            │
                    └─────────────┬───────────────────┘
                                  │
                    ┌─────────────┴───────────────┐
                    │                             │
            ┌───────▼───────┐             ┌───────▼───────┐
            │  Connector01  │             │  Connector02  │
            │               │             │               │
            │  PSM          │             │  PSM          │
            │  SIA          │             │  SIA          │
            │               │             │               │
            │  10.10.3.8    │             │  10.10.3.70   │
            └───────────────┘             └───────────────┘
```

### 8.2 Download Connector Package

1. **Login to Privilege Cloud Portal:**
   ```
   https://[tenant-id].privilegecloud.cyberark.cloud
   ```

2. **Navigate to Connector Download:**
   ```
   Administration → Connectors → Add Connector → Download
   ```

3. **Download Package:**
   - Click "Download Connector Package"
   - Save `PrivilegeCloudConnector.exe` to accessible location

4. **Copy to Connector Servers:**
   ```
   Copy PrivilegeCloudConnector.exe to:
   - \\Connector01\C$\Installers\
   - \\Connector02\C$\Installers\
   ```

### 8.3 Install PSM on Connector01 (Primary)

#### 8.3.1 Pre-Installation Verification

On Connector01, run verification:

```powershell
# Final pre-installation check
Write-Host "PSM Installation Pre-Check - Connector01" -ForegroundColor Cyan
Write-Host "=========================================" -ForegroundColor Cyan

# Check Windows Features
$RDS = Get-WindowsFeature -Name RDS-RD-Server
Write-Host "RDS Session Host: $(if($RDS.Installed){'[OK]'}else{'[MISSING - INSTALL REQUIRED]'})" -ForegroundColor $(if($RDS.Installed){'Green'}else{'Red'})

# Check Cloud connectivity
$Cloud = Test-NetConnection -ComputerName "connector.privilegecloud.cyberark.cloud" -Port 443 -WarningAction SilentlyContinue
Write-Host "Cloud Connectivity: $(if($Cloud.TcpTestSucceeded){'[OK]'}else{'[FAILED]'})" -ForegroundColor $(if($Cloud.TcpTestSucceeded){'Green'}else{'Red'})

# Check DC connectivity
$DC = Test-NetConnection -ComputerName "DC01.indramind.cyblab" -Port 389 -WarningAction SilentlyContinue
Write-Host "DC Connectivity: $(if($DC.TcpTestSucceeded){'[OK]'}else{'[FAILED]'})" -ForegroundColor $(if($DC.TcpTestSucceeded){'Green'}else{'Red'})

# Check installer exists
$Installer = Test-Path "C:\Installers\PrivilegeCloudConnector.exe"
Write-Host "Installer Present: $(if($Installer){'[OK]'}else{'[MISSING]'})" -ForegroundColor $(if($Installer){'Green'}else{'Red'})

# Check free disk space
$Disk = Get-WmiObject Win32_LogicalDisk -Filter "DeviceID='C:'"
$FreeGB = [math]::Round($Disk.FreeSpace / 1GB, 2)
Write-Host "Free Disk Space: $FreeGB GB $(if($FreeGB -gt 20){'[OK]'}else{'[LOW]'})" -ForegroundColor $(if($FreeGB -gt 20){'Green'}else{'Yellow'})

Write-Host "`nIf all checks pass, proceed with installation." -ForegroundColor Cyan
```

#### 8.3.2 Run Installation Wizard

1. **Launch Installer as Administrator:**
   ```
   Right-click PrivilegeCloudConnector.exe → Run as Administrator
   ```

2. **Welcome Screen:**
   - Click **"Next"**

3. **License Agreement:**
   - Accept terms → Click **"Next"**

4. **Select Components:**

   | Component | Selection |
   |-----------|-----------|
   | Privileged Session Manager (PSM) | ☑ Selected |
   | Secure Infrastructure Access (SIA) | ☑ Selected |

   Click **"Next"**

5. **Privilege Cloud Connection:**

   | Setting | Value |
   |---------|-------|
   | **Tenant URL** | [tenant-id].privilegecloud.cyberark.cloud |
   | **Installer User** | [your-admin-username] |
   | **Password** | [your-admin-password] |

   Click **"Test Connection"** → Verify success → Click **"Next"**

6. **PSM Configuration:**

   | Setting | Value |
   |---------|-------|
   | **PSM Server ID** | Connector01-PSM |
   | **PSM Address** | Connector01.indramind.cyblab |

   Click **"Next"**

7. **PSM Service Account:**

   | Setting | Value |
   |---------|-------|
   | **Use Domain Account** | Yes |
   | **Account** | INDRAMIND\svc-CyberArkPSM |
   | **Password** | [Service account password] |

   Click **"Validate"** → Verify success → Click **"Next"**

8. **Recording Settings:**

   | Setting | Value |
   |---------|-------|
   | **Recording Path** | D:\PSMRecordings |
   | **Temporary Files** | C:\Windows\Temp\PSM |

   Click **"Next"**

9. **SIA Configuration:**

   | Setting | Value |
   |---------|-------|
   | **Enable SIA** | Yes |
   | **Connector Name** | SIA-Connector01 |

   Click **"Next"**

10. **Installation Summary:**
    - Review all settings
    - Click **"Install"**

11. **Wait for Installation:**
    - Progress bar shows installation status
    - Typical duration: 10-20 minutes
    - Do not interrupt the process

12. **Installation Complete:**
    - Verify "Installation Successful" message
    - Note any warnings
    - Click **"Finish"**

#### 8.3.3 Post-Installation Verification (Connector01)

```powershell
# Verify PSM Installation on Connector01
Write-Host "PSM Installation Verification - Connector01" -ForegroundColor Cyan
Write-Host "=============================================" -ForegroundColor Cyan

# Check PSM Service
$PSMService = Get-Service -Name "CyberArk Privileged Session Manager" -ErrorAction SilentlyContinue
if ($PSMService) {
    Write-Host "[PSM Service]" -ForegroundColor White
    Write-Host "  Name:   $($PSMService.DisplayName)"
    Write-Host "  Status: $($PSMService.Status)" -ForegroundColor $(if($PSMService.Status -eq 'Running'){'Green'}else{'Red'})
} else {
    Write-Host "[ERROR] PSM Service not found!" -ForegroundColor Red
}

# Check SIA Service
$SIAService = Get-Service -Name "CyberArk Secure Infrastructure Access" -ErrorAction SilentlyContinue
if ($SIAService) {
    Write-Host "`n[SIA Service]" -ForegroundColor White
    Write-Host "  Name:   $($SIAService.DisplayName)"
    Write-Host "  Status: $($SIAService.Status)" -ForegroundColor $(if($SIAService.Status -eq 'Running'){'Green'}else{'Red'})
} else {
    Write-Host "[ERROR] SIA Service not found!" -ForegroundColor Red
}

# Check Installation Directory
$InstallPath = "C:\Program Files (x86)\CyberArk\PSM"
if (Test-Path $InstallPath) {
    Write-Host "`n[Installation Directory]" -ForegroundColor White
    Write-Host "  Path: $InstallPath [EXISTS]" -ForegroundColor Green
} else {
    Write-Host "`n[ERROR] Installation directory not found!" -ForegroundColor Red
}

# Check Event Log for Errors
Write-Host "`n[Recent Event Log Entries]" -ForegroundColor White
Get-EventLog -LogName Application -Source "CyberArk*" -Newest 5 -ErrorAction SilentlyContinue |
    Format-Table TimeGenerated, EntryType, Message -AutoSize -Wrap
```

### 8.4 Install PSM on Connector02 (Secondary)

#### 8.4.1 Pre-Installation Verification

Repeat the pre-installation check on Connector02 (same script as 8.3.1).

#### 8.4.2 Run Installation Wizard

Follow the same steps as Connector01 with these differences:

| Setting | Connector02 Value |
|---------|------------------|
| **PSM Server ID** | Connector02-PSM |
| **PSM Address** | Connector02.indramind.cyblab |
| **SIA Connector Name** | SIA-Connector02 |

All other settings remain the same (same service account, same tenant, etc.)

#### 8.4.3 Post-Installation Verification (Connector02)

Run the same verification script on Connector02.

### 8.5 Verify HA Configuration in Privilege Cloud

1. **Login to Privilege Cloud Portal:**
   ```
   https://[tenant-id].privilegecloud.cyberark.cloud
   ```

2. **Navigate to System Health:**
   ```
   Administration → System Health → Components
   ```

3. **Verify PSM Servers:**

   | Server | Expected Status |
   |--------|----------------|
   | Connector01-PSM | Connected (Green) |
   | Connector02-PSM | Connected (Green) |

4. **Verify SIA Connectors:**

   | Connector | Expected Status |
   |-----------|----------------|
   | SIA-Connector01 | Connected (Green) |
   | SIA-Connector02 | Connected (Green) |

### 8.6 Configure PSM Load Balancing

#### 8.6.1 Verify DNS Round Robin

```powershell
# Test DNS round robin is working
Write-Host "Testing PSM DNS Round Robin..." -ForegroundColor Cyan

1..10 | ForEach-Object {
    $Result = Resolve-DnsName -Name "psm.indramind.cyblab" -Type A -DnsOnly
    $IPs = $Result.IPAddress -join ", "
    Write-Host "Query $_`: $IPs"
}
```

#### 8.6.2 Configure PSM Server Address in Privilege Cloud

1. Navigate to:
   ```
   Administration → Configuration Options → Options → Connection Components
   ```

2. For each connection component (RDP, SSH, etc.):
   - Set **PSM Server Address** to: `psm.indramind.cyblab`
   - This enables automatic load balancing via DNS

### 8.7 Test PSM High Availability

#### 8.7.1 Test Normal Operation

1. **Create Test Target Account:**
   - Add a Windows local admin account to a test safe
   - Or use an existing managed account

2. **Connect via PSM:**
   - From Privilege Cloud portal, click "Connect" on the account
   - Verify RDP session opens successfully
   - Note which PSM server handled the connection (shown in session details)

3. **Test Multiple Connections:**
   - Open several connections
   - Verify connections are distributed between Connector01 and Connector02

#### 8.7.2 Test Failover

On Connector01, stop the PSM service:

```powershell
# FAILOVER TEST - Run on Connector01
Write-Host "Stopping PSM Service on Connector01 for failover test..." -ForegroundColor Yellow
Stop-Service -Name "CyberArk Privileged Session Manager" -Force
Get-Service -Name "CyberArk Privileged Session Manager"
```

**Test:**
1. Attempt new PSM connection from Privilege Cloud portal
2. Connection should succeed via Connector02
3. Verify in session details that Connector02-PSM handled the session

**Restore:**
```powershell
# RESTORE - Run on Connector01
Write-Host "Starting PSM Service on Connector01..." -ForegroundColor Cyan
Start-Service -Name "CyberArk Privileged Session Manager"
Get-Service -Name "CyberArk Privileged Session Manager"
```

**Verify Recovery:**
1. In Privilege Cloud portal, verify Connector01-PSM shows "Connected" again
2. New connections can be handled by either server

---

## 9. Phase 6: SIA Configuration

### 9.1 Overview

SIA (Secure Infrastructure Access) was installed alongside PSM. This phase covers verification and configuration for secure access to infrastructure.

### 9.2 Verify SIA Installation

```powershell
# Run on both Connector01 and Connector02
Write-Host "SIA Connector Verification" -ForegroundColor Cyan
Write-Host "==========================" -ForegroundColor Cyan

# Check SIA Service
$SIAService = Get-Service -Name "CyberArk Secure Infrastructure Access*" -ErrorAction SilentlyContinue

if ($SIAService) {
    foreach ($Svc in $SIAService) {
        Write-Host "`nService: $($Svc.DisplayName)"
        Write-Host "Status:  $($Svc.Status)" -ForegroundColor $(if($Svc.Status -eq 'Running'){'Green'}else{'Red'})
    }
} else {
    Write-Host "[WARNING] No SIA services found" -ForegroundColor Yellow
}

# Check SIA Installation Path
$SIAPath = "C:\Program Files\CyberArk\SIA"
if (Test-Path $SIAPath) {
    Write-Host "`nInstallation Path: $SIAPath [EXISTS]" -ForegroundColor Green
    Get-ChildItem $SIAPath -Directory | ForEach-Object { Write-Host "  - $($_.Name)" }
} else {
    Write-Host "`nInstallation Path: $SIAPath [NOT FOUND]" -ForegroundColor Red
}
```

### 9.3 Verify SIA in Privilege Cloud Portal

1. Navigate to:
   ```
   Administration → System Health → Secure Infrastructure Access
   ```

2. Verify connectors appear:

   | Connector | Status | Version |
   |-----------|--------|---------|
   | SIA-Connector01 | Active (Green) | [Version] |
   | SIA-Connector02 | Active (Green) | [Version] |

### 9.4 Configure SIA Target Systems

SIA enables secure access to infrastructure without traditional VPN. Configure target systems as needed:

1. Navigate to:
   ```
   Secure Infrastructure Access → Targets → Add Target
   ```

2. Configure target systems (servers, databases, etc.) that should be accessible via SIA.

---

## 10. Phase 7: Naming Conventions and Standards

### 10.1 Safe Naming Convention

#### 10.1.1 Format

```
{ENV}-{TIER}-{TYPE}-{OWNER}
```

#### 10.1.2 Environment Codes

| Code | Environment | Description |
|------|-------------|-------------|
| LAB | Laboratory | Test and development |
| DEV | Development | Development environment |
| STG | Staging | Pre-production |
| PRD | Production | Production environment |

#### 10.1.3 Tier Codes

| Code | Tier | Description |
|------|------|-------------|
| T0 | Tier 0 | Domain Controllers, Forest Admins |
| T1 | Tier 1 | Enterprise Servers |
| T2 | Tier 2 | Workstations |
| SVC | Service | Service Accounts |
| APP | Application | Application-specific |

#### 10.1.4 Type Codes

| Code | Type | Description |
|------|------|-------------|
| WIN | Windows | Windows accounts |
| LNX | Linux | Linux/Unix accounts |
| DB | Database | Database accounts |
| NET | Network | Network device accounts |
| CLD | Cloud | Cloud platform accounts |
| SSH | SSH Keys | SSH key pairs |

#### 10.1.5 Examples

| Safe Name | Description |
|-----------|-------------|
| LAB-T0-WIN-DomainAdmins | Lab Tier 0 Windows Domain Admin accounts |
| LAB-T1-WIN-Servers | Lab Tier 1 Windows Server local admins |
| LAB-T1-LNX-Servers | Lab Tier 1 Linux root accounts |
| LAB-T1-DB-Oracle | Lab Tier 1 Oracle DBA accounts |
| LAB-SVC-Automation | Lab Service accounts for automation |
| PRD-T0-WIN-DomainAdmins | Production Domain Admin accounts |

### 10.2 Create Initial Safes

In Privilege Cloud Portal:

```
Policies → Safes → Add Safe
```

#### 10.2.1 Create Tier 0 Safe

| Setting | Value |
|---------|-------|
| **Safe Name** | LAB-T0-WIN-DomainAdmins |
| **Description** | Tier 0 Domain Administrator accounts for lab environment |
| **Managing CPM** | (Cloud-managed) |

**Safe Members:**

| Member | Permissions |
|--------|-------------|
| PAM-Vault-Admins | Full (all permissions) |
| PAM-T0-Admins | Use accounts, Retrieve passwords |
| PAM-Auditors | List accounts, View audit |
| PAM-Approvers | Authorize requests |

**Workflow:**

| Setting | Value |
|---------|-------|
| **Require Dual Control** | Yes |
| **Required Approvers** | 2 |
| **Approval Timeout** | 60 minutes |

#### 10.2.2 Create Tier 1 Safes

**LAB-T1-WIN-Servers:**

| Setting | Value |
|---------|-------|
| **Safe Name** | LAB-T1-WIN-Servers |
| **Description** | Tier 1 Windows Server local administrator accounts |

| Member | Permissions |
|--------|-------------|
| PAM-Safe-Managers | Full |
| PAM-T1-WindowsAdmins | Use, Retrieve, List |
| PAM-T1-Operators | Use only |
| PAM-Auditors | List, View audit |

**LAB-T1-LNX-Servers:**

| Setting | Value |
|---------|-------|
| **Safe Name** | LAB-T1-LNX-Servers |
| **Description** | Tier 1 Linux/Unix root and privileged accounts |

| Member | Permissions |
|--------|-------------|
| PAM-Safe-Managers | Full |
| PAM-T1-LinuxAdmins | Use, Retrieve, List |
| PAM-T1-Operators | Use only |
| PAM-Auditors | List, View audit |

#### 10.2.3 Create Service Account Safe

| Setting | Value |
|---------|-------|
| **Safe Name** | LAB-SVC-Automation |
| **Description** | Service accounts for automation and scripts |

| Member | Permissions |
|--------|-------------|
| PAM-Vault-Admins | Full |
| PAM-Safe-Managers | Add, Update, Retrieve |
| PAM-Auditors | List, View audit |

### 10.3 Platform Naming Convention

#### 10.3.1 Use Standard Platforms

Where possible, use built-in platforms:

| Platform | Use Case |
|----------|----------|
| Windows Domain Account | AD domain accounts |
| Windows Server Local Account | Windows local admins |
| Unix via SSH | Linux/Unix accounts |
| Oracle Database | Oracle accounts |
| MS SQL Server | SQL Server accounts |

#### 10.3.2 Custom Platform Naming

When custom platforms are needed:

```
{BasePlatform}-{Customization}
```

**Examples:**

| Custom Platform | Base | Purpose |
|-----------------|------|---------|
| Windows Domain Account-LabServers | Windows Domain Account | Customized for lab servers |
| Unix via SSH-RHEL8 | Unix via SSH | RHEL 8 specific settings |
| Oracle Database-ERP | Oracle Database | ERP system specific |

### 10.4 Account Naming Convention

#### 10.4.1 Format

```
{PLATFORM}_{TYPE}_{TARGET}_{USERNAME}
```

#### 10.4.2 Examples

| Account Name | Description |
|--------------|-------------|
| WIN_DA_LAB_Administrator | Windows Domain Admin on LAB domain |
| WIN_LA_SRV01_Administrator | Windows Local Admin on SRV01 |
| LNX_ROOT_WEB01_root | Linux root on WEB01 |
| ORA_DBA_PROD_sys | Oracle DBA sys on PROD |

---

## 11. Phase 8: OKTA Integration

### 11.1 Overview

This phase configures OKTA as an external SAML Identity Provider for specific user groups.

> **Prerequisites:**
> - OKTA Admin access
> - OKTA organization URL
> - List of pilot users

### 11.2 Create SAML Application in OKTA

#### 11.2.1 Access OKTA Admin Console

```
URL: https://[your-org].okta.com/admin
```

#### 11.2.2 Create New Application

1. Navigate to:
   ```
   Applications → Applications → Create App Integration
   ```

2. Select:
   - **Sign-in method:** SAML 2.0
   - Click **"Next"**

#### 11.2.3 General Settings

| Setting | Value |
|---------|-------|
| **App Name** | CyberArk Privilege Cloud |
| **App Logo** | [Upload CyberArk logo - optional] |
| **App Visibility** | Show in Okta dashboard |

Click **"Next"**

#### 11.2.4 SAML Configuration

| Setting | Value |
|---------|-------|
| **Single sign-on URL** | https://[tenant-id].id.cyberark.cloud/saml/consume |
| **Audience URI (SP Entity ID)** | https://[tenant-id].id.cyberark.cloud |
| **Name ID format** | EmailAddress |
| **Application username** | Email |

**Attribute Statements:**

| Name | Value |
|------|-------|
| firstName | user.firstName |
| lastName | user.lastName |
| email | user.email |

**Group Attribute Statements:**

| Name | Filter | Value |
|------|--------|-------|
| groups | Starts with | CyberArk |

Click **"Next"**

#### 11.2.5 Feedback

- Select: "I'm an Okta customer adding an internal app"
- Click **"Finish"**

#### 11.2.6 Download SAML Metadata

1. On the application page, click **"Sign On"** tab

2. In **SAML 2.0** section:
   - Click **"View SAML setup instructions"** or
   - Click **"Identity Provider metadata"** to download XML

3. Note the following values:
   - **Identity Provider Single Sign-On URL**
   - **Identity Provider Issuer**
   - **X.509 Certificate** (download)

### 11.3 Configure OKTA as IdP in CyberArk Identity

#### 11.3.1 Navigate to External IdP

```
Settings → Authentication → External Identity Providers → Add
```

#### 11.3.2 Configure SAML IdP

**Basic Settings:**

| Setting | Value |
|---------|-------|
| **IdP Name** | OKTA |
| **IdP Type** | SAML 2.0 |
| **Status** | Enabled |

**SAML Configuration:**

| Setting | Value |
|---------|-------|
| **IdP Entity ID** | [From OKTA - Identity Provider Issuer] |
| **IdP SSO URL** | [From OKTA - Single Sign-On URL] |
| **IdP Certificate** | [Upload OKTA X.509 certificate] |
| **SP Entity ID** | https://[tenant-id].id.cyberark.cloud |

**User Mapping:**

| Setting | Value |
|---------|-------|
| **Match Users By** | Email |
| **Auto-Create Users** | No |

**Group Mapping (Optional):**

| OKTA Group | CyberArk Group |
|------------|----------------|
| CyberArk-Admins | PAM-Vault-Admins |
| CyberArk-Users | PAM-Users |
| CyberArk-Auditors | PAM-Auditors |

Click **"Save"**

### 11.4 Assign Users in OKTA

1. In OKTA Admin Console:
   ```
   Applications → CyberArk Privilege Cloud → Assignments
   ```

2. Click **"Assign"** → **"Assign to People"**

3. Select pilot users → Click **"Assign"** → **"Done"**

### 11.5 Test OKTA Integration

#### 11.5.1 Test IdP-Initiated Login

1. Login to OKTA dashboard as assigned user
2. Click "CyberArk Privilege Cloud" tile
3. Should redirect to CyberArk Identity with active session
4. Verify successful access

#### 11.5.2 Test SP-Initiated Login

1. Navigate to:
   ```
   https://[tenant-id].id.cyberark.cloud
   ```

2. Click "Login with OKTA" (or external IdP option)

3. Redirects to OKTA login

4. After OKTA authentication, returns to CyberArk Identity

5. Verify successful login and correct group mappings

---

## 12. Validation and Testing

### 12.1 Complete System Validation Checklist

Execute this comprehensive validation after completing all phases:

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                    CYBERARK LAB VALIDATION CHECKLIST                         ║
╚══════════════════════════════════════════════════════════════════════════════╝

ACTIVE DIRECTORY INTEGRATION
☐ AD Connector service account created (svc-CyberArkIdentity)
☐ TIER model OU structure exists
☐ All security groups created
☐ Service accounts created
☐ AD directory connected in CyberArk Identity
☐ User sync working (check sync status)
☐ Group sync working (verify groups in Identity)
☐ AD authentication tested successfully

USER MANAGEMENT
☐ Team member accounts created in AD
☐ Users placed in correct OU (Active)
☐ Users added to appropriate groups
☐ Departed users disabled and moved
☐ Users appear in CyberArk Identity after sync

MULTI-FACTOR AUTHENTICATION
☐ Email OTP configured and tested
☐ SMS OTP configured and tested
☐ Mobile Authenticator configured
☐ LAB-2FA-Standard policy created
☐ LAB-2FA-HighSecurity policy created
☐ Policy priority order correct
☐ Standard users get all MFA options
☐ High security users get Mobile App only
☐ At least one user enrolled in Mobile App

PSM HIGH AVAILABILITY
☐ Connector01 - PSM installed and running
☐ Connector01 - SIA installed and running
☐ Connector02 - PSM installed and running
☐ Connector02 - SIA installed and running
☐ Both PSM servers show Connected in portal
☐ Both SIA connectors show Active in portal
☐ DNS round robin configured (psm.indramind.cyblab)
☐ PSM connections work via either server
☐ Failover tested - connections work with one server down
☐ Recovery tested - server rejoins after restart

NAMING CONVENTIONS
☐ Safe naming document created/communicated
☐ Initial safes created with correct naming
☐ Safe permissions configured correctly
☐ Account naming convention documented

OKTA INTEGRATION (if applicable)
☐ SAML application created in OKTA
☐ OKTA configured as IdP in CyberArk Identity
☐ Pilot users assigned in OKTA
☐ IdP-initiated login tested
☐ SP-initiated login tested
☐ Group mapping verified
```

### 12.2 Functional Test Scenarios

#### 12.2.1 End-to-End User Login Test

1. **AD User Login with MFA:**
   - Login as AD user (test.pamuser@indramind.cyblab)
   - Verify AD password accepted
   - Complete MFA challenge
   - Verify access to Privilege Cloud

2. **Access Privileged Account:**
   - Navigate to account in safe
   - Request access (if dual control enabled)
   - Connect via PSM
   - Verify session recording

#### 12.2.2 High Availability Test

```powershell
# HA Test Script - Run from admin workstation
Write-Host "PSM High Availability Test" -ForegroundColor Cyan
Write-Host "==========================" -ForegroundColor Cyan

# Test 1: Both servers up
Write-Host "`n[Test 1] Both servers operational" -ForegroundColor White
$Results = @()
1..4 | ForEach-Object {
    $IP = (Resolve-DnsName -Name "psm.indramind.cyblab" -Type A -DnsOnly).IPAddress | Select-Object -First 1
    $Results += $IP
}
Write-Host "DNS responses: $($Results -join ', ')"
Write-Host "Load balanced: $(if(($Results | Select-Object -Unique).Count -gt 1){'YES'}else{'NO - Check DNS'})"

# Instructions for manual tests
Write-Host "`n[Test 2] Failover test (manual):" -ForegroundColor White
Write-Host "  1. Stop PSM service on Connector01"
Write-Host "  2. Attempt PSM connection from portal"
Write-Host "  3. Verify connection succeeds via Connector02"

Write-Host "`n[Test 3] Recovery test (manual):" -ForegroundColor White
Write-Host "  1. Start PSM service on Connector01"
Write-Host "  2. Verify Connector01-PSM shows Connected in portal"
Write-Host "  3. Attempt new connection - may route to either server"
```

### 12.3 Security Validation

```powershell
# Security Configuration Check
Write-Host "Security Configuration Validation" -ForegroundColor Cyan
Write-Host "==================================" -ForegroundColor Cyan

# Check service account permissions
Write-Host "`n[Service Accounts]" -ForegroundColor White
Get-ADUser -Filter "Name -like 'svc-CyberArk*'" -SearchBase "OU=ServiceAccounts,OU=PAM,DC=indramind,DC=cyblab" -Properties PasswordNeverExpires, CannotChangePassword, Enabled |
    Select-Object Name, Enabled, PasswordNeverExpires, CannotChangePassword |
    Format-Table -AutoSize

# Check PSM service running as correct account
Write-Host "`n[PSM Service Identity]" -ForegroundColor White
$PSMService = Get-WmiObject Win32_Service -Filter "Name='CyberArk Privileged Session Manager'"
Write-Host "Service Account: $($PSMService.StartName)"

# Check Windows Defender exclusions
Write-Host "`n[Defender Exclusions]" -ForegroundColor White
$Exclusions = Get-MpPreference
Write-Host "Path Exclusions:"
$Exclusions.ExclusionPath | ForEach-Object { Write-Host "  - $_" }
```

---

## 13. Troubleshooting

### 13.1 AD Connector Issues

#### Problem: Directory sync fails

**Symptoms:**
- Sync status shows "Failed"
- Users not appearing in CyberArk Identity

**Troubleshooting Steps:**

1. **Verify network connectivity:**
   ```powershell
   # From a cloud-connected system, test LDAP
   Test-NetConnection -ComputerName DC01.indramind.cyblab -Port 636
   ```

2. **Verify service account:**
   ```powershell
   # Test LDAP bind
   $Cred = Get-Credential -UserName "svc-CyberArkIdentity@indramind.cyblab"
   $LDAP = New-Object System.DirectoryServices.DirectoryEntry("LDAP://DC01.indramind.cyblab:636/DC=indramind,DC=cyblab", $Cred.UserName, $Cred.GetNetworkCredential().Password)
   $LDAP.Name
   ```

3. **Check service account not locked:**
   ```powershell
   Get-ADUser svc-CyberArkIdentity -Properties LockedOut, Enabled, PasswordExpired
   ```

4. **Verify Base DN and filters:**
   - Check that OU=PAM exists
   - Verify LDAP filter syntax

### 13.2 PSM Connection Issues

#### Problem: PSM connections fail

**Symptoms:**
- "Connection failed" error in portal
- RDP window doesn't open

**Troubleshooting Steps:**

1. **Check PSM service status:**
   ```powershell
   Get-Service -Name "CyberArk Privileged Session Manager"
   ```

2. **Check PSM connectivity to cloud:**
   ```powershell
   Test-NetConnection -ComputerName connector.privilegecloud.cyberark.cloud -Port 443
   ```

3. **Check PSM logs:**
   ```powershell
   Get-EventLog -LogName Application -Source "CyberArk*" -Newest 20 |
       Where-Object { $_.EntryType -eq "Error" } |
       Format-Table TimeGenerated, Message -Wrap
   ```

4. **Verify PSM server in portal:**
   - Check Administration → System Health → Components
   - Status should be "Connected"

5. **Check target system connectivity:**
   ```powershell
   # From PSM server, test connection to target
   Test-NetConnection -ComputerName [target-server] -Port 3389
   ```

### 13.3 MFA Issues

#### Problem: MFA codes not received

**Symptoms:**
- User doesn't receive Email OTP
- SMS OTP not delivered

**Troubleshooting Steps:**

1. **Verify user email/phone:**
   - Check user profile in CyberArk Identity
   - Confirm email address is correct
   - Confirm phone number includes country code

2. **Check spam/junk folders:**
   - Email OTP may be filtered

3. **Verify MFA settings:**
   - Settings → Authentication → Multi Factor Authentication
   - Confirm Email/SMS OTP enabled

4. **Check policy assignment:**
   - Verify user is in group covered by MFA policy
   - Check policy priority (lower number = higher priority)

### 13.4 Common Error Messages

| Error | Cause | Solution |
|-------|-------|----------|
| "Directory connection failed" | Network or credential issue | Verify LDAP port open, check service account |
| "User not found" | User not synced or wrong directory | Force sync, check user OU is in sync scope |
| "Invalid credentials" | Wrong password or account locked | Reset password, unlock account |
| "MFA required" but no options | Policy misconfiguration | Check MFA policy covers user's group |
| "PSM connection refused" | PSM service down or blocked | Check service status, firewall rules |
| "Session recording failed" | Disk full or permissions | Check D:\PSMRecordings space and permissions |

---

## 14. Appendices

### Appendix A: Complete Server IP Reference

| Server | IP Address | Role |
|--------|------------|------|
| DC01 | 10.10.3.151 | Domain Controller, DNS |
| Connector01 | 10.10.3.8 | PSM, SIA |
| Connector02 | 10.10.3.70 | PSM, SIA |
| psm (HA) | 10.10.3.8, 10.10.3.70 | DNS Round Robin |

### Appendix B: Service Account Reference

| Account | UPN | Purpose |
|---------|-----|---------|
| svc-CyberArkIdentity | svc-CyberArkIdentity@indramind.cyblab | AD connector |
| svc-CyberArkPSM | svc-CyberArkPSM@indramind.cyblab | PSM service |
| svc-CyberArkScan | svc-CyberArkScan@indramind.cyblab | Account discovery |
| svc-CyberArkReconcile | svc-CyberArkReconcile@indramind.cyblab | Reconciliation |

### Appendix C: URL Reference

| Service | URL |
|---------|-----|
| Privilege Cloud Portal | https://[tenant].privilegecloud.cyberark.cloud |
| Identity Admin Portal | https://[tenant].id.cyberark.cloud/admin |
| Identity User Portal | https://[tenant].id.cyberark.cloud |
| OKTA Admin (if used) | https://[org].okta.com/admin |

### Appendix D: Group to Safe Permission Matrix

| Group | T0 Safes | T1 Safes | T2 Safes | SVC Safes |
|-------|----------|----------|----------|-----------|
| PAM-Vault-Admins | Full | Full | Full | Full |
| PAM-Safe-Managers | - | Full | Full | Add/Update |
| PAM-T0-Admins | Use/Retrieve | - | - | - |
| PAM-T1-WindowsAdmins | - | Use/Retrieve (WIN) | - | - |
| PAM-T1-LinuxAdmins | - | Use/Retrieve (LNX) | - | - |
| PAM-T2-Admins | - | - | Use/Retrieve | - |
| PAM-Auditors | List/Audit | List/Audit | List/Audit | List/Audit |
| PAM-Approvers | Authorize | Authorize | - | - |

### Appendix E: Change Log Template

```
═══════════════════════════════════════════════════════════════════════════════
CHANGE LOG ENTRY
═══════════════════════════════════════════════════════════════════════════════

Date:           [YYYY-MM-DD]
Time:           [HH:MM]
Performed By:   [Name]
Change ID:      [Ticket/Reference Number]

CHANGE DESCRIPTION:
[Detailed description of the change]

SYSTEMS AFFECTED:
- [System 1]
- [System 2]

PRE-CHANGE STATE:
[Description of state before change]

POST-CHANGE STATE:
[Description of state after change]

VERIFICATION:
☐ Change implemented successfully
☐ Functionality verified
☐ No negative impact observed
☐ Documentation updated

ROLLBACK PERFORMED: Yes / No
If Yes, reason: [Reason]

NOTES:
[Any additional notes or observations]
═══════════════════════════════════════════════════════════════════════════════
```

---

## Document End

**Version:** 1.0
**Last Updated:** January 2026
**Classification:** Technical Implementation Guide

For questions or issues, contact the CyberArk Lab administration team.
