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




simpleshell.c









#include <stdio.h>

#include <stdlib.h>

#include <string.h>

#include <unistd.h>

#include <sys/types.h>

#include <sys/wait.h>

#include <sys/time.h>

#include <dirent.h>



#define MAX_INPUT_SIZE 1024

#define MAX_HISTORY_SIZE 100



char *history[MAX_HISTORY_SIZE];

int history_count = 0;

int pid_list[MAX_HISTORY_SIZE];

int pid_count = 0;

double execution_times[MAX_HISTORY_SIZE];



void add_to_history(const char *command, pid_t pid, double execution_time)

{

    if (history_count < MAX_HISTORY_SIZE)

    {

        history[history_count] = strdup(command);

        pid_list[history_count] = pid;

        execution_times[history_count] = execution_time;

        history_count++;

    }

}



double get_execution_time(struct timeval start_time, struct timeval end_time)



{



    return (end_time.tv_sec - start_time.tv_sec) * 1000.0 + (end_time.tv_usec - start_time.tv_usec) / 1000.0;

}



void display_history()



{



    printf("Process PID     Execution Time (ms)     Processes\n");



    for (int i = 0; i < history_count; i++)



    {



        printf("%d. %d %10.2f %s\n", i + 1, pid_list[i], execution_times[i], history[i]);

    }

}



int launch(const char *command)



{



    char *args[MAX_INPUT_SIZE];



    int arg_count = 0;



    // Tokenize the command while considering quoted strings



    char *token = strtok((char *)command, "\"\t\n "); // Tokenize by spaces and double quotes



    while (token != NULL)



    {



        args[arg_count] = token;



        arg_count++;



        // Check if the token contains a closing double quote



        if (token[strlen(token) - 1] == '"')



        {



            // Remove the closing double quote



            token[strlen(token) - 1] = '\0';



            token = strtok(NULL, "\"\t\n "); // Continue tokenization

        }



        else



        {



            token = strtok(NULL, "\"\t\n ");

        }

    }



    args[arg_count] = NULL;



    if (arg_count == 0)



    {



        return 1;

    }



    // TO EXIT THE SHELL



    if (strcmp(args[0], "exit") == 0)



    {



        struct timeval start_time, end_time;



        gettimeofday(&start_time, NULL);



        pid_t pid = getpid();



        pid_list[pid_count++] = pid;



        gettimeofday(&end_time, NULL);



        double elapsed_time = get_execution_time(start_time, end_time);



        printf("Command executed in %.2f ms\n", elapsed_time);



        printf("PID: %d  Exit successfully\n", pid);



        return 0;

    }



    pid_t child_pid;



    struct timeval start_time, end_time;



    gettimeofday(&start_time, NULL);



    // Create a child process



    child_pid = fork();



    if (child_pid == -1)



    {



        perror("fork");



        exit(1);

    }



    if (child_pid == 0)



    {



        pid_t pid = getpid();



        printf("Process PID: %d\n", pid);



        pid_list[pid_count++] = pid;



        if (strcmp(args[0], "history") == 0)



        {



            display_history();



            exit(0);

        }



        if (strcmp(args[0], "clear") == 0)



        {



            system("clear");



            exit(0);

        }

        // give a tab space  between dest and source address

        // otherwise an error will be generated

        else if (strcmp(args[0], "cp") == 0)

        {

            // checking if arg[1] and arg[2] is present

            if (args[1] == NULL || args[2] == NULL)

            {

                printf("cp: missing file operand\n");

            }

            else

            {

                // Use execvp to run the cp command

                if (execvp("cp", args) == -1)

                {

                    perror("Error executing the cp command");

                }

            }

        }

        // Implement the "which" command here

        if (strcmp(args[0], "which") == 0)

        {

            if (execvp("which", args) == -1)

            {

                perror("execvp");

                exit(1);

            }

        }

        // Inside your launch function



        if (strcmp(args[0], "sort") == 0)

        {

            // Check if there are any command-line options

            int reverse = 0; // By default, not reverse

            int numeric = 0; // By default, not numeric

            int unique = 0;  // By default, not unique

            int key_field = 0;



            int arg_index = 1; // Start checking options from the second argument



            while (args[arg_index] != NULL)

            {

                if (strcmp(args[arg_index], "-r") == 0)

                {

                    reverse = 1;

                }

                else if (strcmp(args[arg_index], "-n") == 0)

                {

                    numeric = 1;

                }

                else if (strcmp(args[arg_index], "-u") == 0)

                {

                    unique = 1;

                }

                else if (strcmp(args[arg_index], "-k") == 0)

                {

                    // Check if there is a key field specified after "-k"

                    if (args[arg_index + 1] != NULL)

                    {

                        key_field = atoi(args[arg_index + 1]);

                        arg_index++; // Move to the next argument

                    }

                    else

                    {

                        perror("Invalid usage of -k option.");

                        exit(1);

                    }

                }

                else

                {

                    break; // Stop parsing options if an unrecognized argument is encountered

                }



                arg_index++; // Move to the next argument

            }



            // At this point, args[arg_index] should point to the first file or input source

            // Implement the sorting logic based on the options you parsed

            // You can use execvp to run the 'sort' command with the specified options

            // ...



            // If you implement 'sort' here, you can use execvp to execute it

            execvp("sort", args + arg_index);



            // If execvp fails, handle the error

            perror("Error executing sort");

            exit(1);

        }



        if (strcmp(args[0], "ls") == 0)



        {



            if (arg_count == 1)



            {



                // Just 'ls' command



                execlp("/bin/ls", "ls", (char *)NULL);

            }



            else



            {



                // 'ls' command with arguments



                execvp(args[0], args);

            }



            perror("No such directory");



            exit(1);

        }



        if (strcmp(args[0], "./grep") == 0)



        {



            if (arg_count != 3)



            {



                perror("Usage: ./grep <search_string> <filename>");



                exit(1);

            }



            // Execute the grep command with search string and filename



            execlp("grep", "grep", args[1], args[2], (char *)NULL);



            // If execlp fails, we reach here



            perror("Error executing grep");



            exit(1);

        }



        if (strcmp(args[0], "cat") == 0)



        {



            if (arg_count != 2)



            {



                perror("Usage: cat <filename>");



                exit(1);

            }



            execlp("cat", "cat", args[1], (char *)NULL);



            perror("Error executing cat");



            exit(1);

        }



        // Handle ./fib command



        if (strcmp(args[0], "./fib") == 0)



        {



            // Check if the correct number of arguments are provided for ./fib



            if (arg_count != 2)



            {



                perror("Usage: fib <n>");



                exit(1);

            }



            // Execute the fib program with the specified argument



            execlp("./fib", "./fib", args[1], (char *)NULL);



            perror("Error executing fib");



            exit(1);

        }



        // Handle wc command



        if (strcmp(args[0], "wc") == 0)



        {



            // Check if the correct number of arguments are provided for wc



            if (arg_count != 2)



            {



                perror("Usage: wc <filename>");



                exit(1);

            }



            // Execute the wc program with the specified argument



            execlp("wc", "wc", args[1], (char *)NULL);



            perror("Error executing wc");



            exit(1);

        }



        // perror("No such command exist");



        exit(1);

    }



    else



    {



        int status;



        waitpid(child_pid, &status, 0);



        gettimeofday(&end_time, NULL);



        double elapsed_time = get_execution_time(start_time, end_time);



        printf("Command executed in %.2f ms\n", elapsed_time);



        // Pass the correct execution time to add_to_history



        if (status != -1)



        {



            add_to_history(command, child_pid, elapsed_time);

        }

    }



    return child_pid;

}



