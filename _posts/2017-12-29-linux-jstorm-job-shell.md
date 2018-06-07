---
layout: post
categories: Linux Shell
title: Jstorm 多任务管理脚本
date: 2017-08-03 15:39:53 +0800
description: Jstorm 多任务管理脚本
keywords: jstorm shell
---


>由于之前一直都是手动的执行命令去提交任务和启动停止任务,但是由于计算任务的逐个增加，每次都要敲一大串的命令或者去执行单个任务脚本是一件很痛苦的事情......
为了方便对jstorm任务日常管理操作，将同一业务下的多个任务进行整理写入一个脚本来进行统一操作


``` sh
#!/bin/bash
## Jstorm 多任务管理脚本
##

USAGE="Usage: $(basename $0) [-hms] [SRV] [TYPE] [ACTION] 

SRV (SRV Option) is one of:

    web         \t web
    app         \t app
    all              \t web and app

Where [TYPE] is on of:
    flow        \t flow
    status      \t status
    all         \t all

Where [ACTION] is on of:
    start          \t start job
    stop          \t stop  job

-h    \t display this help.
-d    \t date   e.g: YYYYmmdd
-t    \t times  e.g: HHMM"

## color
red="\033[31m"
green="\033[32m"
yellow="\033[33m"
plan="\033[0m"


## webFlow
webFlow(){
      if [ $KEY == 0 ];then

      else

      fi
}

appFlow() {
      if [ $KEY == 0 ];then

      else

      fi
}

webStatus() {
     if [ $KEY == 0 ];then

     elif [ $KEY == 1 ];then
        
     fi
}

appStatus() {
     if [ $KEY == 0 ];then
        echo ""
     elif [ $KEY == 1 ];then
        echo ""
     fi
}

RUN(){
if [ $APP == "web" ];then
           if [ $BW == "flow" ];then
              webFlow
           elif [ $BW == "status" ];then
              webStatus
           elif [ $BW == "all" ];then
              webFlow
              webStatus
           fi
        elif [ $APP == "app" ];then
           if [ $BW == "flow" ];then
              appFlow
           elif [ $BW == "status" ];then
              appStatus
           elif [ $BW == "all" ];then
              appFlow
              appStatus
           fi
        elif [ $APP == "all" ];then
           if [ $BW == "flow" ];then
              webFlow
              appFlow
           elif [ $BW == "status" ];then
              webStatus
              appStatus
           elif [ $BW == "all" ];then
              webFlow
              appFlow
              webStatus
              appStatus
           fi
        fi
}

main() {  
while getopts "hm:t:" opt
do
    case "$opt" in
       h)
        echo -e "${USAGE}"
        exit 0
        ;;
       m)
        Date=$OPTARG
        ;;
       t)
         Time=$OPTARG
         ;; 
       *)
        echo -e "${USAGE}"
        exit 0
        ;;
     esac
done

shift $((${OPTIND} - 1))

SRV=$1
case "${SRV}" in
     web) APP=web ;;
     app) APP=app ;;
     all) APP=all ;;
     *)
        echo -e "${red}Error: no SRV specified $plan" >&2
        echo -e "${USAGE}"
        exit 1
        ;;
esac
shift

TYPE=$1
case $TYPE in
    flow)
       BW="flow" ;;
    status)
       BW="status" ;;
    all)
       BW="all" ;;
    *)
     echo -e "${red}Error: no TYPE specified $plan" >&2
     exit 1
     ;;
esac
shift

ACTION=$1
case $ACTION in 
     start)
        KEY=0 
        RUN
        ;;
     stop)
        KEY=1
        RUN
        ;;
     *)
       echo -e "${USAGE}"
       exit 1
       ;;
esac
}

main "$@"

```

