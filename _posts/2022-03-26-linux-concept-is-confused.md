---
title: 리눅스 헷갈리는 개념 포스팅
date: 2022-03-12 16:54:00 +09:00
categories: [linux]
tags: [linux, chmod, permission, log, man]
image:
    src: /assets/images/linux-logo.png
    alt: linux logo
---

 개발을 하면서 서버의 운영체제는 대부분 리눅스를 사용할 것입니다. 저 또한 리눅스 환경의 서버에 배포하고 운영하고 있으며 서버에 조치사항이 있을 때 터미널로 접근해 해결하고 있습니다. 리눅스의 여러가지 개념 중 헷갈리고 잘 기억되지 않는 것들이 몇 개 있습니다. 파일권한, 데몬 프로세스, Inode 등 리눅스를 사용하는 프로그래머에겐 필수적으로 알아야할 내용이지만 위 개념들 까지 생각하면서 사용하고 있지는 않아 계속 잊어버리는데요. 이번에 글로 정리해놓아 제대로 기억해보려고 합니다.

## 프로세스

 프로세스란 메모리에 적재되어 실행하고 있는 프로그램을 의미합니다. 제어 터미널(Teletype)에 의해 실행되거나 데몬 프로세스로 실행됩니다. `ps -ef`를 통해 프로세스를 확인 할 수 있습니다.
 ![프로세스 상태](/assets/images/process-status.png)

 위 정보를 통해 프로세스가 어떤 정보를 가지고 있는지 대략 알 수 있습니다. UID는 이 프로세스를 실행시킨 유저의 아이디이고 PID는 프로세스 ID입니다. C는 프로세스 스케줄링에서 CPU사용량을 측정할 때 사용되었지만 지금은 사용하지 않는다고 합니다. STIME은 프로세스 시작시간, TTY는 제어 터미널이고 ?는 제어터미널 없이 실행되는 데몬 프로세스입니다. TIME은 CPU를 사용한 시간이고 CMD는 실행되는 프로그램의 위치를 알려줍니다.

### foreground 프로세스

 해당 제어 터미널에서 프로세스를 실행한 뒤, 수행이 종료될 때까지 사용자는 다른 입력을 못하는 프로세스입니다. 현재 터미널에서 실행되고 있는 프로세스 리스트는 `jobs`를 통해 검색 가능하고 각 작업 번호(job number)와 프로세스 id(pid)가 보입니다.
 ```shell
 jobs
 ```
 작업 리스트 중 foreground로 실행시키고 싶은 프로세스가 있다면 작업번호로 가져올 수 있습니다.
 ```shell
 fg [작업 번호]
 ```
### background 프로세스

 프로세스를 실행한 뒤, 사용자가 다른 입력이 가능하고 실행되는 프로세스입니다. 표준출력은 따로 redirect하지 않는 이상 터미널에 그대로 출력됩니다. background 프로세스 또한 `jobs`를 통해 조회 가능합니다.
 현재 foreground로 실행되고 있는 프로세스를 background로 바꾸고 싶다면 먼저 ctrl+z로 프로세스를 중지하세요. 그러면 중지되고 `jobs`로 작업을 조회해보면 STOPPED로 표시되어 있을 겁니다. `bg`와 작업번호로 백그라운드에서 실행할 수 있습니다.
 ```shell
 bg [작업 번호]
 ```
 또한 프로세스를 시작할 때 끝에 &를 붙혀주면 백그라운드에서 실행합니다.

### 데몬 프로세스

 백그라운드 프로세스 중 부모 프로세스 id가 1 혹은 다른 데몬 프로세스인 프로세스를 데몬 프로세스라고 합니다. 쉘에서 실행한 백그라운드 프로세스는 부모 프로세스 id를 타고 올라가면 사용자 쉘 프로세스와 연결 되어있습니다. 따라서 사용자가 현재 쉘을 종료하고 나가버리면 해당 프로세스도 종료됩니다. 반면 데몬 프로세스는 부모가 데몬 프로세스이거나 프로세스 id가 1인 init 프로세스이기 때문에 종료되지 않습니다. 만약 백그라운드로 실행시킨 프로세스를 tty를 끊어도 실행시키고 싶다면 `nohup` 명령을 프로세스 실행명령 앞에 붙이면 됩니다. 데몬 프로세스의 종류는 stand-alone 데몬과 inted 데몬으로 나뉘어집니다.

### Stand-alone 데몬

 stand-alone 데몬은 항시 프로세스가 실행 중이며 소켓과 메모리에 상주해 있습니다. 요청이 들어왔을 때 즉각적으로 처리할 수 있어 빠르지만 자원낭비가 있습니다. 자주 요청이 들어오면 서버는 stand-alone으로 띄우는게 좋습니다. /etc/init.d에서 stand-alone 프로세스 구동 스크립트를 확인할 수 있습니다.

### xinetd 데몬

 stand-alone으로 띄워져 있는 xinetd 서비스가 들어오는 요청에 따라 적절한 서비스를 띄워 요청을 처리하는 방식입니다. 요청이 들어올 때 마다 프로세스를 올리는 비용이 있어 느리지만 자원 효율성에서 좋습니다. 자주 요청이 들어오지 않는 서버에 적절합니다. /etc/xinetd.conf에 설정파일이 있고, 각 서비스별 설정파일은 /etc/xinetd.d에 따로 둡니다. 
 
