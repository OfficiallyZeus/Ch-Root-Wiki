# Ch-Root-Wiki
(chroot) root #emerge-webrsync (chroot) root #emerge --sync
root #mkdir /mnt/mychroot
root #cd /mnt/mychroot
#!/sbin/openrc-run
 
depend() {
   need localmount
   need bootmisc
}
 
start() {
     ebegin "Mounting chroot directories"
     mount -o rbind /dev /mnt/mychroot/dev > /dev/null &
     mount -t proc none /mnt/mychroot/proc > /dev/null &
     mount -o bind /sys /mnt/mychroot/sys > /dev/null &
     mount -o bind /tmp /mnt/mychroot/tmp > /dev/null &
     eend $? "An error occurred while mounting chroot directories"
}
 
stop() {
     ebegin "Unmounting chroot directories"
     umount -f /mnt/mychroot/dev > /dev/null &
     umount -f /mnt/mychroot/proc > /dev/null &
     umount -f /mnt/mychroot/sys > /dev/null &
     umount -f /mnt/mychroot/tmp > /dev/null &
     eend $? "An error occurred while unmounting chroot directories"
}
(chroot) user $xhost +local:
#!/bin/bash
# 20150922  havp-chroot.sh
# GPL-3

PKGDIR="/var/cache/binpkgs"
CATEGORY="net-proxy"
PN="havp"
CHROOT="/usr/chroot/${PN}"
WORKD=`pwd`

# Cleaning chroot directory.
umount "${CHROOT}"/var/lib/clamav "${CHROOT}/var/log/${PN}" "${CHROOT}"/var/run "${CHROOT}"/var/tmp "${CHROOT}"/dev 1>/dev/null 2>&1
rm -rf "${CHROOT}"

# Make common directory and symlinks.
mkdir -p "${CHROOT}"/{dev,etc}
if [ -d /lib64 ]
    then
        mkdir -p "${CHROOT}"/{lib64,usr/lib64}
        cd "${CHROOT}" && ln -s lib64 lib
        cd "${CHROOT}/usr" && ln -s lib64 lib
    else
        mkdir -p "${CHROOT}"/{lib,usr/lib}
fi
mkdir -p /var/log/"${PN}" "/var/tmp/${PN}" "${CHROOT}"/var/lib/clamav "${CHROOT}/var/log/${PN}" "${CHROOT}"/var/tmp/ "${CHROOT}"/var/run
chown -R ${PN}:${PN} /var/log/${PN} /var/tmp/${PN} "${CHROOT}/var/log/${PN}"
chmod -R o-rwx /var/log/${PN} /var/tmp/${PN} "${CHROOT}/var/log/${PN}"
chmod -R g-rwx /var/log/${PN} "${CHROOT}"/var/log/${PN}

# Extract package.
tar -xjphf `ls ${PKGDIR}/${CATEGORY}/${PN}* | tail -n 1` -C "${CHROOT}"

# Copy necessary libraries.
cp -pRPd /lib/ld-* "${CHROOT}/lib"
cp `ldd "${CHROOT}/usr/sbin/${PN}" | awk '{print $3}' | grep "^/lib"` "${CHROOT}/lib"
cp `ldd "${CHROOT}/usr/sbin/${PN}" | awk '{print $3}' | grep "^/usr/lib"` "${CHROOT}/usr/lib"
cp -pRPd /usr/lib/libclam* "${CHROOT}/usr/lib"
cp `ldd "${CHROOT}/usr/lib/libclamav.so" | awk '{print $3}' | grep "^/lib"` "${CHROOT}/lib"
cp `ldd "${CHROOT}/usr/lib/libclamav.so" | awk '{print $3}' | grep "^/usr/lib"` "${CHROOT}/usr/lib"
cp `ldd "${CHROOT}/usr/lib/libclamunrar_iface.so" | awk '{print $3}' | grep "^/lib"` "${CHROOT}/lib"
cp `ldd "${CHROOT}/usr/lib/libclamunrar_iface.so" | awk '{print $3}' | grep "^/usr/lib"` "${CHROOT}/usr/lib"
cp `ldd "${CHROOT}/usr/lib/libclamunrar.so" | awk '{print $3}' | grep "^/lib"` "${CHROOT}/lib"

