---
title: 리눅스 종합문제 1
author: minyeokue
date: 2024-03-07 18:00:00 +0900
last_modified_at: 2024-03-14 22:00:00 +0900
categories: [Exercise]
tags: [Linux]
render_with_liquid: false

toc: true
toc_sticky: true
---

리눅스 기본 명령어 및 하드디스크 레이드

---

## 1. /linux/test 디렉토리 생성 후 vi 편집기로 "test mail" 입력 후 test.txt로 저장

    [root@localhost ~]# mkdir /linux
    [root@localhost ~]# mkdir /linux/test
    [root@localhost ]# cd /linux/test
    [root@localhost test]# vim test.txt
    test mail

    :wq

리디렉션을 통해 하는 방법
(/linux/test/ 까지의 디렉토리가 생성되어 있을 경우)

>[root@localhost ~]# echo test mail > /linux/test/test.txt우


## 2. /usr/bin 디렉토리의 내용을 ls 확인한 내용을 리디렉션을 이용해서 a.lst로 저장 하시오.

    [root@localhost ~]# ls /usr/bin > a.lst


## 3./linux/test 에 크기가 0byte인 abc.txt 생성

    [root@localhost ~]# cd /linux/test
    [root@localhost test]# touch abc.txt

다음 문제로 이어진다.

## 4. abc.txt 파일을 cba.txt 복사

    [root@localhost test]# cp ./abc.txt ./cba.txt

다음 문제로 이어진다.

## 5. cba.txt 파일을 aaa.txt 이름 변경

    [root@localhost test]# mv ./cba.txt ./aaa.txt

다음 문제로 이어진다.

## 6. /linux/test 하위 디렉토리로 test2 디렉토리 생성후 /linux/test 디렉토리 'a' 시작하는 모든 파일 이동

    [root@localhost test]# mkdir test2
    [root@localhost test]# mv a* ./test2/

다음 문제로 이어진다.

## 7. test2 디렉토리 삭제

    [root@localhost test]# rm -rf test2


## 8. 그룹생성 : linuxgroup

    [root@localhost ~]# groupadd linuxgroup

다음 문제로 이어진다.

## 9. 사용자 생성 시나리오

사용자 이름 :hgd
암호 : 111111(단 암호화 해서 생성)
사용자 ID : 555
그룹 : linuxgroup
홈디렉토리 : /linux/test/hgd
셀 : /bin/sh 로 생성

    [root@localhost ~]# useradd -p 'openssl passwd 111111' -u 555 -g linuxgroup -d /linux/test/hgd -s /bin/sh hgd

다음 문제와 이어진다.

## 10. hgd 사용자 홈디렉토리 /home/hgd 셀 : /bin/bash 셀로 수정하시오..

    [root@localhost ~]# usermod -s /bin/bash hgd


## 11. 사용자 생성 시나리오

hgd2 사용자 계정 생성 암호 : 111111
hgd2 사용자 로그인 못하도록 설정
hgd2 사용자가 다시 로그인 가능 하도록 설정

    [root@localhost ~]# useradd -p 'openssl passwd 111111' hgd2
    [root@localhost ~]# usermod -s /bin/false hgd2
    [root@localhost ~]# usermod -s /bin/bash hgd2

생성된 계정들은 /etc/passwd에 보관되는데 각 계정은 구분자(**:**)를 이용해 7개 필드로 관리된다.

root를 예시로 들면,

    root  :  x  :  0  :  0  :  root  :  /root  :  /bin/bash
    이름   패스워드 UID   GID   계정정보  홈디렉토리  사용자 로그인 쉘

위의 순서와 각 필드의 의미인데, x라고 저장된 필드는 /etc/shadow 파일에 암호화되어 저장된다.

새 사용자를 추가할 때 **-s** 옵션으로 사용자 로그인 쉘을 지정할 수 있다.

다음 리스트로 설명하겠다.

- /bin/bash

사용자 계정에 로그인 했을 때 기본적으로 사용하는 쉘

- /bin/false

shell, ssh 접근, 홈 디렉토리 등 모든 것이 제한됨

- /sbin/nologin

shell, ssh 접근 및 홈 디렉토리는 제공하지 않으나 ftp 접근은 허용
주로 사용하지 않는 디폴트 데몬 계정에 적용, 보안 상 불필요한 계정에 nologin을 적용한다.
시스템 계정이나 apache 등이 해당된다.


## 12. hgd2 홈디렉토리까지 한 번에 삭제하도록 옵션을 이용해서 삭제 하시오

    [root@localhost ~]# userdel -r hgd2


## 13. root 권한으로 /tmp 폴더에 용량이 0 byte인 a.txt 문서 생성 권한 확인 후 hgd 사용자가 a.txt 문서를 수정 가능 하도록 권한 설정 후 hgd가 수정 후 저장 하도록 하시오.

    [root@localhost ~]# touch /tmp/a.txt
    [root@localhost ~]# chmod 777 /tmp/a.txt


## 14. a.txt 파일의 소유자 : hgd 소유그룹 : linuxgroup로 설정하시오

    [root@localhost ~]# chown hgd.linuxgroup /tmp/a.txt


