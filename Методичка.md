## 1. Подключение к СКЦ

Я не использовал какие-либо зумерские программы, а подключался по чистому SSH. 

Установка SSH-agent на Windows и настройка ключей: [Заметка по SSH](https://kb.tishenko.dev/servers/ssh/)

Далее нам нужно подключиться к СКЦ с помощью нашего приватного ключа, который нам выдали: 
```bash 
ssh -v [логин]@[асдрес] -i [путь до приватного ключа]
```
Например, так:
```bash 
ssh -v tm3u33@login1.hpc.spbstu.ru -i ~/.ssh/id_rsa  
```

Так мы сможем попасть в нашу папку на СКЦ. 
Далее необходимо перебросить некоторые файлы. Для шифровки файлов используется публичный ключ, который Вам выдали. 

Для копирования файлов на СКЦ используется следующая команда:
```bash 
scp [путь к файлу] [имя пользователя]@[имя сервера/ip-адрес]:[путь к файлу] 
```
Например, так:
```bash 
scp -r "E:\kernel.cu" tm3u33@login1.hpc.spbstu.ru:home/kernel.cu
```
Произошло копирование файла kernel.cu в папку home на СКЦ.
Теперь у нас есть исполняемый файл для работы на СКЦ.

## 2. Добавление работы в очередь СКЦ

На СКЦ используется Slurm Workload Manager. Он позволяет контролировать работу с узлами, создавать отдельные задачи, 
а также создавать очередность задач. 

Первым делом нам нужно создать скрипт, который будет создавать задачу, а также координировать действия узла.
Для этого создаем файл с расширение .script со следующим содержанием:

```bash 
#!/bin/bash
#SBATCH --nodes=1 
#SBATCH -p tornado-k40 
#SBATCH -t 00-00:20:00
#SBATCH -J cudaLab
#SBATCH -o cudaLab-%j.out
#SBATCH -e cudaLab-%j.err

module load compiler/gcc/11
module load nvidia/cuda/11.6u2
nvcc -arch=sm_35 --run kernel.cu
```
Где --nodes=1 - количество узлов, которое нам нужно 

-p tornado-k40 - тип узлов(Нам нужен именно такой) 

-t 00-00:20:00 - максимальное время исполнения нашей задачи(Если время выполнения задачи превысит этот показатель, 
то задача перестанет выполняться) 

-J cudaLab - название задачи 

-o cudaLab-%j.out - название файла, куда будет выведены выходные данные программы. 

-e cudaLab-%j.err - название файла, куда будет выведены ошибки программы. 

Также через флаг -i можно записать название файла с входными данными для программы(файл должен иметь расширение .in).

Далее идут команды для испорта модулей, компиляции и запуска кода. \
ИХ НЕ МЕНЯТЬ!!! Мне уже по шапке админ СКЦ настучал.

После создания такого скрипта его необходимо запустить. Делаем так:
```bash 
sbatch my.script
```
sbatch - запускает скрипт, указанный в файле.

## 3. Контроль выполнения работы 

У каждой работы генерируется свой ID. Чтобы отслеживать статус задачи, его нужно не потерять.

Чтобы просмотреть статус работы, можно использовать одну из следующих команд:
```bash 
scontrol show job
scontrol show jobid 4276916
```
Первая команда выводит информацию о всех Ваших задачах, вторая - только о задаче с определенным id.

Мы получим следующую информацию:
```bash 
JobId=4276916 JobName=cudaLab
   UserId=tm3u33(50293) GroupId=ipmmstudy1(50157) MCS_label=N/A
   Priority=842 Nice=0 Account=ipmmstudy1 QOS=normal WCKey=*default:ipmmstudy1
   JobState=RUNNING Reason=None Dependency=(null)
   Requeue=0 Restarts=0 BatchFlag=1 Reboot=0 ExitCode=0:0
   RunTime=00:00:12 TimeLimit=00:20:00 TimeMin=N/A
   SubmitTime=2025-03-19T20:51:10 EligibleTime=2025-03-19T20:51:10
   AccrueTime=2025-03-19T20:51:10
   StartTime=2025-03-19T20:51:11 EndTime=2025-03-19T21:11:11 Deadline=N/A
   SuspendTime=None SecsPreSuspend=0 LastSchedEval=2025-03-19T20:51:11
   Partition=tornado-k40 AllocNode:Sid=login1:79912
   ReqNodeList=(null) ExcNodeList=(null)
   NodeList=n02p003
   BatchHost=n02p003
   NumNodes=1 NumCPUs=56 NumTasks=1 CPUs/Task=1 ReqB:S:C:T=0:0:*:*
   TRES=cpu=56,node=1,billing=56
   Socks/Node=* NtasksPerN:B:S:C=0:0:*:* CoreSpec=*
   MinCPUsNode=1 MinMemoryNode=0 MinTmpDiskNode=0
   Features=(null) DelayBoot=00:00:00
   OverSubscribe=NO Contiguous=0 Licenses=(null) Network=(null)
   Command=/home/ipmmstudy1/tm3u33/home/my.script
   WorkDir=/home/ipmmstudy1/tm3u33/home
   StdErr=/home/ipmmstudy1/tm3u33/home/cudaLab-4276916.err
   StdIn=/dev/null
   StdOut=/home/ipmmstudy1/tm3u33/home/cudaLab-4276916.out
   Power=
```

Нам интересны следующие поля: 

JobState - показывает состояние работы. За все время я наблюдал такие типы: 
* PENDING - оставлен в очередь и ожидает запуска
* FAILED - 	выполнение завершено неудачно
* CANCELLED - отменено пользователем или администратором
* COMPLETED - выполнение завершено успешно
* RUNNING - выполняется 
* TIMEOUT - завершено из-за достижения лимита времени 

RunTime - время, которое прошло с момента запуска Вашей программы. То есть текущие время выполнение.

StartTime - приблизительное время начала выполнения работы(Может быть, программа начнет выполнения раньше данного времени. 
Это полная случайность. Как он захочет, так и будет).

Если Вам нужно завершить дострочно выполнение работы, используется следующая команда:
```bash 
scancel 4276916
```

После завершения выполения задачи файлы cudaLab-%j.err и cudaLab-%j.out обновятся. 

С командами Linux Вы разберетесь сами)

