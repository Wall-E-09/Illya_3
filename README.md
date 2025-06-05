#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <signal.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/resource.h>
#include <syslog.h>
#include <fcntl.h>

void handle_signal(int signum);
void daemonize(void);

int main() {
    // Перетворення на демона
    daemonize();
    
    // Встановлення обробників сигналів
    signal(SIGUSR1, handle_signal);
    signal(SIGINT, handle_signal);
    
    // Головний цикл демона
    while(1) {
        pause(); // Очікування сигналів
    }
    
    return 0;
}

void daemonize(void) {
    pid_t pid;
    
    // 1. Створення дочірнього процесу
    if ((pid = fork()) < 0) {
        perror("fork");
        exit(EXIT_FAILURE);
    } else if (pid != 0) { // Батьківський процес
        printf("Daemon started with PID: %d\n", pid);
        exit(EXIT_SUCCESS);
    }
    
    // 2. Дочірній процес продовжує роботу
    
    // 3a. Створення нової сесії
    if (setsid() < 0) {
        perror("setsid");
        exit(EXIT_FAILURE);
    }
    
    // 3b. Зміна поточного каталогу на кореневий
    if (chdir("/") < 0) {
        perror("chdir");
        exit(EXIT_FAILURE);
    }
    
    // 3c. Закриття всіх відкритих файлових дескрипторів
    struct rlimit flim;
    getrlimit(RLIMIT_NOFILE, &flim);
    for (int fd = 0; fd < flim.rlim_max; fd++) {
        close(fd);
    }
    
    // Перенаправлення stdin, stdout, stderr
    open("/dev/null", O_RDONLY); // stdin
    open("/dev/null", O_RDWR);   // stdout
    open("/dev/null", O_RDWR);   // stderr
    
    // Відкриття системного журналу
    openlog("i38a1-daemon", LOG_PID, LOG_LOCAL0);
    syslog(LOG_INFO, "Daemon started successfully");
}

void handle_signal(int signum) {
    switch(signum) {
        case SIGUSR1:
            syslog(LOG_INFO, "Received signal SIGUSR1");
            break;
        case SIGINT:
            syslog(LOG_INFO, "Received SIGINT, shutting down");
            closelog();
            exit(EXIT_SUCCESS);
            break;
        default:
            syslog(LOG_INFO, "Received unknown signal %d", signum);
            break;
    }
}