## 15. ln 명령어 활용한 링크파일 생성 시나리오

/linktest 디렉토리 생성, 그 안에 basefile 생성 vi 편집기를 이용해서 적당히 편집 
하드링크명 : hardlink 
소프트링크명 : softlink로  하드링크와 소프트 링크 생성 후 ls -il 로 inode을 값을 확인하시오mkdir

    [root@localhost ~]# mkdir /linktest
    [root@localhost ~]# vim /linktest/basefile
    [root@localhost ~]# ln /linktest/basefile /linktest/hardlink
    [root@localhost ~]# ln -s /linktest/basefile /linktest/softlink

    [root@localhost ~]# ls -il /linktest
    2214145 -rw-r--r--. 2 root root 17  3월  9일  09:26 basefile
    2214145 -rw-r--r--. 2 root root 17  3월  9일  09:26 hardlink
    1759458 lrwxrwxrwx. 1 root root 22  3월  9일  09:27 softlink -> /root/test/basefile


##  16.  yum 명령으로 httpd 데몬 설치 후 /var/www/html 디렉토리에 index.html 파일 생성 "<h2>Web Site</h2>" 내용 입력 사이트 확인..

    [root@localhost ~]# yum -y install httpd

    ...
    설치 완료!
    ...

    [root@localhost ~]# cd /var/www/html
    [root@localhost html]# vim index.html
    <h2>Web Site</h2>

    :wq


## 17. 웹사이트 확인 후 httpd 데몬이 리부팅 후에도 계속 사용 가능하도록 설정하시오..

    [root@localhost ~]# systemctl enable httpd

or

    [root@localhost ~]# systemctl enable --now httpd

단 아래의 명령은 활성화하며 시작하는 명령어로 **systemctl start httpd && systemctl enable httpd** 와 같다


## 18. 압축 명령 시나리오

tar 명령을 이용해서 /etc 디렉토리를 etc.tar.gz으로 압축하시오
압축한 etc.tar.gz 파일을 압축을 해제하시오.

    [root@localhost ~]# tar cfz etc.tar.gz /etc
    [root@localhost ~]# tar xfz etc.tar.gz


## 19. find 명령을 이용해서 /etc 디렉토리에 확장자가 .conf 끝나는 파일들을 리디렉션 기능을 이용해서 b.lst 파일로 저장하시오

    [root@localhost ~]# find /etc -name *.conf > b.lst


## 20. 쉘 스크립트 시나리오

매월 15일 새벽 4시1에 자동백업이 되도록 설정하시오.
쉘 스크립트로 처리하시오..

백업 위치 :/backup에 /home 디렉토리 백업
자동스크립트 파일 : backup.sh
파일 이름 : backup월일.tar.gz 형태로 만드시오..

    [root@localhost ~]# cd /etc/cron.monthly
    [root@localhost cron.monthly]# vim backup.sh

    datetime=$(date "+%M%d")
    fname="backup$datetime.tar.gz"

    tar cfzP /backup/$fname /home

    :wq
    [root@localhost cron.monthly]# mkdir /backup 
    [root@localhost cron.monthly]# vim /etc/crontab
    
    ...
    # Example of job definition:
    # .---------------- minute (0 - 59)
    # |  .------------- hour (0 - 23)
    # |  |  .---------- day of month (1 - 31)
    # |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
    # |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
    # |  |  |  |  |
    # *  *  *  *  * user-name command to be executed

     01 04 15  *  * root    run-parts   /etc/cron.monthly

    :wq


## 22. 하드디스크 추가 시나리오

용량 : 1Giga 
파티션을 2개로 설정하시오..(논리적인 파티션 2개 생성)
/dev/sdb1 /dev/sdb2 

/mydata1 /mydata2  
각각의 파티션을 마운트 하시오..
/etc/fstab 등록 시키시오

    [root@localhost ~]# fdisk /dev/sdb

    n
        e
        1
        (enter)
        (enter)
    
    n
        l
        (enter)
        +512M
    
    n
        l
        (enter)
        (enter)
    w

    [root@localhost ~]# mkfs -t xfs /dev/sdb5

    ...

    [root@localhost ~]# mkfs -t xfs /dev/sdb6

    ...

    [root@localhost ~]# mkdir /mydata1
    [root@localhost ~]# mkdir /mydata2
    [root@localhost ~]# mount /dev/sdb5 /mydata1
    [root@localhost ~]# mount /dev/sdb6 /mydata2
    [root@localhost ~]# vim /etc/fstab


fstab 등록은 추후 작성하도록 하겠다.


## 24. 레이드 시나리오

/dev/sdc /dev/sdd -> raid 0설정
/dev/sde /dev/sdf -> raid 1로 구성
/raid0data   /raid1data
각각 마운트 하시오

/raid0data /raid1data에 test 파일을 생성한다.
/dev/sdf  하드디스크를 제거 후  내결함성 확인
하드 디스크 하나를 추가해서 원래의 상태로 복구하시오..


추후 작성하도록 하겠다.