# Copy user information and necessary libraries for it.
cp -pRPd /lib/libnss* /lib/libnsl* /lib/libresolv* "${CHROOT}/lib"
cp /usr/lib/libnss3.so "${CHROOT}/usr/lib"
grep "^${PN}" "/etc/passwd" > "${CHROOT}/etc/passwd"
grep "^${PN}" "/etc/group" > "${CHROOT}/etc/group"

# fstab stuff.
if [[ `grep "HAVP chroot stuff/ and /usr file system when execute it! ." /etc/fstab` == '' ]]
    then
        cat >> /etc/fstab << EOF

# HAVP chroot stuff.
/var/lib/clamav         ${CHROOT}/var/lib/clamav        none    bind,nodev,noexec,nosuid,rw                                     0 0
/var/log/${PN}          ${CHROOT}/var/log/${PN}         none    bind,nodev,noexec,nosuid,rw                                     0 0
/var/tmp/${PN}          ${CHROOT}/var/tmp               none    bind,nodev,noexec,nosuid,rw                                     0 0
none                    ${CHROOT}/var/run               tmpfs   rw,nodev,noexec,nosuid,relatime,size=1024k,mode=755             0 0
none                    ${CHROOT}/dev                   tmpfs   rw,noexec,nosuid,relatime,size=1024k,nr_inodes=384443,mode=755  0 0
EOF
fi
mount -a

