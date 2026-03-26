## Лабораторная работа №1 (продвинутый уровень)

Был выбран 2 вариант выполнения лабораторной работы, в котором необходимо было разработать утилиту, которая запускает команду в контейнере.

Был создан скрипт на питоне, который создает контейнер с ограничением по памяти 256мб.

```
import os, sys, json, ctypes

libc = ctypes.CDLL('libc.so.6', use_errno=True)

CLONE_NEWUTS = 0x04000000
CLONE_NEWPID = 0x20000000
CLONE_NEWNS = 0x00020000


def limit_resources(container_id):
    cpath = f"/sys/fs/cgroup/{container_id}"
    os.makedirs(cpath, exist_ok=True)
    with open(f"{cpath}/memory.max", 'w') as f: f.write("256M")
    with open(f"{cpath}/cgroup.procs", 'w') as f: f.write(str(os.getpid()))


def setup_overlay(container_id):
    base = f"/var/lib/my-tool/{container_id}"
    lower = "/tmp/alpine/rootfs"
    upper, work, merged = f"{base}/upper", f"{base}/work", f"{base}/merged"
    for d in [upper, work, merged]: os.makedirs(d, exist_ok=True)

    opts = f"lowerdir={lower},upperdir={upper},workdir={work}"
    libc.mount(b"overlay", merged.encode(), b"overlay", 0, opts.encode())
    return merged


def child_process(container_id):
    config = json.load(open('config.json'))

    limit_resources(container_id)

    hostname = config.get('hostname', 'container')
    libc.sethostname(hostname.encode(), len(hostname))

    merged_path = setup_overlay(container_id)

    proc_path = os.path.join(merged_path, 'proc')
    os.makedirs(proc_path, exist_ok=True)
    libc.mount(b"proc", proc_path.encode(), b"proc", 0, None)

    os.chroot(merged_path)
    os.chdir("/")

    args = config['process']['args']
    os.execvp(args[0], args)


def main():
    if len(sys.argv) < 3:
        print("Usage: sudo python3 tool.py run <id>")
        return


    try:
        libc.unshare(CLONE_NEWUTS | CLONE_NEWPID | CLONE_NEWNS)
    except Exception as e:
        print(f"Failed unshare: {e}")
        return

    pid = os.fork()
    if pid == 0:
        child_process(sys.argv[2])
    else:
        os.waitpid(pid, 0)
        print("\n[Контейнер завершен]")


if __name__ == "__main__":
    main()
```

В одну папку со скриптом необходимо разместить конфигурационный json файл:
```
{
  "hostname": "my-secure-box",
  "process": {
    "args": ["/bin/sh"],
    "cwd": "/"
  }
}
```

Удостоверимся, что скрипт работает.

Для начала подготовим файловую систему для контейнейра и скачаем минимальный образ Alpine.<img width="1214" height="431" alt="2026-03-23_21-35-45" src="https://github.com/user-attachments/assets/a861a8ef-2a82-495a-b840-4b6b7e1fd3e2" />

Запустим наш скрипт:
```
sudo python3 script.py run my-cont-01
```
Итак, скрипт запущен, контейнер работает. Проверим, что все так, как должно быть. 

Проверим, что namespace изолированы.<img width="779" height="98" alt="2026-03-26_21-20-17" src="https://github.com/user-attachments/assets/51b51437-08e6-4853-be08-66075051a934" />

В самом контейнере мы видим имя, указанное в кофигурационном в файле.

<img width="817" height="71" alt="2026-03-26_21-14-05" src="https://github.com/user-attachments/assets/7d63f7f2-2198-4065-b4a0-c8fa8a415a40" />

А вне контейнера выводится имя виртуальной машины.


Посмотрим на изоляцию процессов. В самом контейнере мы видим всего лишь несколько процессов.

<img width="865" height="152" alt="2026-03-23_21-36-22" src="https://github.com/user-attachments/assets/d0a736d3-ede9-4938-8605-d042ac0f7773" />

Тогда как вне контейнера процессов очень много.

<img width="816" height="585" alt="2026-03-26_21-05-31" src="https://github.com/user-attachments/assets/8fbcbed2-141b-4d90-9a90-a97bc2e09854" />

Проверим работу файловой системы. Для этого создадим файл в контейнере.
<img width="686" height="123" alt="2026-03-26_21-22-06" src="https://github.com/user-attachments/assets/a6e2d98a-11bf-4bae-bd55-9433c63ca06c" />

<img width="1005" height="162" alt="2026-03-26_21-24-20" src="https://github.com/user-attachments/assets/ab5aa254-84e8-487b-a57d-204c3d2ff38c" />

Вне контейнера мы видим его не в той же директории, где запущен контейнер, а в папке upper.


Как можно заметить, скрипт отлично справляется с поставленной задачей)
