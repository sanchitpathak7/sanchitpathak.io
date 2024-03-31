+++
title = 'Linux Cheat Sheet'
date = 2024-03-05
+++

Top Linux Commands

- find: Search for files and directories based on various criteria.
```
find /path/to/search -name "*.txt"
```

- grep: Use advanced patterns and options for searching text.
```
grep -E "pattern1|pattern2" file.txt
```

- awk: A powerful text processing tool for extracting and manipulating data.
```
awk '{print $1}' file.txt
```

- sed: Stream editor for modifying text using patterns.
```
sed 's/search/replace/' file.txt
```

- tar: Create or extract compressed archive files.
```
tar -czvf archive.tar.gz /path/to/directory
```

- rsync: Synchronize files and directories between two locations.
```
rsync -av /source/directory/ /destination/directory/
```

- iptables: Configure firewall rules.
```
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```

- cron: Schedule tasks to run at specific times.
```
crontab -e
```

- systemd: Control system services and manage system settings.
```
systemctl start|stop|restart|status service_name
```

- journalctl: Query and display messages from the systemd journal.
```
journalctl -u service_name
```

- tcpdump: Capture and analyze network packets.
```
tcpdump -i eth0 -n 'port 80'
```

- strace: Trace system calls and signals.
```
strace -p PID
```

- lsof: List open files and the processes that opened them.
```
lsof -i :port_number
```

- Convert a File to Sparse (Files with blocks of empty data that are not allocated on disk)
```
fallocate -d sparsefile
```