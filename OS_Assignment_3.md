#include <stdio.h>

#include <stdlib.h>

#include <unistd.h>

#include <string.h>

#include <time.h>

#include <signal.h>

#include <sys/time.h>

#include <ucontext.h>

#include <sys/types.h>

#include <sys/wait.h>

#include <sys/time.h>

#include <pthread.h>

#include <fcntl.h>

#include <semaphore.h>

#include <sys/resource.h>

#define MAX_INPUT_SIZE 102

#define MAX_HISTORY_SIZE 100

#define MAX_HISTORY_DISPLAY 10

char *history[MAX_HISTORY_SIZE];

int history_count = 0;

// define proc_list and data_sec

struct CommandRecord

{

    int pid;

    time_t start_time;

    time_t end_time;

    double time1; // Time while executing

    double time2; // Time spent in ready queue

    char command[MAX_INPUT_SIZE];
};

struct data_sec

{

    pid_t pros;

    int status;

    char command[MAX_INPUT_SIZE];

    int output_pipe[2]; // Pipe for capturing child process output

    time_t execution_time;

} proc_list[100];

void print_message(const char *message);

void display_history()

{

    printf("\nCommand History:\n");

    for (int i = 0; i < history_count; i++)

    {

        printf("%d: %s\n", i + 1, history[i]);
    }
}

struct CommandRecord command_records[MAX_HISTORY_SIZE];

int command_record_count = 0;

void add_to_history(const char *command)

{

    if (history_count < MAX_HISTORY_SIZE)

    {

        history[history_count++] = strdup(command);
    }

    else

    {

        free(history[0]);

        for (int i = 0; i < history_count - 1; i++)

        {

            history[i] = history[i + 1];
        }

        history[history_count - 1] = strdup(command);
    }
}

void record_command(int pid, const char *command)

{

    if (command_record_count < MAX_HISTORY_SIZE)

    {

        command_records[command_record_count].pid = pid;

        command_records[command_record_count].start_time = time(NULL);

        command_records[command_record_count].end_time = -1; // Initialize to -1

        strncpy(command_records[command_record_count].command, command, sizeof(command_records[command_record_count].command));

        command_record_count++;
    }
}

void display_command_history()

{

    printf("\nCommand Execution History:\n");

    for (int i = 0; i < command_record_count; i++)

    {

        printf("%d: %s", i + 1, command_records[i].command);

        if (command_records[i].end_time != -1)

        {

            double execution_time = difftime(command_records[i].end_time, command_records[i].start_time);

            printf(" (Execution time: %.2lf seconds)", execution_time);
        }

        printf("\n");
    }
}

// function to calculate the execution time

void run_and_record(const char *command)

{

    struct timeval start_time, end_time;

    struct rusage usage;

    // Record the start time before running the process

    gettimeofday(&start_time, NULL);

    pid_t child = fork();

    if (child == 0)

    {

        // Child process

        execl("/bin/sh", "/bin/sh", "-c", command, NULL);

        perror("execl");

        exit(1);
    }

    else if (child > 0)

    {

        int status;

        wait4(child, &status, 0, &usage); // Calulculates the time at user level which doesn't includes the time spent in ready queue. For that use System level

        // Record the end time after the process has finished

        gettimeofday(&end_time, NULL);

        // Calculate the execution time in milliseconds

        double execution_time_ms = (end_time.tv_sec - start_time.tv_sec) * 1000.0 + (end_time.tv_usec - start_time.tv_usec) / 1000.0;

        // Get user and system CPU time from the resource usage

        double user_time_ms = usage.ru_utime.tv_sec * 1000.0 + usage.ru_utime.tv_usec / 1000.0;

        double system_time_ms = usage.ru_stime.tv_sec * 1000.0 + usage.ru_stime.tv_usec / 1000.0;

        printf("Execution time: %.2lf milliseconds\n", execution_time_ms);

        printf("User CPU time: %.2lf milliseconds\n", user_time_ms);

        printf("System CPU time: %.2lf milliseconds\n", system_time_ms);
    }

    else

    {

        perror("Failed to fork a child process");
    }
}

void display_command_records()

{

    printf("\nCommand Execution Records:\n");

    printf("%-6s %-6s %-15s %-15s %-15s\n", "Index", "PID", "Command", "Start Time", "End Time");

    for (int i = 0; i < command_record_count; i++)

    {

        printf("%-6d %-6d %-15s %-15s %-15s\n",

               i + 1,

               command_records[i].pid,

               command_records[i].command,

               ctime(&command_records[i].start_time),

               command_records[i].end_time == -1 ? "N/A" : ctime(&command_records[i].end_time));
    }
}

int launch(const char *command, struct data_sec *process)