### 좀비 프로세스

 만약 프로세스 실행 중 부모 프로세스나 자식 프로세스가 먼저 종료되면 어떻게 될까요? 부모 프로세스가 먼저 종료된 경우에는 고아 프로세스(Orphan process)가 됩니다. 이런 고아 프로세스는 PPID가 1로 설정되어 종료되지 않고 계속 실행되게 됩니다. (만약 tty 프로세스가 종료된다면 hup시그널로 프로세스 종료) 반대로 자식 프로세스가 먼저 종료된다면 자식 프로세스는 부모 프로세스에게 상태를 알리기 위해 최소한의 자원을 가지고 기다리게 됩니다. 이런 프로세스가 좀비 프로세스(Zombie process)이고 `ps aux | grep Z`명령어를 통해 확인할 수 있습니다. 이런 경우를 막기 위해 부모 프로세스에서는 wait 시스템 콜로 자식 프로세스를 항상 회수해야 합니다.
 
## 파일 시스템

 리눅스에는 다양한 파일 시스템이 있다. 그 중 EXT계열의 파일시스템이 가장 많이 쓰이고 안정성, 속도, 공간 효율성 측면에서 파일시스템마다 차이가 있다. 공통적인 부분에 대해 설명하도록 하겠다.  
 파일시스템은 디스크를 블록별로 나누고 그 블록을 설명하는 inode라는 개념을 가져온다. inode에는 사용자와 그룹 정보, 권한 정보, 파일 종류에 대한 정보가 있고 해당 파일의 block의 개수와 위치 정보가 있어 알 수 있다. 예를 들어 `/home/oopty/test.txt`라는 파일을 찾는다고 가정해보자. 루트 디렉토리의 inode는 항상 2번이고 inode 2번의 정보를 보면 루트 디렉토리의 block 번호를 알 수 있다. 해당 block 번호로 가면 루트 디렉토리 하위의 디렉토리와 파일 이름과 inode 번호가 보인다. 그 중 우리는 `home`이라는 디렉토리의 inode를 찾을 것이고 같은 과정을 반복해서 `test.txt` 파일을 찾을 것이다. 이 과정이 진행되는 중 디렉토리와 파일의 권한을 확인하면서 진행을 하고 권한이 위배된다면 `Permission Denied` 예외를 발생할 것이다.

## 디렉토리 구조

 리눅스에 다양한 배포판이 있지만 디렉토리 구조는 비슷하고 대표적인 디렉토리의 쓰임은 비슷한 거 같다. 그 내용을 정리하겠다.
 - /bin
   - 기본적인 명령어가 있는 디렉토리 root와 일반사용자 모두가 사용할 수 있고 PATH 환경변수에 설정되어 있어 디렉토리 안에 있는 파일들은 파일명만으로 실행할 수 있다.
 - /boot
   - 리눅스 부트로더의 설정파일이 존재하는 곳, 예를 들어 GRUB와 같은 부트로더의 설정파일들이 존재
 - /dev
   - 디바이스 정보를 저장하는 곳, 여기서 하드디스크의 파티션을 나누고 파일시스템을 만들고 마운트하는 파일들이 존재한다.
 - /etc
   - 시스템 대부분의 설정파일이 존재
 - /home
   - 사용자의 홈 디렉토리
 - /lib
   - 커널 모듈 파일과 라이브러리 파일이 있는 곳, 커널이 필요로하는 라이브러리와 모듈이 존재
 - /media
   - DVD, CD-ROM, USB같은 것들의 마운트 포인트
 - /mnt
   - 일시적인 마운트를 위한 마운트 포인트
 - /tmp
   - 공용 임시 디렉토리, 모든 사용자가 공동으로 사용하고 시스템이 주기적으로 청소한다. 해킹의 주요 대상이 되니까 사용하는데는 주의가 필요함
 - /usr
   - root가 아닌 일반 사용자들이 주로 이용하는 디렉토리
 - /var
   - 시스템 운용에서 일시적으로 데이터를 생성하거나 로그를 남기는데 사용되는 디렉토리
 - /lost+found
   - 파일시스템마다 존재할 수 있고 fsck나 e2fsck같은 파일시스템 체크 및 복구 유틸리티 실행 후에 생성이 되는 것, 시스템의 오류나 강제종료로 인해 생성되는 파일들이고 시스템 부팅시 파일 시스템 체크 유틸리티가 속한 디렉토리가 없는 파일들을 여기에 모아 놓는다.

## 출처

 - [POSIX 알아보기 #1: LINUX(리눅스) 파일시스템의 종류와 특징](https://medium.com/naver-cloud-platform/posix-%EC%95%8C%EC%95%84%EB%B3%B4%EA%B8%B0-1-linux-%EB%A6%AC%EB%88%85%EC%8A%A4-%ED%8C%8C%EC%9D%BC-%EC%8B%9C%EC%8A%A4%ED%85%9C%EC%9D%98-%EC%A2%85%EB%A5%98%EC%99%80-%ED%8A%B9%EC%A7%95-96a2e93e33b3)
 - [리눅스 디렉토리 구조](https://webdir.tistory.com/101)