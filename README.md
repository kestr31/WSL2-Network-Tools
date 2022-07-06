## 0. Functionalities

- Enables remote access to WSL2 services from external hosts
- Enables access to Windows localhost from WSL2
- Auto start preferred WSL2 services based on user's need

## 1. Quick Start

1. Install [Docker](#### 2.1.1 Docker) & [SSH](#### 2.1.2 SSH) on WSL2 Environment
2. [Grant permission to run `sudo` command without certification to user](#### 2.1.3 Sudoers)
3. Copy `init-wsl` to `/etc`. (As `/etc/init-wsl`)
4. [Grant permission to run user-defined PowerShell script](#### 2.2.4 Modifying PowerShell script directory & Granting PowerShell script run permission)
5. [Modify `service-auto-start.bat` based on `wsl-network.ps1`'s location](#### 2.2.4 Modifying PowerShell script directory & Granting PowerShell script run permission)
6. Run `service-auto-start.bat` or [make it to be run on every Windows startup](#### 2.2.5 Run `service-auto-start.bat` on Windows startup)

## 2. Prequisites & Configuration

### 2.1 Dependencies

#### 2.1.1 Docker

- docker-ce

```shell
curl -fsSL https://get.docker.com -o get-docker.sh
DRY_RUN=1 sh ./get-docker.sh
sudo service docker start
sudo usermod -aG docker $USER
```

- NVIDIA Container Toolkit

```shell
distribution=$(. /etc/os-release;echo $ID$VERSION_ID) \
      && curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
      && curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | \
            sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
            sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

```shell
sudo apt-get update && sudo apt-get install -y nvidia-docker2
sudo service restart docker
```

```shell
sudo docker run --rm --gpus all nvidia/cuda:11.0.3-base-ubuntu20.04 nvidia-smi
```

#### 2.1.2 SSH

```shell
sudo apt update
sudo apt install -y openssh-server
```

- `/etc/ssh/sshd_config`

```shell
PORT 22
PasswordAuthentication yes
```

#### 2.1.3 Sudoers

```shell
sudo apt update && sudo apt install -y vim
sudo visudo
```

- `/etc/sudoers`

```shell
root        ALL=(ALL:ALL) ALL
USERNAME    ALL=(ALL:ALL) NOPASSWD: ALL
```

### 2.2 Automation Scritps

#### 2.2.1 Service auto start

- In default, `./init-wsl` is made to run docker and SSH service on startup
    - Modify it based on your need
- If script is not executable by `sudo ./init-wsl`, grant permission

```shell
chmod +x ./init-wsl
```

- Copy it into designated directory

```shell
sudo cp ./init-wsl /etc/init-wsl
```

- Following lines in `service-auto-start.bat` execute `init-wsl` on run

```bat
...
:: Run services on WSL2 via shell script /etc/init-wsl
C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe wsl -d Ubuntu -u merlin sudo /etc/init-wsl
```

#### 2.2.2 Network mapping: Access Windows localhost from WSL2

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

#### 2.2.3 Network mapping: Access WSL2 service from external host

- Whole lines of `wsl-network.ps1` except first two makes it possible via creating firewall rules

#### 2.2.4 Modifying PowerShell script directory & Granting PowerShell script run permission

- On `service-auto-start.bat`, modify it based on PowerShell script directory

```bat
:: Run PowerShell script for network access settings
C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -ExecutionPolicy Bypass -File "C:\Users\<USERNAME>\DIRECTORY\TO\wsl-network.ps1"
```

- For example, if Windows username is john and directory is $USER\Documents\Scripts...

```bat
:: Run PowerShell script for network access settings
C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -ExecutionPolicy Bypass -File "C:\Users\john\Documents\Scripts\wsl-network.ps1"
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

1. [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
2. [NVIDIA Container Toolkit Installation Guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html)
3. [WSL2, 외부 네트워크와 연결하기](https://codeac.tistory.com/118)
4. [How to connect wsl to a windows localhost?](https://superuser.com/questions/1535269/how-to-connect-wsl-to-a-windows-localhost)