# Configuration.
cp -fpRPd /etc/${PN}/* ${CHROOT}/etc/${PN}/
cd ${WORKD}
cp -f ${PN} /etc/init.d/
cp -f ${PN} ${CHROOT}/etc/init.d/

exit 0

#!/sbin/openrc-run
# Copyright 1999-2015 Gentoo Foundation
# Distributed under the terms of the GNU General Public License v2

CHROOT=/usr/chroot/${SVCNAME}
PIDFILE=/var/run/${SVCNAME}/${SVCNAME}.pid

depend() {
#       need net
        use clamd \
#               squid apache2 bfilter mman junkbuster oops polipo privoxy tinyproxy tor wwwoffled
#       #havp could be used in conjuction with any parent proxies enumerated above
}

get_havp_opt() {
        eval HAVP_$1=`awk '/^[ \t]*'$1'[ \t]+/ { print $2; }' < /etc/${SVCNAME}/${SVCNAME}.config`
}

checkconfig() {
        if [ ! -f ${CHROOT}/etc/${SVCNAME}/${SVCNAME}.config ] ; then
                eerror "No ${CHROOT}/etc/${SVCNAME}/${SVCNAME}.config file exists!"
                return 1
        fi

        local HAVP_USER
        get_havp_opt USER
        if [ -n "${HAVP_USER}" ] && ! getent passwd ${HAVP_USER} > /dev/null ; then
                eerror "${HAVP_USER} user is missing!"
                return 1
        fi
        local HAVP_GROUP
        get_havp_opt GROUP
        if [ -n "${HAVP_GROUP}" ] && ! getent group ${HAVP_GROUP} > /dev/null ; then
                eerror "${HAVP_GROUP} group is missing!"
                return 1
        fi

        if [ ! -c ${CHROOT}/dev/null ] ; then
                mknod -m 666 ${CHROOT}/dev/null c 1 3
                mount -ro remount ${CHROOT}/dev
        fi

        checkpath --directory \
            --owner "${HAVP_USER:-havp}:${HAVP_GROUP:-havp}" --mode 0755 --directory `dirname ${CHROOT}${PIDFILE}`
        checkpath --directory \
            --owner "${HAVP_USER:-havp}:${HAVP_GROUP:-havp}" --mode 0700 ${CHROOT}/var/log/havp
        checkpath --directory \
            --owner "${HAVP_USER:-havp}:${HAVP_GROUP:-havp}" --mode 0750 "${CHROOT}/var/tmp/${SVCNAME}"
}

start() {
        checkconfig || return 1
/ and /usr file system when execute it! 
        ebegin "Starting chrooted HTTP AntiVirus Proxy"
        start-stop-daemon --start --pidfile=${CHROOT}${PIDFILE} --exec chroot ${CHROOT} /usr/sbin/${SVCNAME} > /dev/null
        eend $?
}

stop() {
        local HAVP_PIDFILE
        get_havp_opt PIDFILE

        ebegin "Stopping chrooted HTTP AntiVirus Proxy"
        start-stop-daemon --stop --pidfile=${CHROOT}${PIDFILE}
        
}
#!/bin/bash
# 20150922
# GPL-3

PKGDIR="/var/cache/binpkgs"
CATEGORY="net-proxy"
PN="dante"
CHROOT="/usr/chroot/${PN}"
USER="sockd"
WORKD=`pwd`

# Cleaning chroot directory.
umount "${CHROOT}"/var/lock/ "${CHROOT}"/var/log/${PN} "${CHROOT}"/var/run "${CHROOT}"/dev "${CHROOT}"/tmp 1>/dev/null 2>&1
rm -rf "${CHROOT}"

# Make common directory and symlinks.
mkdir -p "${CHROOT}"/{dev,etc}
if [ -d /lib64 ]
    then
        mkdir -p "${CHROOT}"/lib64
        cd "${CHROOT}" && ln -s lib64 lib
    else
        mkdir -p "${CHROOT}"/{lib}
fi
mkdir -p /var/log/${PN} /var/tmp/${PN} "${CHROOT}"/var/log/${PN} "${CHROOT}"/var/lock "${CHROOT}"/var/run "${CHROOT}"/tmp
chown -R ${USER}:root /var/log/${PN} /var/tmp/${PN} "${CHROOT}"/var/log/${PN}
chmod -R o-rwx /var/log/${PN} /var/tmp/${PN} "${CHROOT}"/var/log/${PN}

# Extract package.
tar -xjphf `ls ${PKGDIR}/${CATEGORY}/${PN}* | tail -n 1` -C "${CHROOT}"

# Copy necessary libraries.
cp -pRPd /lib/ld-* "${CHROOT}/lib"
cp `ldd "${CHROOT}/usr/sbin/${USER}" | awk '{print $3}' | grep "^/lib"` "${CHROOT}/lib"

# Copy user information and necessary libraries for it.
cp -pRPd /lib/libnss* /lib/libnsl* /lib/libresolv* "${CHROOT}/lib"
cp /usr/lib/libnss3.so "${CHROOT}/usr/lib"
grep "^${USER}" "/etc/passwd" > "${CHROOT}/etc/passwd"

# fstab stuff.
if [[ `grep "Dante chroot stuff." /etc/fstab` == '' ]]
    then
        cat >> /etc/fstab << EOF

# Dante chroot stuff.
/var/log/${PN}          ${CHROOT}/var/log/${PN}         none    bind,nodev,noexec,nosuid,rw                                     0 0
/var/tmp/${PN}          ${CHROOT}/tmp                   none    bind,nodev,noexec,nosuid,rw                                     0 0
none                    ${CHROOT}/var/lock              tmpfs   rw,nodev,noexec,nosuid,relatime,size=1024k,mode=755             0 0
none                    ${CHROOT}/var/run               tmpfs   rw,nodev,noexec,nosuid,relatime,size=1024k,mode=755             0 0
EOF
fi
mount -a

# Configuration.
cp -fp /etc/socks/* ${CHROOT}/etc/socks/
cp -fp /etc/conf.d/${PN}-sockd ${CHROOT}/etc/conf.d/
cd ${WORKD}
cp -f ${PN}-sockd /etc/init.d/
cp -f ${PN}-sockd ${CHROOT}/etc/init.d/
#!/sbin/openrc-run
# Copyright 1999-2015 Gentoo Foundation
# Distributed under the terms of the GNU General Public License v2
# $Id$

CHROOT=/usr/chroot/dante
SOCKD_OPT=""
[ "${SOCKD_FORKDEPTH:-1}" -gt 1 ] && SOCKD_OPT="${SOCKD_OPT} -N ${SOCKD_FORKDEPTH}"
[ "${SOCKD_DEBUG:-0}" -eq 1 ] && SOCKD_OPT="${SOCKD_OPT} -d"
[ "${SOCKD_DISABLE_KEEPALIVE:-0}" -eq 1 ] && SOCKD_OPT="${SOCKD_OPT} -n"
PIDFILE=/var/run/dante/sockd.pid
SOCKDIR=/var/lock/dante-sockd/

depend() {
        need net
}

checkconfig() {
        # first check that it exists
        if [ ! -f ${CHROOT}/etc/socks/sockd.conf ] ; then
                eerror "You need to setup /etc/socks/sockd.conf first"
                eerror "Examples are in /usr/share/doc/dante[version]/example"
                eerror "for more info, see: man sockd.conf"
                return 1
        fi

        /usr/sbin/sockd -V -f ${CHROOT}/etc/socks/sockd.conf >/tmp/dante-sockd.checkconf 2>&1
        if [ $? -ne 0 ]; then
                cat /tmp/dante-sockd.checkconf
                eerror "Something is wrong with your configuration file"
                eerror "for more info, see: man sockd.conf"
                return 1
        fi
        rm /tmp/dante-sockd.checkconf

        DAEMON_UID=`sed -e '/^[ \t]*user[.]notprivileged[ \t]*:/{s/.*:[ \t]*//;q};d' /etc/socks/sockd.conf`

        if [ -n "$DAEMON_UID" ]; then
                checkpath --quiet --mode 755 --owner "${DAEMON_UID}":root --directory `dirname ${CHROOT}${PIDFILE}`
                checkpath --quiet --mode 750 --owner "${DAEMON_UID}":root --directory "${CHROOT}${SOCKDIR}"
        else
                checkpath --quiet --mode 755 --directory `dirname ${CHROOT}${PIDFILE}`
                checkpath --quiet --mode 750 --directory "${CHROOT}${SOCKDIR}"
        fi

        return 0
}

