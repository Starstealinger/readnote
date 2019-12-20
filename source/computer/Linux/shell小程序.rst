.. contents::
.. _header-n47:

shell小程序
===========

.. _header-n31:

显示系统信息
------------

``show_sys_info.sh``

.. code:: shell

   #!/bin/bash
   #show system information
   PS3="Your choice is: "
   function os_check() {
   	if [ -e /etc/redhat_release ]; then
   		REDHAT=`cat /etc/redhat_release |awk '{print $1}'`
   	else
   		DEBIAN=`cat /etc/issue |awk '{print $1}'`
   	fi
   	if [ $REDHAT == "CentOS" -o $REDHAT == "Red" ];then
   		P_M=yum
   	elif [ $DEBIAN == "Ubuntu" -o $DEBIAN == "ubuntu" ];then
   		P_M=apt-get
   	else
   		echo "Operating System doesn't support."
   		exit
   	fi
   }

   if [ "$USER" != "root" ];then
   	echo "Please use root account operation."
   	exit 1
   fi

   if ! which vmstat &>/dev/null;then
   	echo "vmstat command not found, now the install."
   	sleep 1
   	os_check
   	$P_M install procps -y
   	echo "----------------------------------------------------------"
   fi

   which iostat &>/dev/null
   if [ $? -ne 0 ];then
   	echo "iostat command not found, now the install."
   	sleep 1
   	os_check
   	$P_M install sysstat -y
   fi

   function cpuLoad() {
   	echo "---------------------------------------"
   	for i in {1..3}
   	do
   		echo -e "\e[1;32m  参考值#${i}\e[0m"
   		UTIL=`vmstat |awk '{if(NR==3){print 100-$(NF-2)"%"}}'`   #使用率
   		USER=`vmstat |awk '{if(NR==3){print $(NF-4)"%"}}'`	#用户占用率
   		SYS=`vmstat |awk '{if(NR==3){print $(NF-3)"%"}}'`	#系统占用率
   		IOWAIT=`vmstat |awk '{if(NR==3){print $(NF-1)"%"}}'`	#IO等待
   		echo "UTIL: $UTIL"
   		echo "USER: $USER"
   		echo "SYS: $SYS"
   		echo "IOWAIT: $IOWAIT"
   		sleep 1
   	done
   	echo "---------------------------------------"
   }

   function diskLoad() {
   	#硬盘I/O负载	
   	echo "---------------------------------------"
   	for i in {1..3}
   	do
   		echo -e "\033[32m  参考值${i}\033[0m"
   		UTIL=`iostat -x -k |awk '/^[v|s]d/{OFS=": ";print $1,$NF"%"}'`	#使用率
   		READ=`iostat -x -k |awk '/^[v|s]d/{OFS=": ";print $1,$6"kB/s"}'`	#读
   		WRITE=`iostat -x -k |awk '/^[v|s]d/{OFS=": ";print $1,$7"kB/s"}'`	#写
   		IOWAIT=`vmstat |awk '{if(NR==3) print $(NF-1)"%"}'`
   		echo -e "Util:\n$UTIL"
   		echo -e "Read:\n$READ"
   		echo -e "Write:\n$WRITE"
   		echo "I/O Wait: $IOWAIT"
   		sleep 1
   	done
   	echo "---------------------------------------"
   }

   function diskUse() {
   	#硬盘利用率
   	DISK_LOG=/tmp/disk_use.tmp
   	DISK_TOTAL=`fdisk -l |awk '/^Disk.*bytes/ && /\/dev/ && !/docker/ {printf $2" "; printf "%d",$3;print $4}'`
   	DISK_RATE=`df -h |awk '/^\/dev/{print int($5)}'`
   	for i in $DISK_RATE
   	do
   		if [ $i -gt 90 ];then
   			PART=`df -h |awk '{if(int($5)=='''$i''') print $6}'`
   			echo "$PART = ${i}%" >> $DISK_LOG
   		fi
   	done
   	echo "---------------------------------------"
   	echo -e "Disk total:\n${DISK_TOTAL}"
   	if [ -f $DISK_LOG ];then
   		echo "---------------------------------------"
   		cat $DISK_LOG
   		echo "---------------------------------------"
   		rm -f $DISK_LOG
   	else
   		echo "---------------------------------------"
   		echo "Disk use rate no than 90% of the partition."
   		echo "---------------------------------------"
   	fi
   }

   function diskInode() {
   	INODE_LOG=/tmp/inode_use.tmp
   	INODE_USE=`df -i |awk '/^\/dev/{print int($5)}'`
   	for i in $INODE_USE
   	do
   		if [ $i -ge 90 ];then
   			PART=`df -i |awk '{if(int($5)=='''$i''') print $6}'`
   			echo "$PART = %{i}%" >> $INODE_LOG
   		fi
   	done
   	if [ -f $INODE_LOG ];then
   		echo "---------------------------------------"
   		cat $INODE_LOG
   		rm -f $INODE_LOG
   	else
   		echo "---------------------------------------"
   		echo "Inode use rate no than 90% of the partition."
   		echo "---------------------------------------"
   	fi
   }

   function memUse() {
   	#内存利用率
   	echo "---------------------------------------"
   	MEM_TOTAL=`free -m |awk '{if(NR==2)printf "%.1f",$2/1024} END{print "G"}'`
   	USE=`free -m |awk '{if(NR==2)printf "%.1f",$3/1024} END{print "G"}'`
   	FREE=`free -m |awk '{if(NR==2)printf "%.1f",$4/1024} END{print "G"}'`
   	CACHE=`free -m |awk '{if(NR==2)printf "%.1f",$6/1024} END{print "G"}'`
   	echo "Total: $MEM_TOTAL"
   	echo "Used: $USE"
   	echo "Free: $FREE"
   	echo "Cache: $CACHE"
   	echo "---------------------------------------"
   }

   function tcpStatus() {
   	#网络连接状态
   	echo "---------------------------------------"
   	COUNT=`netstat -ant |awk '/tcp/{state[$(NF)]++} END{for (i in state) print i,state[i]}'`
   	echo -e "TCP connection state: \n$COUNT"
   	echo "---------------------------------------"
   }

   function cpuTop10() {
   	echo "---------------------------------------"
   	CPU_LOG=/tmp/cpu_top.tmp
   	i=1
   	while [ $i -le 3 ];
   	do
   		ps aux |awk '{if($3>0.1){{printf "PID: "$2" CPU: "$3"% --> "}for(j=11;j<=NF;j++)if(j==NF)print " " $j;else printf " "$j}}' |sort -k4 -rn |head > $CPU_LOG
   		if [[ -n `cat $CPU_LOG` ]];then
   			echo -e "\033[32m  参考值${i}\033[0m"
   			cat $CPU_LOG
   			>$CPU_LOG
   		else
   			echo "No process using CPU."
   			break
   		fi
   		let i++
   		sleep 1
   	done
   	echo "---------------------------------------"
   }

   function memTop10() {
   	echo "---------------------------------------"
   	MEM_LOG=/tmp/mem_top.tmp
   	i=1
   	while [ $i -le 3 ];
   	do
   		ps aux |awk '{if($4>0.1)print "PID: "$2" Memory: "$4"% --> " $11}' |sort -k4 -rn |head > $MEM_LOG
   		if [[ -n `cat $MEM_LOG` ]];then
   			echo -e "\033[32m  参考值${i}\033[0m"
   			cat $MEM_LOG
   			> $MEM_LOG
   		else
   			echo "No process using Memory."
   			break
   		fi
   		let i++
   		sleep 1
   	done
   	echo "---------------------------------------"
   }

   function Traffic() {
   	#查看网络流量
   	while true
   	do
   		read -p "Please enter the network name: " eth
   		if [ `ifconfig |grep -c "\<$eth\>"` -eq 1 ];then
   			break
   		else
   			echo "Input error,please input again."
   		fi
   	done
   	echo "---------------------------------------"
   	echo -e " In ----- Out"
   	i=1
   	while [ $i -le 3 ];do
   		OLD_RX=`ifconfig $eth |awk '{if(NR==5)print $5}'`
   		OLD_TX=`ifconfig $eth |awk '{if(NR==7)print $5}'`
   		sleep 1
   		NEW_RX=`ifconfig $eth |awk '{if(NR==5)print $5}'`
   		NEW_TX=`ifconfig $eth |awk '{if(NR==7)print $5}'`
   		
   		RX=`awk 'BEGIN{printf "%.1f\n",'$((${NEW_RX}-${OLD_RX}))'/1024/128}'`
   		TX=`awk 'BEGIN{printf "%.1f\n",'$((${NEW_TX}-${OLD_TX}))'/1024/128}'`
   		echo "${RX}MB/s  ${TX}MB/s"
   		let i++
   		sleep 1
   	done
   	echo "---------------------------------------"
   }

   while true
   do
   	select input in cpu_load disk_load disk_use disk_inode mem_use tcp_status cpu_top10 mem_top10 traffic quit
   	do
   		case $input in
   			cpu_load)
   			#CPU利用率与负载
   			cpuLoad
   			break
   			;;
   			disk_load)
   			diskLoad
   			break
   			;;
   			disk_use)
   			diskUse
   			break
   			;;
   			disk_inode)
   			diskInode
   			break
   			;;
   			mem_use)
   			memUse
   			break
   			;;
   			tcp_status)
   			tcpStatus
   			break
   			;;
   			cpu_top10)
   			cpuTop10
   			break
   			;;
   			mem_top10)
   			memTop10
   			break
   			;;
   			traffic)
   			Traffic
   			break
   			;;
   			quit)
   			exit
   			;;
   			*)
   			echo "error."
   			break
   		esac
   	done
   done

