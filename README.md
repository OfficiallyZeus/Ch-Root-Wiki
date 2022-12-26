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
chmod -R o-rwx /var/log/${PN} /var/tmp/${PN} "${CHROOT}"/var/log/${