start() {
        checkconfig || return 1
        ebegin "Starting chrooted dante sockd"
        start-stop-daemon --start --quiet \
                --background --pidfile ${CHROOT}$PIDFILE --make-pidfile --env TMPDIR=$SOCKDIR \
                --exec chroot ${CHROOT} /usr/sbin/sockd -- ${SOCKD_OPT} >/dev/null 2>&1
        eend $? "Failed to start sockd"
}

stop() {
        ebegin "Stopping chrooted dante sockd"
        start-stop-daemon --stop --quiet --pidfile ${CHROOT}$PIDFILE
}
#!/bin/bash
# 20150922
# GPL-3

PKGDIR="/var/cache/binpkgs"
CATEGORY="app-antivirus"
PN="clamav"
CHROOT="/usr/chroot/${PN}"
WORKD=`pwd`

# Cleaning chroot directory.
umount "${CHROOT}"/var/lib/${PN} "${CHROOT}"/var/log/${PN} "${CHROOT}"/var/run "${CHROOT}"/var/tmp "${CHROOT}"/dev 1>/dev/null 2>&1
rm -rf "${CHROOT}"

# Make common directory and symlinks.
mkdir -p "${CHROOT}"/{dev,etc}
if [ -d /lib64 ]
    then
        mkdir -p "${CHROOT}"/{lib64,usr/lib64}
        cd "${CHROOT}" && ln -s lib64 lib
        cd "${CHROOT}/usr" && ln -s lib64 lib
    else
        mkdir -p "${CHROOT}"/{lib,usr/lib}
fi
mkdir -p /var/log/${PN} "${CHROOT}"/var/log/${PN} "${CHROOT}"/var/run "${CHROOT}"/var/tmp
chown -R ${PN}:${PN} /var/log/${PN} "${CHROOT}"/var/log/${PN}
chmod -R o-rwx /var/log/${PN} "${CHROOT}"/var/log/${PN}

# Extract package.
tar -xjphf `ls ${PKGDIR}/${CATEGORY}/${PN}-0* | tail -n 1` -C "${CHROOT}"

# Copy necessary libraries.
cp -pRPd /lib/ld-* "${CHROOT}/lib"
cp `ldd "${CHROOT}/usr/bin/freshclam" | awk '{print $3}' | grep "^/lib"` "${CHROOT}/lib"
cp `ldd "${CHROOT}/usr/bin/freshclam" | awk '{print $3}' | grep "^/usr/lib"` "${CHROOT}/usr/lib"

