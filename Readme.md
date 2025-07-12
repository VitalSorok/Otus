#!/bin/bash
#set -x #Отладка
var=$1
p='       '
if [[ "$var" == "a" ]]; then
   echo -e "\e[95mPID Command_name State Tty_nr Utime(x0,01) Stime(x0,01)\e[0m" | sed "s/ /$p $p $p/2" | sed "s/ /$p/1"
   cat /proc/[0-9]*/stat | awk '{print $1,$2,$3,$7,$14,$15}' | column -t -o "$p" | awk '$4 != 0'
elif [[ "$var" == "x" ]]; then
  echo -e "\e[95mPID Command_name State Tty_nr Utime(x0,01) Stime(x0,01)\e[0m" | sed "s/ /$p $p $p/2" | sed "s/ /$p/1"
  cat /proc/[0-9]*/stat | awk '{print $1,$2,$3,$7,$14,$15}' | column -t -o "$p" | awk '$4 == 0'
elif [[ "$var" == "ax" ]]; then
  echo -e "\e[95mPID Command_name State Tty_nr Utime(x0,01) Stime(x0,01)\e[0m" | sed "s/ /$p $p $p/2" | sed "s/ /$p/1"
  cat /proc/[0-9]*/stat | awk '{print $1,$2,$3,$7,$14,$15}' | column -t -o "$p"
elif [[ "$var" == "" ]]; then
  echo -e "\e[95mPID Command_name State Tty_nr Utime(x0,01) Stime(x0,01)\e[0m" | sed "s/ /$p $p $p/2" | sed "s/ /$p/1"
  cat /proc/[0-9]*/stat | awk '{print $1,$2,$3,$7,$14,$15}' | column -t -o "$p"
elif [[ "$var" == "h" ]]; then
echo -e "
        Используемые опции к команде:
        \e[95ma\e[0m -  Показать процессы, связанные с терминалом, кроме главных системных процессов сеанса
        \e[95mx\e[0m -  Показать процессы, отсоединённые от терминала
        \e[95max\e[0m - ПОказать все процессы"
echo  -e "
        Расшифровка вывода:
        \e[95mPID\e[0m - The process ID

        \e[95mCommand_name\e[0m - The filename of the executable, in parentheses.This is visible whether or not the executable is swapped out

        \e[95mState One\e[0m - of the following characters, indicating process state:

                   \e[95mR\e[0m  Running

                   \e[95mS\e[0m  Sleeping in an interruptible wait

                   \e[95mD\e[0m  Waiting in uninterruptible disk sleep

                   \e[95mZ\e[0m  Zombie

                   \e[95mT\e[0m  Stopped (on a signal) or (before Linux 2.6.33) trace stopped

                   \e[95mt\e[0m  Tracing stop (Linux 2.6.33 onward)

                   \e[95mW\e[0m  Paging (only before Linux 2.6.0)

                   \e[95mX\e[0m  Dead (from Linux 2.6.0 onward)

                   \e[95mx\e[0m  Dead (Linux 2.6.33 to 3.13 only)

                   \e[95mK\e[0m  Wakekill (Linux 2.6.33 to 3.13 only)

                   \e[95mW\e[0m  Waking (Linux 2.6.33 to 3.13 only)

                   \e[95mP\e[0m  Parked (Linux 3.9 to 3.13 only)

                   \e[95mI\e[0m  Idle (Linux 4.14 onward)

        \e[95mTty_nr\e[0m - The controlling terminal of the process.(The minor device number is contained in the combination of bits 31 to 20 and 7 to 0; the major device number is in bits 15 to 8.)

        \e[95mUtime\e[0m -  Amount of time that this process has been scheduled in user mode, measured in clock ticks (divide by sysconf(_SC_CLK_TCK)).  This includes guest time, guest_time (time spent running a virtual CPU, see

        below), so that applications that are not aware of the guest time field do not lose that time from their calculations

        \e[95mStime\e[0m - Amount of time that this process has been scheduled in kernel mode, measured in clock ticks (divide by sysconf(_SC_CLK_TCK))."
else
echo "Ошибка ввода команды"
#set +x #Выключение отладки
fi

