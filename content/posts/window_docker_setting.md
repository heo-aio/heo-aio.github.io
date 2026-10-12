+++
title= "[Docker] Windows OS에서 Docker 쓰려면? WSL2 설정부터~
date= 2026-10-12
draft= false
categories= ["dev"]
tags= ["Docker", "WSL2", "Windows", "Docker Desktop", "환경설정"]
summary= "Docker Desktop만 깔면 되는 줄 알았는데 WSL2부터 켜야 했다. 명령어를 순서대로 따라 치면서 Windows에서 Docker를 돌리기까지의 과정을 정리했다."
+++

AI 서비스를 만들다 보면 "내 컴퓨터에선 되는데요?" 문제가 꼭 나온다고 해서 Docker를 써보기로 했다. 그런데 Windows에서는 Docker Desktop 설치 파일만 받으면 끝이 아니었다. **WSL2**라는 게 먼저 켜져 있어야 한다. 명령어가 한꺼번에 나와서 "이게 다 뭐지?" 싶었는데, 하나씩 의미를 확인하면서 정리했다.

## 목차
- 🧭 Docker와 WSL2는 왜 같이 나올까
- ✅ 시작 전에 확인할 것
- 🪜 설치 순서 한눈에 보기
- ⚙️ 1단계 - WSL, 가상 머신 플랫폼 기능 켜기
- 🔄 2단계 - 재부팅 후 WSL2 커널 업데이트
- 🐳 3단계 - Docker Desktop 설치
- 🧪 4단계 - 설치 확인하기
- 🛠️ 막혔던 부분
- 💡 정리하면

---

## 🧭 Docker와 WSL2는 왜 같이 나올까

Docker는 **리눅스 기술(컨테이너)** 로 만들어진 도구다. 그런데 Windows는 리눅스가 아니다. 그래서 Windows 안에서 리눅스를 가볍게 돌려줄 무언가가 필요한데, 그게 **WSL2(Windows Subsystem for Linux 2)** 다.

```
[Windows] → [WSL2 (가벼운 리눅스)] → [Docker 엔진] → [컨테이너]
```

비유하면 Docker는 **리눅스 전용 주방 기구**고, WSL2는 Windows 집 안에 만들어주는 **작은 리눅스 주방**이다. 주방이 있어야 기구를 쓸 수 있다.

## ✅ 시작 전에 확인할 것

- Windows 10 (버전 2004 이상) 또는 Windows 11
- **BIOS에서 가상화(Virtualization) 기능이 켜져 있어야 한다.**
  - 작업 관리자 → 성능 → CPU에서 `가상화: 사용` 이라고 보이면 OK
- PowerShell을 **관리자 권한**으로 실행해야 한다.
  - 시작 메뉴에서 PowerShell 검색 → 우클릭 → 관리자 권한으로 실행

<!-- TODO: 작업 관리자 가상화 '사용' 화면 캡처 붙이기 -->

## 🪜 설치 순서 한눈에 보기

| 단계 | 하는 일 | 쓰는 것 |
|---|---|---|
| 1 | WSL 기능, 가상 머신 플랫폼 기능 켜기 | `dism.exe` |
| 2 | 재부팅 → WSL2 커널 업데이트 → 기본 버전을 2로 | `wsl --update`, `wsl --set-default-version 2` |
| 3 | Docker Desktop 설치 | 설치 파일 |
| 4 | 정상 동작 확인 | `docker run hello-world` |

## ⚙️ 1단계 - WSL, 가상 머신 플랫폼 기능 켜기

관리자 권한 PowerShell에서 아래 두 줄을 차례로 실행한다.

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart

dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

`dism.exe`는 Windows의 **기능을 켜고 끄는 도구**다. 제어판의 "Windows 기능 켜기/끄기"를 명령어로 하는 거라고 보면 된다.

- 첫 번째 줄: **WSL(리눅스 하위 시스템)** 기능을 켠다.
- 두 번째 줄: **가상 머신 플랫폼** 기능을 켠다. WSL2가 이 기능 위에서 돌아간다.
- `/norestart`: 켜고 나서 바로 재부팅하지 말라는 뜻이다. 두 개를 다 켠 다음에 한 번에 재부팅하려고 붙인다.

성공하면 `작업을 완료했습니다.` 라고 나온다.

<!-- TODO: dism 실행 결과 캡처 붙이기 -->

### Windows 기능 켜기/끄기에서도 확인할 수 있다

위 `dism.exe` 두 줄은 **GUI로 체크하는 것과 같은 결과**다. 시작 메뉴에서 `Windows 기능 켜기/끄기`를 검색해서 열면 아래 두 항목이 체크되어 있어야 한다.

| 체크할 항목 | dism 명령어의 featurename |
|---|---|
| ✅ Linux용 Windows 하위 시스템 | `Microsoft-Windows-Subsystem-Linux` |
| ✅ 가상 머신 플랫폼 | `VirtualMachinePlatform` |

- 명령어로 켰으면 여기서 체크 표시가 자동으로 들어가 있다. 반대로 여기서 직접 체크하고 확인을 눌러도 같은 효과다.
- 체크하고 나면 재부팅하라는 안내가 나온다.

참고로 비슷해 보이는 항목이 몇 개 더 있는데, **WSL2 기반 Docker Desktop에서는 필수가 아니다.**

| 항목 | 필요한가? |
|---|---|
| Hyper-V | 필수 아님 (Docker Desktop을 WSL2 방식으로 쓸 때) |
| Windows 하이퍼바이저 플랫폼 | 필수 아님 |
| 컨테이너 | 필수 아님 (Windows용 컨테이너를 쓸 때만 해당) |

