# Volumiser

## Introduction

Volumiser is a command line tool and interactive console GUI for listing, browsing and extracting files from common virtual machine hard disk image formats.

The tool was written to combat a regular problem where massive 100G+ disk images are often hard to work with or exfiltrate when performing red team operations.  Whilst the tool was created for offensive operations, the tool also has benefits outside of it's original intended purpose.

![Volumiser Interactive GUI](Volumiser.gif)

Volumiser would not be possible without the brilliant [DiscUtils](https://github.com/DiscUtils/DiscUtils) project that does most of the heavy lifting parsing volumes and file systems within the virtual disks.  Whilst the EBS volume support is a feature added as part of volumiser, this also leverages this excellent library to add this particular disk image format.

Volumiser supports the following disk image formats:

* Amazon EBS Snapshots
* Direct Raw Disk (a la NinjaCopy)
* VHDX
* VMDK
* VHD


along with the following file systems:

* NTFS
* FAT
* ext3
* ext4

## Usage

### Interactive Mode

Interactive mode is started by supplying the `--image` argument followed by a local file or EBS snapshot id

**EBS Snapshot Example**

EBS snapshot can be loaded via the custom EBS protocol which levereges the EBS direct API for seeking and reading sectors from an EBS volume.  By default the AWS CLI credentials file is used for authentication, various AWS options can also be specified using the `--awsprofile`, `--awskey`, `--awssecret` and `--awsregion` arguments 

```powershell
Volumiser.exe --image "ebs://snap-12345675c8173707d"
```

**Direct Raw Disk Example**

Providing you have administrator rights, local raw disk volumes can be accessed via the `\\.\PhysicalDriveX` image specifier or `\\.\C:`.  When using the `PhysicalDrive` method, this will parse all volumes contained on the entire disk.  In situations where a volume is encrypted with BitLocker, you can switch to the drive letter method, this will access the single volume exposed by the drive letter which will automatically be decrypted by Windows when the volume sectors are read.

*Reading all the volumes present on the first physical drive attached to the host*
```powershell
Volumiser.exe --image "\\.\PhysicalDrive0"
```


*Reading the single volume attached to the drive letter C: (use this method for BitLocker'd volumes)*
```powershell
Volumiser.exe --image "\\.\C:"
```

**Local Image File Example**

Disk images accessible via the file system can also be specified, including files from network shares

```powershell
Volumiser.exe --image "c:\Virtual Machines\Domain Controller.vhdx"
```

### C2 Mode

In the event that the interactive console cannot be used, Volumiser supports listing volumes and file systems directly via the `--command` and `--path` arguments.

**Listing Volumes**

Volumes contained within a disk image can be listed using the volumes command

```
Volumiser.exe --image "c:\Virtual Machines\Domain Controller.vhdx" --command volumes
[+] Opened disk image, Size: 127GB
        Volume ID: VLG{2d02912f-a98f-4074-aaee-c3444d01b43a}, Size: 100 MB, Type: Microsoft FAT
        Volume ID: VLG{22956ef6-5b59-41f7-8751-8331c6183062}, Size: 16 MB, Type: Unknown
        Volume ID: VLG{166c0197-909e-419d-a431-2d9b9df4d1fe}, Size: 129376 MB, Type: Microsoft NTFS
        Volume ID: VLG{bdd5d39c-a214-4ac2-a6b9-2477fe02ffc1}, Size: 553 MB, Type: Microsoft NTFS
```

**Listing File System**

Once the volumes have been discovered, the file system for each volume can be listed

```
Volumiser.exe --image "c:\Virtual Machines\Domain Controller.vhdx" --command ls --path "VLG{166c0197-909e-419d-a431-2d9b9df4d1fe}:\Windows"
[+] Opened disk image, Size: 127GB
[+] Opened volume with ID VLG{166c0197-909e-419d-a431-2d9b9df4d1fe}
17/10/2022 18:51:29  DIR             appcompat
17/10/2022 19:52:06  DIR             apppatch
17/10/2022 18:59:17  DIR             AppReadiness
17/10/2022 19:53:57  DIR             assembly
14/05/2022 09:26:58  DIR             bcastdvr
14/05/2022 09:42:31  DIR             Boot
14/05/2022 09:26:54  DIR             Branding
14/05/2022 11:35:21  DIR             BrowserCor
...
```

**Downloading Files**

Files can be downloaded to your local machine using the download command.  By default files are written to the current working directory using the source filename.  The `--output` argument can be used to specify a destination file path or directory.

```
Volumiser.exe --image "c:\Virtual Machines\Domain Controller.vhdx" --command download --path "VLG{166c0197-909e-419d-a431-2d9b9df4d1fe}:\Windows\system32\config\SYSTEM"
[+] Opened disk image, Size: 127GB
[+] Opened volume with ID VLG{166c0197-909e-419d-a431-2d9b9df4d1fe}
[+] Opened file with path \Windows\System32\config\SYSTEM for with size: 12058624
[+] Saved to SYSTEM
```

Specifying a custom output path or directory

```
Volumiser.exe --image "c:\Virtual Machines\Domain Controller.vhdx" --command download --path "VLG{166c0197-909e-419d-a431-2d9b9df4d1fe}:\Windows\system32\config\SYSTEM" --output "C:\loot\DC-SYSTEM"
[+] Opened disk image, Size: 127GB
[+] Opened volume with ID VLG{166c0197-909e-419d-a431-2d9b9df4d1fe}
[+] Opened file with path \Windows\System32\config\SYSTEM for with size: 12058624
[+] Saved to C:\loot\DC-SYSTEM
```

**Extracting Credential Files**

The extract command provides a quick way to pull the files needed for offline credential extraction.  Rather than downloading each file individually, the extract command knows which files are needed based on the target type and handles everything in a single step.  Extracted files are written with random names and a `.log` extension to avoid leaking the original filenames on disk or in the process command line.

The `--mode` argument specifies the target type:
* `ws` - Workstation/Server: extracts SAM, SYSTEM and SECURITY from `\Windows\system32\config\`
* `dc` - Domain Controller: extracts SYSTEM from `\Windows\system32\config\` and ntds.dit from `\Windows\NTDS\`

The `--volume` argument specifies which volume to extract from and `--output` specifies the destination directory.

*Extracting from a workstation*
```
Volumiser.exe --image "c:\Virtual Machines\Workstation.vhdx" --command extract --volume "VLG{166c0197-909e-419d-a431-2d9b9df4d1fe}" --mode ws --output C:\loot\
[+] Opened disk image, Size: 127GB
[+] Opened volume with ID VLG{166c0197-909e-419d-a431-2d9b9df4d1fe}
[+] \Windows\system32\config\SAM -> b1c4e892.log (56.00KB)
[+] \Windows\system32\config\SYSTEM -> 5d3a71f0.log (11.50MB)
[+] \Windows\system32\config\SECURITY -> e8f23a01.log (28.00KB)
[+] Extract complete: 3 files
```

*Extracting from a domain controller*
```
Volumiser.exe --image "c:\Virtual Machines\Domain Controller.vhdx" --command extract --volume "VLG{166c0197-909e-419d-a431-2d9b9df4d1fe}" --mode dc --output C:\loot\
[+] Opened disk image, Size: 127GB
[+] Opened volume with ID VLG{166c0197-909e-419d-a431-2d9b9df4d1fe}
[+] \Windows\system32\config\SYSTEM -> a3f1b20c.log (11.50MB)
[+] \Windows\NTDS\ntds.dit -> 7e09d4f1.log (48.20MB)
[+] Extract complete: 2 files
```

The `--zip` flag bundles all extracted files into a single zip archive.  Files are streamed directly into the zip without being written to disk individually.  The zip itself is also given a random name with a `.log` extension.

```
Volumiser.exe --image "c:\Virtual Machines\Domain Controller.vhdx" --command extract --volume "VLG{166c0197-909e-419d-a431-2d9b9df4d1fe}" --mode dc --output C:\loot\ --zip
[+] Opened disk image, Size: 127GB
[+] Opened volume with ID VLG{166c0197-909e-419d-a431-2d9b9df4d1fe}
[+] \Windows\system32\config\SYSTEM -> a3f1b20c.log (11.50MB)
[+] \Windows\NTDS\ntds.dit -> 7e09d4f1.log (48.20MB)
[+] Zipped to 9c2e4d87.log (41.30MB)
[+] Extract complete: 2 files
```