int execute_pipeline(char *commands[], int num_commands)

{

    pid_t child_pids[num_commands];

    int status;

    int pipefds[num_commands - 1][2];



    for (int i = 0; i < num_commands; i++)

    {

        if (i < num_commands - 1)

        {

            if (pipe(pipefds[i]) == -1)

            {

                perror("pipe");

                exit(1);

            }

        }



        pid_t child_pid = fork();



        if (child_pid == -1)

        {

            perror("fork");

            exit(1);

        }



        if (child_pid == 0)

        {

            // Child process



            // Redirect input from the previous command (if not the first command)

            if (i > 0)

            {

                dup2(pipefds[i - 1][0], STDIN_FILENO); // Redirect input from the previous pipe

                close(pipefds[i - 1][0]);              // Close read end of the previous pipe

            }



            // Redirect output to the current pipe (if not the last command)

            if (i < num_commands - 1)

            {

                close(pipefds[i][0]);               // Close read end of the current pipe

                dup2(pipefds[i][1], STDOUT_FILENO); // Redirect output to the current pipe

                close(pipefds[i][1]);               // Close write end of the current pipe

            }



            // Execute the command

            execvp(commands[i], &commands[i]);

            perror("execvp");

            exit(1);

        }

        else

        {

            // Parent process

            child_pids[i] = child_pid;

        }

    }



    // Close all pipe descriptors in the parent

    for (int i = 0; i < num_commands - 1; i++)

    {

        close(pipefds[i][0]);

        close(pipefds[i][1]);

    }



    // Wait for all child processes to complete

    for (int i = 0; i < num_commands; i++)

    {

        waitpid(child_pids[i], &status, 0);

    }



    return 0; // Return an appropriate status code

}



int main()

{

    char input[MAX_INPUT_SIZE];

    int status;



    do

    {

        printf("simple-shell$ ");



        if (fgets(input, sizeof(input), stdin) == NULL)

        {

            perror("Error reading input");

        }



        input[strcspn(input, "\n")] = '\0';



        char *commands[MAX_INPUT_SIZE]; // An array to hold individual commands



        // Tokenize the input based on pipe '|' character

        char *token = strtok(input, "|");

        int num_commands = 0;



        while (token != NULL)

        {

            commands[num_commands] = token;

            num_commands++;

            token = strtok(NULL, "|");

        }



        status = execute_pipeline(commands, num_commands);



    } while (status);



    display_history();



    return 0;

}














