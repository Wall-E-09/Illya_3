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
void close_all_files(void);

int main() {
    // Перетворення на демона
    daemonize();
    
    // Встановлення обробників сигналів
    struct sigaction sa;
    sa.sa_handler = handle_signal;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_RESTART; // автоматичне відновлення системних викликів
    
    sigaction(SIGUSR1, &sa, NULL);
    sigaction(SIGINT, &sa, NULL);
    
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
        syslog(LOG_ERR, "Failed to create new session");
        exit(EXIT_FAILURE);
    }
    
    // Ігнорування сигналу SIGHUP для захисту від розриву сесії
    signal(SIGHUP, SIG_IGN);
    
    // Додатковий fork для запобігання отриманню керуючого терміналу
    if ((pid = fork()) < 0) {
        syslog(LOG_ERR, "Second fork failed");
        exit(EXIT_FAILURE);
    } else if (pid != 0) {
        exit(EXIT_SUCCESS);
    }
    
    // 3b. Зміна поточного каталогу на кореневий
    if (chdir("/") < 0) {
        syslog(LOG_ERR, "Failed to change directory to /");
        exit(EXIT_FAILURE);
    }
    
    // 3c. Закриття всіх відкритих файлових дескрипторів
    close_all_files();
    
    // Перенаправлення stdin, stdout, stderr
    int fd = open("/dev/null", O_RDWR);
    if (fd != -1) {
        dup2(fd, STDIN_FILENO);
        dup2(fd, STDOUT_FILENO);
        dup2(fd, STDERR_FILENO);
        if (fd > STDERR_FILENO) {
            close(fd);
        }
    }
    
    // Встановлення маски прав для файлів (0022)
    umask(022);
    
    // Відкриття системного журналу
    openlog("i38a1-daemon", LOG_PID, LOG_LOCAL0);
    syslog(LOG_INFO, "Daemon started successfully");
}

void close_all_files(void) {
    struct rlimit flim;
    getrlimit(RLIMIT_NOFILE, &flim);
    
    for (int fd = 0; fd < (int)flim.rlim_max; fd++) {
        close(fd);
    }
}

void handle_signal(int signum) {
    switch(signum) {
        case SIGUSR1:
            syslog(LOG_INFO, "Received user-defined signal 1 (SIGUSR1)");
            break;
        case SIGINT:
            syslog(LOG_INFO, "Received interrupt signal (SIGINT), shutting down");
            closelog();
            exit(EXIT_SUCCESS);
            break;
        default:
            syslog(LOG_WARNING, "Received unknown signal %d", signum);
            break;
    }
}
