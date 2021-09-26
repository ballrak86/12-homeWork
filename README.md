## Описание файлов в директории
logFileFull.log - полный лог выполнения  
used_commands.txt - команды которые использовал  
usedInetResorce.txt - ресурсы в интернете которые мне помогли решить ДЗ  

Vagrant_folder - все что понадобится для поднятия VM и краткое описание файлов в ней  
Vagrantfile - вагрант файл  

# VagrantFile 
Указал свой vagrant Cloud с подготовленным box файлом  
config.vm.box = "alenchik/PAM"

## Описание как запустить виртуальную машину (кратко)
Выполнить команду
```
vagrant up
```
Зайти на ВМ и выполнить нужные комманды  
Отключим синхронизацию времени и выставим выходной день
```
systemctl stop chronyd; date --set "Sun Sep 26 00:00:00 UTC 2021"
```
Зайдем под пользователем otus (pass: 123) который добавлен в группу admin (и эта группа у пользователя активная), доступ для этой группы разрешен, потом выполним logout
```
ssh otus@127.0.0.1
```
Теперь попробуем зайти под пользователем otus2 (pass: 123), он зайти не сможет т.к. его нет в группе admin
```
ssh otus2@127.0.0.1
```

## Подробнее о проделанной работе
Созданы два пользователя otus, otus2 у обоих пароль 123. Пользователь otus добавлен в группу admin и выставляем ее активной. gid группы admin 1002 (узнали из вывода id otus).
```
sudo useradd otus
echo "123"|sudo passwd --stdin otus
groupadd admin
usermod -a -G admin otus
usermod -g admin otus
useradd otus2
echo "123"|sudo passwd --stdin otus2
id otus
  uid=1001(otus) gid=1002(admin) groups=1002(admin)
```
В файле time.conf нельза указывать группу пользователей, можно указывать только пользователей. С помощью только этого файла задачу не решить. Начал смотреть в сторону модулей PAM. И нашел нужный модуль, pam_succeed_if.so который позволяет сравнить gid активной группы пользователя и числа, в нашем случае gid 1002. Так же ставим проверку sufficient, если попали в группу то дальнейшей проверки account не требуется. Для тех кто не входит в группу admin проверяем время входа.
```
nano /etc/pam.d/sshd
cat /etc/pam.d/sshd
  #%PAM-1.0
  auth       required     pam_sepermit.so
  auth       substack     password-auth
  auth       include      postlogin
  # Used with polkit to reauthorize users in remote sessions
  -auth      optional     pam_reauthorize.so prepare
  account    required     pam_nologin.so
  account    sufficient   pam_succeed_if.so gid eq 1002 #использование модуля pam_succeed_if
  account    required     pam_time.so #использование модуля pam_time
  account    include      password-auth
  password   include      password-auth
  # pam_selinux.so close should be the first session rule
  session    required     pam_selinux.so close
  session    required     pam_loginuid.so
  # pam_selinux.so open should only be followed by sessions to be executed in the user context
  session    required     pam_selinux.so open env_params
  session    required     pam_namespace.so
  session    optional     pam_keyinit.so force revoke
  session    include      password-auth
  session    include      postlogin
  # Used with polkit to reauthorize users in remote sessions
  -session   optional     pam_reauthorize.so prepare
```
Блокируем доступ по sshd, для пользвоателей в выходные дни. Пользователь vagrant добавлен, чтобы не возникло проблем с входом под ним.
```
nano /etc/security/time.conf
cat /etc/security/time.conf | grep -v '#' | grep -v '^$'
  sshd;*;!vagrant;!Wd0000-2400
```
Убеждаемся что сегодня выдоной день и проверяем вход под ранее созданными пользователями. И включаем доступ по паролю.
```
date
  Sat Sep 25 17:29:57 UTC 2021
sed -i 's/^PasswordAuthentication.*$/PasswordAuthentication yes/' /etc/ssh/sshd_config && systemctl restart sshd.service
ssh otus2@127.0.0.1
  otus2@127.0.0.1's password:
  Authentication failed.
ssh otus@127.0.0.1
  otus@127.0.0.1's password:
  [otus@vm-gkuOA1801016 ~]$ 
```
P.S. Рассмотре только один вариант решения, больше в задании и не требовалось. Так же можно было решить через модуль pam_exec, и написать внеший скирпт проверки группы.


📚Домашнее задание/проектная работа разработано(-на) для курса ["Administrator Linux. Professional"](https://otus.ru/lessons/linux-professional/)