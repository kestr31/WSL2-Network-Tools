## 0. Functionalities

- Enables remote access to WSL2 services from external hosts
- Enables access to Windows localhost from WSL2
- Auto start preferred WSL2 services based on user's need

## 1. Quick Start

1. [Grant permission to run `sudo` command without certification to user](#21-dependencies)
2. [Enable Systemd on WSL](#221-enable-systemd-on-wsl)
3. [Add given shell commands on `~/.bashrc` or `~/.zshrc`](#222-access-windows-localhost-from-wsl2)
4. [Specity list of service ports you want to make it accessible from the external network](#223-add-list-of-wsl-ports-you-want-to-make-it-accessible-from-external-network)
5. [Grant permission to run user-defined PowerShell script](#224-modifying-powershell-script-directory-and-granting-powershell-script-run-permission)
6. [Modify `service-auto-start.bat` based directory to `wsl-network.ps1`](#224-modifying-powershell-script-directory-and-granting-powershell-script-run-permission)
7. Run `service-auto-start.bat` or [make it to be run on every Windows startup](#225-run-service-auto-startbat-on-windows-startup)

## 2. Prequisites & Configuration

### 2.1 Dependencies

- Install `sudoers` on WS

```shell
sudo apt update && sudo apt install -y vim
sudo visudo
```

- On `/etc/sudoers`, write a  following line on the last line
- After then, user of username \<USERNAME\> do not needs password for running `sudo` commands

```shell
<USERNAME>    ALL=(ALL:ALL) NOPASSWD: ALL
```

- For example, if username is johnDoe:

```shell
johnDoe      ALL=(ALL:ALL) NOPASSWD: ALL
```

### 2.2 Automation Scritps

#### 2.2.1 Enable Systemd on WSL

- WSL currently supports use of Systemd
- Therefore, previous methos which uses addtional shell script is not needed anymore
<br/>

- Find `/etc/wsl.conf`. If there is no such file, add it.

```shell
sudo touch /etc/wsl.conf
```

- Then, write following contents on `/etc/wsl.conf`

```conf
[boot]
systemd=true
```

- This can be done by following command

```shell
echo '[boot]\nsystemd=true' | sudo tee -a /etc/wsl.conf
```

- Exit WSL and restart WSL

```powershell
exit
wsl --shutdown
wsl
```

#### 2.2.2 Access Windows localhost from WSL2

- IP of windows localhost can be found via `ip route` on WSL2 environment
    - For result `default via 172.xx.xx.xx dev eth0`, 172.xx.xx.xx is a ip of Windows localhost
    - This value changes on every boot since it's a vertual network
- Set windows localhost ip as an environment variable `winhost` by bashrc/zshrc
<br/>

- `~/.bashrc` or `~/.zshrc`

```shell
...
export winhost=$(cat /etc/resolv.conf | grep nameserver | awk '{ print $2 }')
if [ ! -n "$(grep -P "[[:space:]]winhost" /etc/hosts)" ]; then
    printf "%s\t%s\n" "$winhost" "winhost" | sudo tee -a "/etc/hosts"
fi
```

- Following line in `wsl-network.ps1` enables Windows localhost access from WSL2

```powershell
# Allow inbound traffic from WSL network. Allows access from WSL2 to Windows localhost
New-NetFirewallRule -DisplayName "WSL" -Direction Inbound  -InterfaceAlias "vEthernet (WSL)"  -Action Allow
...
```

#### 2.2.3 Add List of WSL Ports You Want to Make it Accessible from External Network

- This is configurable on line 16 of `wsl-network.ps1`

```powershell
...
#[Ports]
#All the ports you want to forward separated by coma
$ports=@(22,8888);
...
``` 

- Assign ports you want to setup to `$ports` in form of integers separated in comma
- For example, default `wsl-network.ps1` allows external connection on SSH and HTTTP ports (Default setup)


#### 2.2.4 Modifying PowerShell script directory and Granting PowerShell script run permission

- On `service-auto-start.bat`, modify directory to the PowerShell script based on PowerShell script directory

```bat
:: Run PowerShell script for network access settings
C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -ExecutionPolicy Bypass -File "C:\Users\<USERNAME>\DIRECTORY\TO\wsl-network.ps1"
```

- For example, if Windows username is johnDoe and directory is $USER\Documents\Scripts...

```bat
:: Run PowerShell script for network access settings
C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -ExecutionPolicy Bypass -File "C:\Users\johnDoe\Documents\Scripts\wsl-network.ps1"
```

- On default configuration, PowerShell does not allow run of user-defined scripts.
    - Grant permission by following command:

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

#### 2.2.5 Run `service-auto-start.bat` on Windows startup

- Open `task scheduler (작업 스케줄러)` with admin permission
- Create new task with following settings:
    - General - Run with privileged permission (가장 높은 수준의 권한으로 실행)
    - Trigger - Create - Task Start (작업 시작): When Logon (로그온할 때)
    - Function (동작) - Create - Function: Run Program - Choose directory to `service-auto-start.bat`

## Reference

1. [WSL2, 외부 네트워크와 연결하기](https://codeac.tistory.com/118)
2. [How to connect wsl to a windows localhost?](https://superuser.com/questions/1535269/how-to-connect-wsl-to-a-windows-localhost)