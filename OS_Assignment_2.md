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

        close(pipeFd[1]); //closes the write end of pipe.



        char outputBuffer[1024];



        ssize_t bytesRead;



        while ((bytesRead = read(pipeFd[0], outputBuffer, sizeof(outputBuffer))) > 0) {



            write(STDOUT_FILENO, outputBuffer, bytesRead);//it will write the output of the child process to the STDOUT of the parent process.



        }



        close(pipeFd[0]); //closes the read end of the pipe.



        wait(NULL);



    }



    return 0;



}



------------*------------*------------*------------*------------*------------


WC commabd in Linux




------------*------------*------------*------------*------------*------------



#include <stdio.h>

#include <stdlib.h>

#include <unistd.h>

#include <sys/types.h>

#include <sys/wait.h>

#include <string.h>



int main(int argc, char *argv[]) {//the below of the code is same as grep.c. The only difference is that the argc not equals to 2 part because the command will now have low number of character than the grep.c.

    if (argc != 2) {

        fprintf(stderr, "Usage: %s <filename>\n", argv[0]);

        printf("Invalid arguments\n  "); //it will check whether the command wrote to shell is valid or not. In this case which is grep command



        printf("Invalid command written to shell\n   ");

        return 1;

    }



    const char *filename = argv[1];



    int pipe_fd[2];



    if (pipe(pipe_fd) == -1) {

        perror("pipe");

        return 1;

    }



    pid_t child_pid = fork();//creating a child process using fork system call.



    if (child_pid == -1) {

        perror("fork");

        return 1;

    }



    if (child_pid == 0) { 

        close(pipe_fd[0]); 

        dup2(pipe_fd[1], STDOUT_FILENO); 



        execlp("wc", "wc", filename, NULL);

        perror("execlp");

        exit(1);

    } else { 

        close(pipe_fd[1]); 

        char outputBuffer[1024];

        ssize_t bytesRead;



        if ((bytesRead = read(pipe_fd[0], outputBuffer, sizeof(outputBuffer))) > 0) { //reads data from the read end of the pipe into the outputBuffer array.And then it gets tokenized and printed.

            char *lines = strtok(outputBuffer, " ");

            char *words = strtok(NULL, " ");

            char *characters = strtok(NULL, " ");

            printf("Number of lines: %s\n", lines); //printing the number of lines,

            printf("Number of words: %s\n", words);//printing the number of words.

            printf("Number of characters: %s", characters);//printing the number of characters.

        }



        close(pipe_fd[0]); 

        wait(NULL); //now this system call will suspends execution of the calling process until one of its children terminates.

        

    }



    return 0;

}



------------*------------*------------*------------*------------*------------


