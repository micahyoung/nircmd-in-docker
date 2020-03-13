# Experiments with nircmd in Windows Docker containers

## Docs
Useful features:
* Manipulating Windows and Dialog boxes: http://nircmd.nirsoft.net/win.html

## Experiments
### Import Root CA cert to CurrentUser/Root
```Dockerfile
FROM mcr.microsoft.com/windows/servercore:1809

USER ContainerUser

RUN curl.exe -L -o nircmd-x64.zip https://www.nirsoft.net/utils/nircmd-x64.zip && mkdir nircmd && tar -xf nircmd-x64.zip -C nircmd && del nircmd-x64.zip

# Generate self-signed Root CA and export to pfx
RUN powershell -Command \
    $rootCA = New-SelfSignedCertificate -KeyExportPolicy 'Exportable' -KeyUsage 'CertSign','CRLSign' -Subject CN=testme -CertStoreLocation Cert:\CurrentUser\My; \
    \
    Export-PfxCertificate -Password (ConvertTo-SecureString -String '1234' -Force -AsPlainText) -Cert $rootCA -FilePath 'C:\rootCA.pfx'"

# Run certutil -importpfx in the background; then run nircmd to wait then click "yes" on the import prompt dialog
RUN start /b certutil -p 1234 -user -importpfx Root c:\rootCA.pfx & c:\nircmd\nircmd.exe cmdwait 5000 win dlgclick stitle "Security Warning" yes

# List certs and see "testme"
RUN certutil -user -store Root
```

Build
```powershell
docker build --name nircmd-cert-test .
```

Output
```
...
Step 5/6 : RUN start /b certutil -p 1234 -user -importpfx Root c:\rootCA.pfx & c:\nircmd\nircmd.exe cmdwait 5000 win dlgclick stitle "Security Warning" yes
 ---> Running in 5edebe9718ba
Certificate "testme" added to store.

CertUtil: -importPFX command completed successfully.
...
```

### Close a window by name

`Dockerfile`
```Dockerfile
FROM mcr.microsoft.com/windows/servercore:1809

RUN curl.exe -L -o nircmd-x64.zip https://www.nirsoft.net/utils/nircmd-x64.zip && mkdir nircmd && tar -xf nircmd-x64.zip -C nircmd && del nircmd-x64.zip
```

Build
```powershell
docker build --name nircmd .
```

```powershell
docker run --rm -it nircmd
C:\> notepad

C:\> tasklist /fi "IMAGENAME eq notepad.exe"
Image Name                     PID Session Name        Session#    Mem Usage 
========================= ======== ================ =========== ============
notepad.exe                   1388 Services                   1      8,556 K

C:\> nircmd\nircmdc.exe win close stitle "Untitled"

C:\> tasklist
INFO: No tasks are running which match the specified criteria.
```

### Open then close a dialog

`Dockerfile`
```Dockerfile
FROM mcr.microsoft.com/windows/servercore:1809

RUN curl.exe -L -o nircmd-x64.zip https://www.nirsoft.net/utils/nircmd-x64.zip && mkdir nircmd && tar -xf nircmd-x64.zip -C nircmd && del nircmd-x64.zip
```

Build
```powershell
docker build --name nircmd .
```

```powershell
docker run --rm -it nircmd

C:\> start /b nircmd\nircmdc.exe qbox "Open Notepad?" "question" "notepad.exe"

C:\> tasklist /fi "IMAGENAME eq nircmdc.exe"
Image Name                     PID Session Name        Session#    Mem Usage
========================= ======== ================ =========== ============
nircmdc.exe                   1932 Services                   1      6,224 K

C:\> nircmd\nircmdc.exe win dlgclick title "question" yes

C:\> tasklist /fi "IMAGENAME eq nircmdc.exe"
INFO: No tasks are running which match the specified criteria.

C:\> tasklist /fi "IMAGENAME eq notepad.exe"
Image Name                     PID Session Name        Session#    Mem Usage 
========================= ======== ================ =========== ============
notepad.exe                   1388 Services                   1      8,556 K
```