<!-- TODO: Windows 기능 켜기/끄기 창 캡처 붙이기 (내가 체크한 항목 표시) -->

## 🔄 2단계 - 재부팅 후 WSL2 커널 업데이트

**여기서 일단 컴퓨터를 재부팅한다.** 이걸 건너뛰고 다음 명령어를 치면 에러가 나기 쉽다.

재부팅 후 WSL2용 리눅스 커널 업데이트 패키지를 설치한다. 아래 주소로 접속하면 `.msi` 파일이 다운로드되고, 실행해서 설치하면 된다.

```
https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi
```

그다음 PowerShell에서 아래 두 줄을 실행한다.

```powershell
wsl --update
wsl --set-default-version 2
```

- `wsl --update`: WSL을 최신 버전으로 업데이트한다.
- `wsl --set-default-version 2`: 앞으로 만드는 리눅스 배포판의 기본 버전을 **WSL2**로 지정한다.

> ⚠️ 블로그나 문서에서 복사하면 `--update`가 `–update`(긴 대시)로 바뀌어 있는 경우가 있다. 이러면 명령어가 인식되지 않으니 **하이픈 두 개(`--`)** 로 직접 쳐야 한다.

```powershell
wsl --version     # WSL 버전 확인
wsl -l -v         # 설치된 배포판과 버전(1/2) 확인
```

<!-- TODO: wsl --set-default-version 2 실행 결과 캡처 붙이기 -->

### 한 줄로 끝내는 방법도 있다

최신 Windows 10/11에서는 아래 명령어 하나가 위 과정(기능 켜기, 커널 설치, 기본 버전 2 설정)을 대부분 알아서 해준다.

```powershell
wsl --install
```

나는 각 단계가 뭘 하는지 알고 싶어서 하나씩 쳤지만, 빠르게 세팅만 하고 싶다면 이쪽이 편하다. 이 방법은 Ubuntu도 같이 설치해준다.

## 🐳 3단계 - Docker Desktop 설치

1. [Docker 공식 사이트](https://www.docker.com/products/docker-desktop/)에서 **Docker Desktop for Windows**를 받는다.
2. 설치 중 `Use WSL 2 instead of Hyper-V` 옵션이 보이면 **체크**한다.
3. 설치가 끝나면 재부팅(또는 로그아웃)하라고 나올 수 있다.
4. Docker Desktop을 실행하고, 왼쪽 아래 상태가 **Engine running** (초록색)이 될 때까지 기다린다.

<!-- TODO: Docker Desktop 실행 화면 캡처 붙이기 -->

## 🧪 4단계 - 설치 확인하기

PowerShell에서 아래를 차례로 실행해본다.

```powershell
docker --version
docker run hello-world
```

`docker run hello-world`는 Docker가 제대로 동작하는지 확인하는 **테스트용 컨테이너**를 실행한다. 아래와 비슷한 문구가 나오면 성공이다.

```
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

이 명령어가 실행되는 동안 Docker는 이런 일을 한다.

1. 내 컴퓨터에 `hello-world` 이미지가 있는지 확인한다.
2. 없으니까 Docker Hub에서 **다운로드(pull)** 한다.
3. 이미지로 **컨테이너를 만들어 실행**하고 메시지를 출력한다.

<!-- TODO: hello-world 실행 결과 캡처 붙이기 -->

## 🛠️ 생각해야할 것

**1) `WSL 2 requires an update to its kernel component` 에러**
2단계의 커널 업데이트(`.msi`)를 설치하지 않았거나, 재부팅을 안 했을 때 나온다. 재부팅 후 `.msi` 설치 → `wsl --update` 순서로 다시 하면 된다.

**2) 가상화 관련 에러 (`Virtualization support not detected` 등)**
BIOS에서 가상화 기능이 꺼져 있는 경우다. 메인보드마다 이름이 다르다.
- Intel: `Intel Virtualization Technology (VT-x)`
- AMD: `SVM Mode`

**3) `docker` 명령어를 찾을 수 없다고 나올 때**
Docker Desktop이 아직 실행 중이 아니거나, 설치 후 터미널을 새로 안 열어서 그럴 수 있다. Docker Desktop이 켜져 있는지 확인하고 PowerShell을 **껐다 다시 켠다.**

## 💡 정리하면

| 하고 싶은 일 | 쓰는 것 |
|---|---|
| WSL 기능 켜기 | `dism.exe ... Microsoft-Windows-Subsystem-Linux` (= Windows 기능 켜기/끄기의 `Linux용 Windows 하위 시스템`) |
| 가상 머신 플랫폼 켜기 | `dism.exe ... VirtualMachinePlatform` (= Windows 기능 켜기/끄기의 `가상 머신 플랫폼`) |
| WSL 업데이트 | `wsl --update` |
| 기본 버전 WSL2로 지정 | `wsl --set-default-version 2` |
| WSL 버전 확인 | `wsl -l -v` |
| Docker 설치 확인 | `docker --version` |
| 동작 테스트 | `docker run hello-world` |

- Windows에서 Docker는 **WSL2 위에서 돈다.** 그래서 WSL2 세팅이 먼저다.
- 기능을 켠 뒤에는 **재부팅**이 필요하다.
- 명령어를 복사할 때 `--`가 `–`로 바뀌지 않았는지 확인한다.
- 시간이 없다면 `wsl --install` 한 줄로 시작해도 된다.
- 세팅이 끝났으니 다음엔 LlamaIndex RAG 프로젝트를 Docker 컨테이너로 묶어서 실행해볼 예정이다.