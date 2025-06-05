Це демон-процес, написаний мовою C, який запускається у фоновому режимі, обробляє сигнали `SIGUSR1` та `SIGINT`, і записує події у системний журнал (`syslog`).

``` bash
gcc -o mydaemon daemon.c
```

```
./mydaemon
```

```
ps aux | grep mydaemon
```

```
sudo tail -f /var/log/syslog | grep i38a1-daemon
```

```
kill -USR1 <PID>
```

```
kill -INT <PID>
```
