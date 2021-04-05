# WinBTRFS read-only hack

This is a fork of Mark Harmstone's [WinBtrfs](https://github.com/maharmstone/btrfs) (the original GitHub front-page is in `README.orig.md`). This version ensures that existing BTRFS filesystems are always mounted **read-only**. In fact, right now it should be **impossible to mount read-write** (which is overkill, but safest; it would be nice to make optional rw-access configurable via separate registry keys).

Why? There are GitHub reports about filesystem corruption (from the Linux side at least; sticking to WinBTRFS only may well be perfectly safe). BTRFS is a complex piece of software: to really trust a filesystem, it needs to have seen a lot of real-world usage. While I need occasional (but essential) access to Linux data from the occasional (but essential) Windows boot, I don't really need to write to BTRFS. Surely a reboot is preferrable to risking data loss. YMMV &mdash; install the original WinBTRFS if permanent read-only mode isn't for you.

The downside is that (assuming you don't have access to "real" certificates) the new driver will be self-signed, so it won't load in a normal Windows boot. You can either enable test-signing mode (the only negative effect of which, AFAICT, is an extra watermark on the desktop) or disable signature verification (only lasts for one boot, and the system might delete the driver subsequently, requiring a re-install). See the links section.

More or less detailed instructions follow. The commands work in Powershell 7.

## Build the driver

- install [Visual Studio Community and the WDK](https://docs.microsoft.com/en-us/windows-hardware/drivers/download-the-wdk)
- build
- unpack the WinBTRFS original driver
- replace the files in `x64\` with newly built versions (look in `out/build/x64-Release/` under your Visual Studio build directory)
- edit the `.inf` file to your liking

## Install the driver

Enable test signing:
```
bcdedit -set TESTSIGNING ON
Auditpol /set /Category:System /failure:enable  # needed?
Auditpol /get /Category:System
# reboot
```

For convenience, `exewrap` all utilities in the unzipped driver directory. Copying just the `EXE`'s will yield .NET errors. Now you can use `tool[.bat] ...` instead of `/path/to/tool.exe`
```
$d = '~/drvsign'; $env:Path += ";$d"
md $d; cd $d
$p = 'C:\Program Files (x86)\Windows Kits\10\bin\10.0.19041.0'
function exewrap { param( $cmd ); "`"{0}`" %*" -f $cmd >"$(gci $cmd | % Basename).bat" }
exewrap $p/x64/certmgr.exe
exewrap $p/x64/makecert.exe
exewrap $p/x64/signtool.exe
exewrap $p/x86/Inf2Cat.exe
```

### Create certificate & sign driver

```
$u = [Environment]::UserName
$cname = "Contoso.com($($u))"
./makecert -r -pe -ss PrivateCertStore -n CN=$cname -eku 1.3.6.1.5.5.7.3.3 ContosoTest-$u.cer
./Inf2Cat /v /driver:. /os:10_X64
./signtool sign /v /fd sha256 /s PrivateCertStore /n $cname /t http://timestamp.digicert.com btrfs.cat
```

### Install the driver

Bring the driver directory over to the target machine, if different.

```
./certmgr /add ContosoTest*.cer /s /r localMachine root
mountvol /N  # disable automount! /E to re-enable
RUNDLL32.EXE SETUPAPI.DLL,InstallHinfSection DefaultInstall 132 btrfs.inf
#RUNDLL32.EXE SETUPAPI.DLL,InstallHinfSection DefaultUninstall 132 btrfs.inf
# reboot
```

You can also right-click on the `.inf` file and select "Install" (after installing the test certificate).

## Check & use driver

```
Get-WindowsDriver -Online -All | % Driver
driverquery /v

Get-Disk | Get-Partition | fl PartitionNumber, Size
Get-Volume | ? FileSystem -eq 'Btrfs' | fl

# RAW partition or \\Device\...\Partition\...
mkbtrfs ...  # won't check for existing data!!
mountvol i: '\\?\Volume{3219cb5e-...}\'
#mountvol i: /p  # unmount; must reboot to re-mount here
```

For `mountvol` and `mkbtrfs`, you can use drive letters or device paths (`\\Device\...\Partition\<number>` or `\\?\Volume{<UUID>}\`). It's safest to experiment with an empty partition first.


## Links

- [GitHub - maharmstone/btrfs: WinBtrfs - an open-source btrfs driver for Windows](https://github.com/maharmstone/btrfs)
- [Configuring the Test Computer to Support Test-Signing - Windows drivers](https://docs.microsoft.com/en-us/windows-hardware/drivers/install/configuring-the-test-computer-to-support-test-signing)
- [Introduction to Test-Signing - Windows drivers](https://docs.microsoft.com/en-us/windows-hardware/drivers/install/introduction-to-test-signing)
- [How to Test-Sign a Driver Package - Windows drivers](https://docs.microsoft.com/en-us/windows-hardware/drivers/install/how-to-test-sign-a-driver-package)
- [Installing Test Certificates - Windows drivers](https://docs.microsoft.com/en-us/windows-hardware/drivers/install/installing-test-certificates)
- [Test-Signing a Driver through an Embedded Signature - Windows drivers](https://docs.microsoft.com/en-us/windows-hardware/drivers/install/test-signing-a-driver-through-an-embedded-signature)
- [Windows 10: How to install drivers which are not digitally signed - TechNet Articles - United States (English) - TechNet Wiki](https://social.technet.microsoft.com/wiki/contents/articles/51875.windows-10-how-to-install-drivers-which-are-not-digitally-signed.aspx)
- [Creating a Catalog File for Test-Signing a Driver Package - Windows drivers](https://docs.microsoft.com/en-us/windows-hardware/drivers/install/creating-a-catalog-file-for-test-signing-a-driver-package)
- [Inf2Cat - Windows drivers](https://docs.microsoft.com/en-us/windows-hardware/drivers/devtest/inf2cat)
- [Catalog Files and Digital Signatures - Windows drivers](https://docs.microsoft.com/en-us/windows-hardware/drivers/install/catalog-files)
- [Tools for Signing Drivers - Windows drivers](https://docs.microsoft.com/en-us/windows-hardware/drivers/devtest/tools-for-signing-drivers)
- [Installing a File System Driver - Windows drivers](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/installing-a-file-system-driver)
- [Installing a Test-Signed Driver Package on the Test Computer - Windows drivers](https://docs.microsoft.com/en-us/windows-hardware/drivers/install/installing-a-test-signed-driver-package-on-the-test-computer)
- [Commercial Test Certificate - Windows drivers](https://docs.microsoft.com/en-us/windows-hardware/drivers/install/commercial-test-certificate)
- [Developing and Installing your first Kernel driver in Windows 10(under 10 min)](https://nixhacker.com/creating-and-loading-your-first-kernel-driver-in-windows-10/)