{

    int pipe_fd[2];

    if (pipe(pipe_fd) == -1)

    {

        perror("pipe");

        exit(1);
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

        char *args[MAX_INPUT_SIZE];

        int arg_count = 0;

        char *token = strtok((char *)command, " \t\n");

        while (token != NULL)

        {

            args[arg_count++] = token;

            token = strtok(NULL, " \t\n");
        }

        args[arg_count] = NULL;

        // Close the read end of the pipe

        close(pipe_fd[0]);

        // Redirect stdout to the write end of the pipe

        if (dup2(pipe_fd[1], STDOUT_FILENO) == -1)

        {

            perror("dup2");

            exit(1);
        }

        close(pipe_fd[1]);

        execvp(args[0], args);

        perror("execvp");

        exit(1);
    }

    else

    {

        // Parent process

        close(pipe_fd[1]); // Close the write end of the pipe

        struct timeval start_time, end_time;

        gettimeofday(&start_time, NULL);

        record_command(child_pid, command);

        // Read and print the child process's output

        char output_line[1024];

        FILE *output_stream = fdopen(pipe_fd[0], "r");

        if (output_stream)

        {

            while (fgets(output_line, sizeof(output_line), output_stream) != NULL)

            {

                // store the command name in array name command_name

                char command_name[100];

                strcpy(command_name, command);

                print_message(output_line); // changed
            }

            fclose(output_stream);
        }

        else

        {

            perror("fdopen");
        }

        close(pipe_fd[0]);

        if (waitpid(child_pid, NULL, WUNTRACED) != -1)

        {

            gettimeofday(&end_time, NULL);

            for (int i = 0; i < command_record_count; i++)

            {

                if (command_records[i].pid == child_pid)

                {

                    double execution_time_ms = (end_time.tv_sec - start_time.tv_sec) * 1000.0 + (end_time.tv_usec - start_time.tv_usec) / 1000.0;

                    if (command_records[i].end_time == -1)

                    {

                        // If the process is not continued

                        if (command_records[i].time1 == 0)

                        {

                            // If time1 has not been set previously, set it as execution time

                            command_records[i].time1 = execution_time_ms;
                        }

                        else

                        {

                            // Otherwise, accumulate execution time in time2

                            command_records[i].time2 += execution_time_ms;
                        }
                    }

                    // Set end time and calculate total execution time

                    command_records[i].end_time = end_time.tv_sec;

                    double total_execution_time = command_records[i].time1 + command_records[i].time2;

                    process->execution_time = total_execution_time;

                    char message[102]; // changed akash

                    snprintf(message, sizeof(message), "Execution time: %.2lf milliseconds\n", total_execution_time); // changed akash

                    print_message(message); // changed akash

                    break;
                }
            }
        }
    }

    return 0;
}

//*************************

//**********************

// xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

//*******SCHEDULER****************

sem_t proc_list_semaphore;

pthread_mutex_t print_mutex; // Mutex for protecting printf

struct OutputData

{

    pid_t child_pid;

    char output[1024]; // Adjust the buffer size as needed

    time_t execution_time;

} output_data[100];

int count = 0;

int nproc = 100;

int output_count = 0;

void print_message(const char *message)

{

    pthread_mutex_lock(&print_mutex); // Acquire the lock

    printf("%s", message);

    fflush(stdout); // Flush the output buffer

    pthread_mutex_unlock(&print_mutex); // Release the lock
}

void *schedular(void *arg)

{

    int i = 0;

    while (count < 6)

    {

        continue;
    }

    while (1)

    {

        if (proc_list[i].status == 0)

        {

            sem_post(&proc_list_semaphore);

            i = (i + 1) % count;

            continue;
        }

        else

        {

            if (kill(proc_list[i].pros, SIGCONT) == 0)

            {

                char message[100];

                sprintf(message, "Child %d has been continued\n", i);

                print_message(message);

                // fprintf(stderr, "Child %d has been continued\n", i);

                // fflush(stderr);
            }

            usleep(5 * 10);

            int child;

            if (waitpid(proc_list[i].pros, &child, WNOHANG) > 0)

            {

                proc_list[i].status = 0;

                char message[100];

                output_data[output_count].child_pid = proc_list[i].pros;

                output_data[output_count].execution_time = proc_list[i].execution_time;

                strncpy(output_data[output_count].output, proc_list[i].command, sizeof(output_data[output_count].output));

                output_count++;

                sprintf(message, "Child %d has terminated\n", i);

                print_message(message);

                // printf("%s\n", message);

                // fflush(stdout);

                //  Capture and store child process output if needed

                //  OutputData(i);
            }

            else

            {

                if (kill(proc_list[i].pros, SIGSTOP) == 0)

                {

                    char message[100];

                    sprintf(message, "Child %d has been stopped\n", i);

                    print_message(message);

                    // printf("%s\n", message);

                    // fflush(stdout);
                }

                else

                {

                    char message[100];

                    sprintf(message, "Child %d is still running\n", i);

                    print_message(message);

                    // printf("%s\n", message);

                    // fflush(stdout);
                }

                sem_post(&proc_list_semaphore);
            }

            i = (i + 1) % count;
        }
    }

    return NULL;
}

