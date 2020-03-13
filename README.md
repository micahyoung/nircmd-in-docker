# Experiments with nircmd in Windows Docker containers

## Docs
Useful features:
* Manipulating Windows and Dialog boxes: http://nircmd.nirsoft.net/win.html
* For finding GUI child control IDs https://www.nirsoft.net/utils/gui_prop_view.html

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

### Open a file in notepad, edit, save and exit

`Dockerfile`
```Dockerfile
FROM mcr.microsoft.com/windows/servercore:1809

USER ContainerUser

RUN curl.exe -L -o nircmd-x64.zip https://www.nirsoft.net/utils/nircmd-x64.zip && mkdir nircmd && tar -xf nircmd-x64.zip -C nircmd && del nircmd-x64.zip

RUN echo My Old Text > myfile.txt

# cat before
RUN type myfile.txt

# 1. Open existing file in notepad (needs to be backgrounded; reason TBD)
# 2. write new text to Text Box (consitently ID 15 via https://www.nirsoft.net/utils/gui_prop_view.html)
# 3. click invisible "Save" (consitently ID 3)
# 4. close window
# 5. Ignore nircmd exit code of 1073757860
RUN start /b notepad.exe myfile.txt & \
    c:\nircmd\nircmdc cmdwait 1000 win dlgsettext stitle "myfile" 15 "My New Text" & \
    c:\nircmd\nircmdc win dlgclick stitle "myfile" 3 & \
    c:\nircmd\nircmdc win close stitle "myfile" & \
    exit 0

# cat after
RUN type myfile.txt
```

Build
```powershell
docker build --name nircmd-notepad-test .
```

Output
```powershell
...
Step 5/7 : RUN type myfile.txt
 ---> Running in 4986640420ff
My Old Text 
Removing intermediate container 4986640420ff
 ---> 89914dc3114a
Step 6/7 : RUN start /B notepad.exe myfile.txt &     c:\nircmd\nircmdc cmdwait 1000 win dlgsettext stitle "myfile" 15 "My New Text" &     c:\nircmd\nircmdc win dlgclick stitle "myfile" 3 &     c:\nircmd\nircmdc win close stitle "myfile" &     exit 0
 ---> Running in 5f506be9d8f3
Removing intermediate container 5f506be9d8f3
 ---> 6b5a4c2d5dbf
Step 7/7 : RUN type myfile.txt
 ---> Running in 7f4c6e686233
My New Text
...
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


