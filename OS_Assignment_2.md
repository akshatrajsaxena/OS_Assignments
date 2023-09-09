//Grep command for linux 
------------*------------*------------*------------*------------*------------

#include <stdio.h>

#include <stdlib.h>

#include <unistd.h>

#include <sys/types.h>

#include <sys/wait.h>



int main(int argc, char *argv[]) {

    if (argc != 3) {

        fprintf(stderr, "Usage: %s search_pattern target_file  ", argv[0]);

        printf("Invalid arguments\n  "); //it will check whether the command wrote to shell is valid or not. In this case which is grep command

        printf("Invalid command written to shell\n   ");

        printf("  Usage: %s <search-pattern> <target-file>", argv[0]);//it will print the usage of the command

        return 1;

    }

    char *searchPattern = argv[1];

    char *targetFile = argv[2];



    int pipeFd[2];//pipeFd[0] is the read end of the pipe and pipeFd[1] is the write end of the pipe.

    if (pipe(pipeFd) == -1) {

        perror("pipe"); //If the pipe system call fails, it prints an error message to stderr using perror.

        return 1;

    }



    pid_t childPid = fork();

    if (childPid == -1) {

        perror("fork");

        return 1;

    }

    if (childPid == 0) {

        close(pipeFd[0]); //closes the read end of the pipe.

        dup2(pipeFd[1], STDOUT_FILENO); // it will redirects the STDOUT of the child process in order to the write end of the pipe

        execlp("grep", "grep", searchPattern, targetFile, NULL);

        perror("execlp");

        exit(1);

    } else {

        char outputBuffer[1024];

        ssize_t bytesRead;

        while ((bytesRead = read(pipeFd[0], outputBuffer, sizeof(outputBuffer))) > 0) {

            write(STDOUT_FILENO, outputBuffer, bytesRead);//it will write the output of the child process to the STDOUT of the parent process.

        }

        close(pipeFd[0]); //closes the read end of the pipe.

        wait(NULL);

        parentchild(5); //just of reference that it will create five none existensial child process

    }

    return 0;

}

------------*------------*------------*------------*------------*------------

