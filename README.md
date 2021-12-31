# oracledb-embulk-bigQuery

## 개요

`oracle-server`에 연결해서 `bigQuery`로 이관하는 작업을 수행합니다.

data migration ETL작업 툴은 `embulk`를 이용하겠습니다.

## 시작

## Oracle Server 연결(Oracle Client Setting)

`Hans-On`에서 필요한 자료들은 GCS에 업로드 해두었습니다.

`oracle-instant-basic` : `gs//oracle-client-bucket/oracle-instantclient12.1-basic-12.1.0.2.0-1.x86_64.rpm`

`oracle-instant-devel` : `gs://oracle-client-bucket/oracle-instantclient12.1-basic-12.1.0.2.0-1.x86_64.rpm`

`oracle-instant-sqlplus` : `gs://oracle-sqlplus/ oracle-instantclient12.1-sqlplus-12.1.0.2.0-1.x86_64.rpm`

위 경로대로 파일을 받으신 후에 운영체제에 맞게 진행하시면 됩니다.

혹은 공식적인 [Oracle](http://www.oracle.com/technetwork/indexes/downloads/index.html#database) 사이트에서 받으실 수 있습니다.

`oracle server`와 연동하기 위해 `Oracle Instant Client`를 설치합니다.

버전은 **12.1**을 이용합니다.

## MacOS & Linux 환경 설정

```
$ apt-get install alien
$ alien -i oracle-instantclient12.1-basic-12.1.0.2.0-1.x86_64.rpm
$ alien -i oracle-instantclient12.1-sqlplus-12.1.0.2.0-1.x86_64.rpm
$ alien -i oracle-instantclient12.1-devel-12.1.0.2.0-1.x86_64.rpm
```

```
$ vi /etc/ld.so.conf.d/oracle.conf && sudo chmod o+r /etc/ld.so.conf.d/oracle.conf
/usr/lib/oracle/12.1/client64/lib/
```

```
$ vi /etc/profile.d/oracle.sh && sudo chmod o+r /etc/profile.d/oracle.sh
export ORACLE_HOME=/usr/lib/oracle/12.1/client64
export TNS_ADMIN=/etc/oracle //tnsnames.ora
```

```
$ vi ~/.bash_profile
PATH=$ORACLE_HOME/bin:$PATH
LD_LIBRARY_PATH=$ORACLE_HOME/lib
export LD_LIBRARY_PATH
export PATH

$ source ~/.bash_profile
```

### Tnsnames.ora 파일을 설정

환경설정 셋팅을 완료하였으면 `$ vi /etc/oracle/tnsnames.ora ` 열어서 oracledb에 접속하기 위해 tns 값을 설정해줍니다.

PORT는 기본포트인 1521을 사용합니다.

```
XE=
(DESCRIPTION=
(ADDRESS_LIST=
(ADDRESS = (PROTOCOL = TCP)(HOST=[your-hostName])(PORT=1521))
)
(CONNECT_DATA=
(SERVICE_NAME=[your-serviceName])
)
)
```

`$ sqlplus username/password@dbhost(Ip):port/SID` 명령어로 oracledb에 접근되는지 확인합니다.

## Windows 환경 설정

[여기](https://www.oracle.com/database/technologies/instant-client/winx64-64-downloads.html) 에서 `12.1` 버전으로 `Base Package` 자료를 받으시면 됩니다.

`제어판 -> 시스템 -> 고급 시스템 설정 -> 고급 탭 -> 환경 변수`의 시스템 변수에 다음 추가합니다.

변수 이름: ORACLE_HOME / 변수 값: Instant Client 압축 해제한 경로 (예. D:\instantclient_12_1)

변수 이름: TNS_ADMIN / 변수 값: Instant Client 압축 해제한 경로\network (예. D:\instantclient_12_1\network)

변수 이름: NLS_LANG / 변수 값: KOREAN_KOREA.KO16MSWIN949

Path의 편집 들어가서 %ORACLE_HOME% 추가합니다.

다음은 레지스트리 편집기 실행(regedit)해 보겠습니다.

HKEY_LOCAL_HKEY_LOCAL_MACHINE\SOFTWARE 우클릭하여 `새로 만들기 -> 키` -> ORACLE 추가합니다.

HKEY_LOCAL_HKEY_LOCAL_MACHINE\SOFTWARE\ORACLE 우클릭하여 `새로 만들기 -> 문자열 값` -> TNS_ADMIN 추가하고 더블클릭 하여 값 데이터에 Instant Client 압축 해제한 경로\network (예. D:\instantclient_11_2\network) 입력후 `확인` 합니다.

HKEY_LOCAL_HKEY_LOCAL_MACHINE\SOFTWARE\ORACLE 우클릭하여 `새로 만들기 -> 문자열 값` -> NLS_LANG 추가하고 더블클릭 하여 값 데이터에 KOREAN_KOREA.KO16MSWIN949 입력후 `확인`

### Tnsnames.ora 파일을 설정

Instant Client 압축 해제한 경로\network (예. D:\instantclient_11_2\network)에 tnsnames.ora 파일 생성하고 접속하려는 시스템의 Oracle 접속정보 작성후 저장합니다.

```
XE=
(DESCRIPTION=
(ADDRESS_LIST=
(ADDRESS = (PROTOCOL = TCP)(HOST=[your-hostName])(PORT=1521))
)
(CONNECT_DATA=
(SERVICE_NAME=[your-serviceName])
)
)
```

## Embulk 설치

**Embulk v0.9 와 v0.10 버전은 Java 8, Java 9에서 동작되지 않습니다.**

### Linux & MacOS

```
curl --create-dirs -o ~/.embulk/bin/embulk -L "https://dl.embulk.org/embulk-latest.jar"
chmod +x ~/.embulk/bin/embulk
echo 'export PATH="$HOME/.embulk/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### Windows

```
PowerShell -Command "& {[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::TLS12; Invoke-WebRequest http://dl.embulk.org/embulk-latest.jar -OutFile embulk.bat}"
```

위와 관련된 자세한 내용은 [Embulk QuickStart](https://www.embulk.org/) 에서 참고 부탁드립니다.

## Embulk .yaml 파일 작성

embulk를 실행시키기 위한 관련 package들을 설치합니다.

```
$ embulk gem install embulk-input-oracle
$ embulk gem install embulk-output-bigquery
```

그리고, embulk를 실질적으로 수행하는데 필요한 `.yaml` 파일을 작성합니다.

`Hans-On`에서 필요한 자료들은 GCS에 업로드 해두었습니다.

`odjbc8.jar` : `gs://oracle-sqlplus/ojdbc8.jar`

```
in:
 type: oracle
 driver_path: [jar file location]
 host: [your-host-ip]
 port : "1521"
 user: h2
 password: "1234"
 database: XE
 query : | SELECT * FROM hr.departments

out:
 type: bigquery
 mode: replace
 auth_method : json_key
 json_keyfile : [ json_keyfile 경로를 적어줍니다. ]
 service_account_email : [bigQuery에 접근하기 위해 IAM>service account 에서 생성한 service_account 등록(단, bigQuery admin권한 설정이 등록 완료된 상태)]
 project: [현재 수행중인 프로젝트 위치]
 dataset: [your-bigQuery-dataSet]
 location : [your-location(region)] e.g)asia-northeast3
 table: [your-bigQuery-table]
 auto_create_table: true
 auto_create_dataset: true
 ignore_unknown_values: true
 allow_quoted_newlines: true
 open_timeout_sec : 7200
 send_timeout_sec : 7200
 read_timeout_sec : 7200
```

## 마이그레이션 수행 시 발생할 수 있는 이슈 모음

`sqlplus64: error while loading shared libraries: libsqlplus.so: cannot open shared object file: No such file or directory` :

```
$ sudo apt-get install libaio1 libaio-dev
```

`ORA-01882: timezone region not found ` :

```
sudo java -jar ojdbc8.jar configure -Doracle.jdbc.timezoneAsRegion=false
sudo timedatectl set-timezone Asia/Seoul
```

## 참고 문헌

[KimFactory-[Ubuntu 16.04] 우분투 오라클 클라이언트 설치 - sqlplus php5.6 php7.0 oracle client install](https://blog.kimsfactory.com/entry/Ubuntu-1604-%EC%9A%B0%EB%B6%84%ED%88%AC-%EC%98%A4%EB%9D%BC%ED%81%B4-%ED%81%B4%EB%9D%BC%EC%9D%B4%EC%96%B8%ED%8A%B8-%EC%84%A4%EC%B9%98-sqlplus-php56-php70-oracle-client-install)

[이정운-Embulk 이용해서 Oracle DB 에서 BigQuery 로 데이터 마이그레이션 삽질기](https://medium.com/@jwlee98/embulk-%EC%9D%B4%EC%9A%A9%ED%95%B4%EC%84%9C-oracle-db-%EC%97%90%EC%84%9C-bigquery-%EB%A1%9C-%EB%8D%B0%EC%9D%B4%ED%84%B0-%EB%A7%88%EC%9D%B4%EA%B7%B8%EB%A0%88%EC%9D%B4%EC%85%98-%EC%82%BD%EC%A7%88%EA%B8%B0-141dc1d62b73)

[Oracle Instant Client(11.2.0.4) 설치 및 설정(Windows 10 x64 기준)](https://testtube.tistory.com/entry/Oracle-Instant-Client11204-%EC%84%A4%EC%B9%98-%EB%B0%8F-%EC%84%A4%EC%A0%95Windows-10-x64-%EA%B8%B0%EC%A4%80)