int EmptyInput(const char *input)

{

    if (input == NULL || strlen(input) == 0)

    {

        return 1;
    }

    for (size_t i = 0; i < strlen(input); i++)

    {

        if (input[i] != ' ' && input[i] != '\t' && input[i] != '\n' && input[i] != '\r')

        {

            return 0; // Input is not empty or blank
        }
    }

    return 1;
}

int main()

{

    int status, n = 7; // n used for certain no. of enteries

    char input[MAX_INPUT_SIZE];

    pthread_t scheduler_thread;

    int ncpu, q;

    if (sem_init(&proc_list_semaphore, 0, 1) == -1)

    {

        perror("sem_init");

        return 1;
    }

    if (pthread_mutex_init(&print_mutex, NULL) != 0)

    {

        perror("pthread_mutex_init");

        return 1;
    }

    // can move this part to below and can make loop for certain no. of enteries

    // giving the desired output most of the times.

    if (pthread_create(&scheduler_thread, NULL, schedular, NULL) != 0)

    {

        perror("pthread_create");

        return 1;
    }

    while (1)

    {

        printf("simple-shell$ ");

        if (count >= nproc)

        {

            printf("Maximum number of processes reached.\n");

            continue;
        }

        // Variables

        __pid_t child;

        if (fgets(input, sizeof(input), stdin) == NULL)

        {

            perror("Error reading input");
        }

        input[strcspn(input, "\n")] = '\0';

        if (EmptyInput(input))

        {

            printf("Input is empty or contains only blank spaces.\n");
        }

        else

        {

            if (strcmp(input, "exit") == 0)

            {

                printf("Exit Successfully\n");

                // display_command_records();

                //   to terminate the scheduler  if needed.

                break;
            }

            else if (strcmp(input, "history") == 0)

            {

                display_history();
            }

            else

            {

                // Execute the command asynchronously

                child = fork();

                if (child == 0)

                {

                    // Child process

                    close(proc_list[count].output_pipe[0]); // Close read end of the pipe

                    dup2(proc_list[count].output_pipe[1], STDOUT_FILENO); // Redirect child's stdout to the pipe

                    close(proc_list[count].output_pipe[1]); // Close write end of the pipe

                    // telling the to sleep and hoping till that time parent's else if  part has been runned and has send the stop signal to the

                    usleep(3 * 1000);

                    launch(input, &proc_list[count]);

                    exit(0);
                }

                else if (child > 0)

                {

                    if (sem_wait(&proc_list_semaphore) == -1)

                    {

                        perror("sem_wait");

                        return 1;
                    }

                    proc_list[count].pros = child;

                    strncpy(proc_list[count].command, input, sizeof(input));

                    proc_list[count].status = 1;

                    // Create a pipe for capturing the child process output

                    if (pipe(proc_list[count].output_pipe) == -1)

                    {

                        perror("pipe");

                        exit(1);
                    }

                    sem_post(&proc_list_semaphore);

                    if (kill(proc_list[count].pros, SIGSTOP) == 0)

                    {

                        char message[100];

                        sprintf(message, "Child %d has been created\n", count);

                        print_message(message);
                    }

                    else

                    {

                        perror("Error in stopping the given process\n");

                        exit(0);
                    }
                }

                else

                {

                    perror("Failed to fork a child process");
                }
            }

            count = (count + 1) % nproc;

            n--;
        }
    }

    // showing the output of the processes

    for (int m = 0; m < output_count; m++)

    {

        printf("Child %d: %s\n", output_data[m].child_pid, output_data[m].output);

        printf("Execution time: %ld milliseconds\n", output_data[m].execution_time);
    }

    // Wait for the scheduler thread to complete

    if (pthread_join(scheduler_thread, NULL))

    {

        perror("scheduler_thread");

        return 1;
    }

    // destroyed the proclist sem object

    if (sem_destroy(&proc_list_semaphore) != 0)

    {

        perror("proc_list_sem_destroy");

        return 1;
    }

    // destroying the print message mutex

    if (pthread_mutex_destroy(&print_mutex))

    {

        perror("proc_list_sem_destroy");

        return 1;
    }

    return 0;
}

// ./fib 09

// ./fib 40

//  echo hello world

// ./fib 67

// ./fib 04

// ./fib 01

// submit ./fib 09

//  echo hello world
