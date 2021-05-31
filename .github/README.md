# WinBTRFS read-only hack

This is a fork of Mark Harmstone's [WinBtrfs](https://github.com/maharmstone/btrfs); it ensures that existing BTRFS filesystems are always mounted **read-only**. In fact, right now it should be **impossible to mount read-write** (which is overkill, but I had to be sure &mdash; it would be nice to allow opt-in read-write access via separate registry keys).

Why? There are GitHub reports about filesystem corruption (from the Linux side at least; it may well be that sticking to WinBTRFS only is perfectly safe). BTRFS is a complex piece of software: to really trust a filesystem, it needs to have seen a lot of real-world usage. While I need occasional (but essential) access to Linux data from the occasional (but essential) Windows boot, I don't really need to write to BTRFS. Surely a reboot is preferrable to risking data loss. YMMV &mdash; install the original BTRFS if permanent read-only mode isn't for you.

The downside is that you have to self-sign (actually test-sign) the new driver, so it won't load in a normal Windows boot. You can either enable test-signing mode (the only negative effect of which, AFAICT, is an extra watermark on the desktop) or disable signature verification (only lasts for one boot, and the system might delete the driver subsequently, requiring a re-install).

See the links section. More or less detailed instructions follow. The original project homepage is in `README.orig.md`.

## Build the driver

- install [Visual Studio Community and the WDK](https://docs.microsoft.com/en-us/windows-hardware/drivers/download-the-wdk)
- build
- unpack WinBTRFS original driver
- replace original driver files with newly built from `out/build/x64-Release/`

## Install the driver

Prepare system for test signing:
```
bcdedit -set TESTSIGNING ON
Auditpol /set /Category:System /failure:enable; Auditpol /get /Category:System  # needed?
# reboot
```

For convenience, `exewrap` all utilities somewhere & add to `PATH` (WDK doesn't seem to put anything in the `PATH`, at least not in my install). Then use `tool.bat ...` instead of `/path/to/tool.exe`
```
$d = '~/drvsign'; $env:Path += ";$d"
md $d; cd $d
$p = 'C:\Program Files (x86)\Windows Kits\10\bin\10.0.19041.0'
function exewrap { param( $cmd ); "'{0}' /*" -f $cmd >"$(gci $cmd | % Basename).bat" }  # PS7
exewrap "$p/x64/certmgr.exe"
exewrap "$p/x64/makecert.exe"
exewrap "$p/x64/signtool.exe"
exewrap "$p/x86/Inf2Cat.exe"
```

### Create & install certificate

```
makecert -r -pe -ss PrivateCertStore -n CN=Contoso.com(Test) -eku 1.3.6.1.5.5.7.3.3 ContosoTest.cer
certmgr /add ContosoTest.cer /s /r localMachine root
Inf2Cat /v /driver:. /os:10_X64
signtool sign /v /fd sha256 /s PrivateCertStore /n Contoso.com(Test) /t http://timestamp.digicert.com .\btrfs.cat
```

### Sign & install driver

```
mountvol /N  # disable automount!
#mountvol /E  # to re-enable
RUNDLL32.EXE SETUPAPI.DLL,InstallHinfSection DefaultInstall 132 btrfs.inf
#RUNDLL32.EXE SETUPAPI.DLL,InstallHinfSection DefaultUninstall 132 btrfs.inf
# reboot
```

## Check & use driver

```
Get-WindowsDriver -Online -All | % Driver | Out-File drivers.txt
driverquery.exe /v

Get-Volume | ? FileSystem -eq 'Btrfs' | format-list
mountvol.exe i: '\\?\Volume{3119cb5e-a319-1058-316c-2ba01da10b75}\'
#mountvol i: /p  # unmount; must reboot to re-mount here

Get-Disk | Get-Partition | Format-List PartitionNumber, Size
mkbtrfs j:  # use RAW partition or \\Device; doesn't check for existing filesystem!!
```

## Links

- [GitHub - maharmstone/btrfs: WinBtrfs - an open-source btrfs driver for Windows](https://github.com/maharmstone/btrfs)
- [Configuring the Test Computer to Support Test-Signing - Windows drivers | Microsoft Docs](https://docs.microsoft.com/en-us/windows-hardware/drivers/install/configuring-the-test-computer-to-support-test-signing)
- [Introduction to Test-Signing - Windows drivers | Microsoft Docs](https://docs.microsoft.com/en-us/windows-hardware/drivers/install/introduction-to-test-signing)
- [How to Test-Sign a Driver Package - Windows drivers | Microsoft Docs](https://docs.microsoft.com/en-us/windows-hardware/drivers/install/how-to-test-sign-a-driver-package)
- [Installing Test Certificates - Windows drivers | Microsoft Docs](https://docs.microsoft.com/en-us/windows-hardware/drivers/install/installing-test-certificates)
- [Test-Signing a Driver through an Embedded Signature - Windows drivers | Microsoft Docs](https://docs.microsoft.com/en-us/windows-hardware/drivers/install/test-signing-a-driver-through-an-embedded-signature)
- [Windows 10: How to install drivers which are not digitally signed - TechNet Articles - United States (English) - TechNet Wiki](https://social.technet.microsoft.com/wiki/contents/articles/51875.windows-10-how-to-install-drivers-which-are-not-digitally-signed.aspx)
- [Creating a Catalog File for Test-Signing a Driver Package - Windows drivers | Microsoft Docs](https://docs.microsoft.com/en-us/windows-hardware/drivers/install/creating-a-catalog-file-for-test-signing-a-driver-package)
- [Inf2Cat - Windows drivers | Microsoft Docs](https://docs.microsoft.com/en-us/windows-hardware/drivers/devtest/inf2cat)
- [Catalog Files and Digital Signatures - Windows drivers | Microsoft Docs](https://docs.microsoft.com/en-us/windows-hardware/drivers/install/catalog-files)
- [Tools for Signing Drivers - Windows drivers | Microsoft Docs](https://docs.microsoft.com/en-us/windows-hardware/drivers/devtest/tools-for-signing-drivers)
- [Installing a File System Driver - Windows drivers | Microsoft Docs](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/installing-a-file-system-driver)
- [Commercial Test Certificate - Windows drivers | Microsoft Docs](https://docs.microsoft.com/en-us/windows-hardware/drivers/install/commercial-test-certificate)
- [Developing and Installing your first Kernel driver in Windows 10(under 10 min)](https://nixhacker.com/creating-and-loading-your-first-kernel-driver-in-windows-10/)