# Copy user information and necessary libraries for it.
cp -pRPd /lib/libnss* /lib/libnsl* /lib/libresolv* "${CHROOT}/lib"
cp /usr/lib/libnss3.so "${CHROOT}/usr/lib"
grep "^${PN}" "/etc/passwd" > "${CHROOT}/etc/passwd"
grep "${PN}" "/etc/group" > "${CHROOT}/etc/group"

# fstab stuff.
if [[ `grep "Clamav chroot stuff." /etc/fstab` == '' ]]
    then
        cat >> /etc/fstab << EOF

# Clamav chroot stuff.
/var/lib/${PN}          ${CHROOT}/var/lib/${PN}         none    bind,nodev,noexec,nosuid,rw                                     0 0
/var/log/${PN}          ${CHROOT}/var/log/${PN}         none    bind,nodev,noexec,nosuid,rw                                     0 0
/var/tmp/havp           ${CHROOT}/var/tmp               none    bind,nodev,noexec,nosuid,user=${PN},ro                          0 0
none                    ${CHROOT}/var/run               tmpfs   rw,nodev,noexec,nosuid,relatime,size=1024k,mode=755             0 0
none                    ${CHROOT}/dev                   tmpfs   rw,noexec,nosuid,relatime,size=1024k,nr_inodes=384443,mode=755  0 0
EOF
fi
mount -a

# Configuration.
cp -fp /etc/freshclam.conf ${CHROOT}/etc/
cp -fp /etc/clamd.conf ${CHROOT}/etc/
cd ${WORKD}
cp -f clamd /etc/init.d/
cp -f clamd ${CHROOT}/etc/init.d/
#!/sbin/openrc-run
# Copyright 1999-2015 Gentoo Foundation
# Distributed under the terms of the GNU General Public License v2

CHROOT="/usr/chroot/clamav"
PIDCLAMD="/var/run/clamav/clamd.pid"
PIDFRESH="/var/run/clamav/freshclam.pid"
daemon_clamd="/usr/sbin/clamd"
daemon_freshclam="/usr/bin/freshclam"
daemon_milter="/usr/sbin/clamav-milter"

extra_commands="logfix"

depend() {
        use net
        provide antivirus
}

get_config() {
        clamconf | sed 's/["=]//g' | \
        awk "{
        if(\$0==\"Config file: $1.conf\") S=1
        if(S==1&&\$0==\"\") {
                print \"$3\"
                exit
        }
        if(S==1&&\$1~\"^$2\$\") {
                print \$2!=\"disabled\"?\$2:\"$3\"
                exit
        }
        }"
}

checkconfig() {
        checkpath --directory \
                --owner "${CLAMAV_USER:-clamav}:${CLAMAV_GROUP:-clamav}" \
                --mode 0755 --directory `dirname ${CHROOT}${PIDCLAMD}`
        if [ ! -c ${CHROOT}/dev/null ] ; then
                mknod -m 666 ${CHROOT}/dev/null c 1 3
                mount -ro remount ${CHROOT}/dev
        fi
}

start() {
        # populate variables and fix log file permissions
        logfix

        if [ "${START_CLAMD}" = "yes" ]; then
                checkconfig || return 1
                if [ -S "${CHROOT}${clamd_socket}" ]; then
                        rm -f ${CHROOT}${clamd_socket}
                fi
                ebegin "Starting chrooted clamd"
                start-stop-daemon --start --quiet \
                        --nicelevel ${CLAMD_NICELEVEL:-0} \
                        --ionice ${IONICE_LEVEL:-0} \
                        --pidfile "${CHROOT}${PIDCLAMD}" \
                        --exec chroot ${CHROOT} ${daemon_clamd}
                eend $? "Failed to start clamd"
        fi

        if [ "${START_FRESHCLAM}" = "yes" ]; then
                checkconfig || return 1
                ebegin "Starting chrooted freshclam"
                start-stop-daemon --start --quiet \
                        --nicelevel ${FRESHCLAM_NICELEVEL:-0} \
                        --ionice ${IONICE_LEVEL:-0} \
                        --pidfile "${CHROOT}${PIDFRESH}" \
                        --exec chroot ${CHROOT} ${daemon_freshclam} -- -d --pid "${PIDFRESH}"
                retcode=$?
                if [ ${retcode} = 1 ]; then
                        eend 0
                        einfo "Virus databases are already up to date."
                else
                        eend ${retcode} "Failed to start freshclam"
                fi
        fi

        if [ "${START_MILTER}" = "yes" ]; then
                if [ -z "${MILTER_CONF_FILE}" ]; then
                        MILTER_CONF_FILE="/etc/clamav-milter.conf"
                fi

                ebegin "Starting clamav-milter"
                start-stop-daemon --start --quiet \
                        --nicelevel ${MILTER_NICELEVEL:-0} \
                        --ionice ${IONICE_LEVEL:-0} \
                        --exec ${daemon_milter} -- -c ${MILTER_CONF_FILE}
                eend $? "Failed to start clamav-milter"
        fi
}