.. _header-n34:

ping3次判断主机是否存活
-----------------------

``ping_count3.sh``

.. code:: shell

   #!/bin/bash

   for ip in 192.168.160.{2..254}; do
   	for i in {1..3}; do
   		ping -c1 -W1 $ip &>/dev/null
   		if [ $? -eq 0 ];then
   			echo -e "\e[1;32mping $ip successfully.\e[0m"
   			break
   		else
   			echo "ping $ip is failure :${i}"
   			fail_count[$i]=$ip
   		fi
   	done
   	if [ ${#fail_count[@]} -eq 3 ];then
   		echo -e "\e[1;31mping $ip failure.\e[0m"
   		unset fail_count[*]
   	fi
   done

``ping_count3_01.sh``

.. code:: shell

   #!/bin/bash

   while read ip; do
   	count=0
   	for i in {1..3}; do
   		ping -c1 -W1 $ip &>/dev/null
   		if [ $? -eq 0 ];then
   			echo -e "\e[1;32mping $ip successfully.\e[0m"
   			break
   		else
   			echo "ping $ip is failure :${i}"
   			let count++
   		fi
   	done
   	if [ $count -eq 3 ];then
   		echo -e "\e[1;31mping $ip failure.\e[0m"
   	fi
   done <ip.txt

``ping_count3_02.sh``

.. code:: shell

   #!/bin/bash

   function ping_success() {
   	ping -c1 -W1 $ip &>/dev/null
   	if [ $? -eq 0 ];then
   		echo -e "\e[1;32mping $ip successfully.\e[0m"
   		continue
   	fi
   }
   while read ip;do
   	ping_success
   	ping_success
   	ping_success
   	echo -e "\e[1;31mping $ip failure.\e[0m"
   done

.. _header-n41:

select
------

.. code:: shell

   #!/bin/bash
   PS3="Your choice is(5 for exit): "
   select choice in disk_partition filesystem cpu_load mem_util quit
   do
           case $choice in
                   "disk_partition")
                           fdisk -l
                           ;;
                   "filesystem")
                           df -h
                           ;;
                   "cpu_load")
                           uptime
                           ;;
                   "mem_util")
                           free -m
                           ;;
                   "quit")
                           break
                           ;;
                   *)
                           echo "error"
                           exit
           esac
   done
