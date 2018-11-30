---
layout: post
title: "Yocto寫一個systemd service recipe"
author:     Kevin Lee
tags: 		Yocto
subtitle:   學習Systemd Service Recipe如何去寫
category:  project1
visualworkflow: true
---
本文是從[Yocto寫一個Shared library的Recipe]({% post_url 2018-11-29-write-a-recipe-of-shared-library-for-yocto %})延伸過來的

要寫一個systemd service可以使用的Recipes
在Recipe內，首先

```
SUMMARY = "This package shall provide helper functionality between PAM libraries and the libnss libraries. PAM libraries can store the username and the privilege in pam helper and the libnss libraries can request for that information from pam helper. The pam helper also assigns a unique UID to the username stored. "
SECTION = "apps"
LICENSE = "CLOSED
```

因為Daemon就是在背景內執行的應用程式，所以SECTION要改成"apps"

再來編譯連結的相依問題也要處理一下

```
DEPENDS = "libpamhelper"
RDEPENDS_${PN} = "libpamhelper"
```

定義使用者變數與環境變數
其中
**INHIBIT_PACKAGE_DEBUG_SPLIT**和**PACKAGES**在上一篇已經解釋過用途了
另外使用者定義的變數也是因人而變的。

```
APP_NAME = "pam_helperd"
INHIBIT_PACKAGE_DEBUG_SPLIT  = "1"
PACKAGES = "${PN}"
localdir = "/usr/local"
libdir = "${localdir}/lib"
bindir = "${localdir}/bin"
PKG_MAJOR   = "6"
PKG_MINOR   = "0"
PKG_AUX     = "0"
```

以下就是要當成systemd service的定義

```
inherit systemd
SYSTEMD_SERVICE_${PN} = "pamhelperd.service"
SRC_URI = "file://pamhelperd.service"
```

每個被當作Daemon的應用程式，其實都有一個service的檔案再控制Daemon的行為

以範例pamhelperd.service的內容大致如下

```
[Unit]
Description="PAM Helperd Service"

[Service]
Type=simple
ExecStart=/usr/bin/env /etc/pamhelperd.d/pamhelperd.sh start
SyslogIdentifier=PAM-Helperd
ExecStop=/usr/bin/env /etc/pamhelperd.d/pamhelperd.sh stop
RemainAfterExit=yes

[Install]
WantedBy={SYSTEMD_DEFAULT_TARGET}
```

這些都是告訴systemd要如何去控制這個daemon

接下的編譯與安裝要把應用程式和service放到rootfs對應的位置
大致如下

```
.
|-- etc
|   |-- pamhelperd.d
|   |   `-- pamhelperd.sh
|   `-- systemd
|       `-- system
|           `-- obmc-standby.target.wants
|               `-- pamhelperd.service -> /lib/systemd/system/pamhelperd.service
|-- lib
|   `-- systemd
|       `-- system
|           `-- pamhelperd.service
`-- usr
    `-- local
        `-- bin
            `-- pam_helperd
```

所以Recipe的do_install就要install到對應的位置

```
do_configure_prepend() {
	export TOOLDIR=${STAGING_INCDIR}
	export SPXLIB=${STAGING_LIBDIR}
	export TARGETDIR=${STAGING_DIR_TARGET}
	export IMAGE_TREE=${STAGING_DIR_TARGET}
	export PKG_MAJOR=${PKG_MAJOR}
	export PKG_MINOR=${PKG_MINOR}
	export PKG_AUX=${PKG_AUX}
}

do_compile_prepend() {
	export TOOLDIR=${STAGING_INCDIR}
	export SPXLIB=${STAGING_LIBDIR}
	export TARGETDIR=${STAGING_DIR_TARGET}
	export IMAGE_TREE=${STAGING_DIR_TARGET}
	export PKG_MAJOR=${PKG_MAJOR}
	export PKG_MINOR=${PKG_MINOR}
	export PKG_AUX=${PKG_AUX}
}

do_install () {
	install -m 0755 -d ${D}${localdir}
	install -m 0755 -d ${D}${bindir}
	install -d ${D}${sysconfdir}/pamhelperd.d
	
install -d ${D}${systemd_system_unitdir}
install -m 0644 ${WORKDIR}/pamhelperd.service ${D}${systemd_system_unitdir}

install -d ${D}/etc/systemd/system/obmc-standby.target.wants
ln -s ${systemd_system_unitdir}/pamhelperd.service ${D}/etc/systemd/system/obmc-standby.target.wants/pamhelperd.service 

cd ${S}
install -m 0755 pamhelperd.sh ${D}${sysconfdir}/pamhelperd.d/
install -m 0755 ${APP_NAME} ${D}${bindir}
}

FILES_${PN}-dev = ""
FILES_${PN} = "${bindir}/* ${sysconfdir}/* ${systemd_system_unitdir}/*"
```