stop() {
        if [ "${START_CLAMD}" = "yes" ]; then
                ebegin "Stopping chrooted clamd"
                start-stop-daemon --stop --pidfile "${CHROOT}${PIDCLAMD}" --quiet --name clamd
                eend $? "Failed to stop clamd"
        fi
        if [ "${START_FRESHCLAM}" = "yes" ]; then
                ebegin "Stopping chrooted freshclam"
                start-stop-daemon --stop --quiet \
                --pidfile "${CHROOT}${PIDFRESH}" \
                --name freshclam \
                --exec chroot ${CHROOT} ${daemon_freshclam} -- -K --pid "${PIDFRESH}"
                eend $? "Failed to stop freshclam"
        fi
        if [ "${START_MILTER}" = "yes" ]; then
                ebegin "Stopping clamav-milter"
                start-stop-daemon --stop --quiet --name clamav-milter
                eend $? "Failed to stop clamav-milter"
        fi
}

logfix() {
        clamd_socket=$(get_config clamd LocalSocket /run/clamav/clamd.sock)
        clamd_user=$(get_config clamd User clamav)
        freshclam_user=$(get_config freshclam DatabaseOwner clamav)

        if [ "${START_CLAMD}" = "yes" ]; then
                # fix clamd log permissions
                # (might be clobbered by logrotate or something)
                local logfile=$(get_config clamd LogFile)
                if [ -n "${CHROOT}${logfile}" ]; then
                        checkpath --quiet \
                                --owner "${clamd_user}":"${clamd_user}" \
                                --mode 640 \
                                --file ${CHROOT}${logfile}
                fi
        fi

        if [ "${START_FRESHCLAM}" = "yes" ]; then
                # fix freshclam log permissions
                # (might be clobbered by logrotate or something)
                local logfile=$(get_config freshclam UpdateLogFile)
                if [ -n "${CHROOT}${logfile}" ]; then
                        checkpath --quiet \
                                --owner "${freshclam_user}":"${freshclam_user}" \
                                --mode 640 \
                                --file ${CHROOT}${logfile}
                fi
        fi
}
                  +-------------------------------------------------------------+
                  | Chrooted sockd or torsocks <-> Other Internet applications  |
                  |      ^                                                      |
                  |      |                                                      |
                  |                                Chrooted          HTTP*      |
 +----------+     |  Chrooted  <->  Chrooted  <->    HAVP    <->    Internet    |
 | Internet | <-> | <-> Tor         Privoxy            +          applications  |
 +----------+     |                    ^           libClamAV                    |
                  |                    |                                        |
                  |                                                             |
                  |                 Chrooted                                    |
                  |                 FreshClam                                   |
                  +-------------------------------------------------------------+
 Security options  --->
    Grsecurity  --->
        [*] Grsecurity
            Customize Configuration  --->
                Filesystem Protections  --->
                    [*] Chroot jail restrictions
                    [*]   Deny mounts
                    [*]   Deny double-chroots
                    [*]   Deny pivot_root in chroot
                    [*]   Enforce chdir("/") on all chroots
                    [*]   Deny (f)chmod +s
                    [*]   Deny fchdir and fhandle out of chroot
                    [*]   Deny mknod
                    [*]   Deny shmat() out of chroot
                    [*]   Deny access to abstract AF_UNIX sockets out of chroot
                    [*]   Protect outside processes
                    [*]   Restrict priority changes
                    [*]   Deny sysctl writes
                    [*]   Deny bad renames
                    [*]   Capability restrictions                 




(chroot) root #useradd -m user
(chroot) root #su -l user
