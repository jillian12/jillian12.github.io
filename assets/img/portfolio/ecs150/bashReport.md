
# Simple Shell

## Summary

This program creates a shell that will take command-line inputs and execute
them in proper order. This is done through a single C file, **sshell.c** , 
which implements the aforementioned shell and parses the command-line 
arguments in accordance with standard Linux formats while catching any errors. 
This shell will then execute various system calls and builtin functions with 
an emphasis on processes, files, and pipes to complete the parsed commands.

## Implementation

1. Parsing the command line arguments and identifying behavior of each 
command.
2. Storing every command in its own struct object and forming a doubly linked 
list with those struct objects. 
3. Executing single or multiple commands successfully based on their system 
calls or recognizing improper commands while reporting the specific error.
4. Displaying return statuses in a completion message and then continuously
prompting the user to enter further commands until exit is entered.

### Parsing Commands Implementation

The execution begins by creating our two main data structures inside of our 
`createCommand` function which is called every time a user is prompted and 
subsequently enters a command into the command line. We use a `struct aCommand` 
to contain each individual command’s data and a `commandList cmdList` which is 
a doubly linked list of `struct commandListNode` objects that then hold the 
`struct aCommand`. This enables us to identify all of the information that a 
given command contains and allows us to access the data at any time through 
`cmdList`. We iterate through each token until the command line is empty to 
determine the specific syscalls present such as file manipulation or piping.
This functionality is handled through two functions: parseIn, for a single 
command, and parseInPipes for multiple commands that will be involved in the
formation of pipes. 

The `struct aCommand` contains: <br> 
>>**processName**: The command
to be executed and serves as our first parameter in our `execvp` call <br>

>>**redirectFileName**: The filename if a file will need to be manipulated, it
also serves as a way for us to classify if output or input redirection is
needed <br>

>>**arguments[ARGS_MAX]**: An array which stores the tokens from the command
line with a max of 16 arguments, this allows us to identify specific
syscalls and manipulate the information given <br>  

>>**cmdSave[CMDLINE_MAX]**: A
copy of the original command line for printing out the completion message with 
a maximum of 512 characters<br>

>>**numOfArgs**: The number of arguments allows us to
ensure that arguments in specific commands are parsed correctly and to error
check the maximum number of arguments <br>

>>**fd[2]**: This int array of size two serves as our file descriptors for the
creation of pipes between processes. This is stored in `struct aCommand` in
that it allows linked aCommand objects to access the previous pipe's fd[0]
and fd[1] enabling the formation pipes. <br>

>>**pid**: Similarly to fd[2], pid is stored in `struct aCommand` to allow for
linked aCommand objects to be parsed through until all pids have been waited on
successfully. <br>

If a piping is required, meaning there are multiple commands, we then store
each of the `struct aCommand` objects in a doubly linked list 
`commandList cmdList`. This is done after parsing and storing our commands. Two
pointers of type commandListNode `p_head and p_tail` are used to represent the
entry point of the list and the last item of the list respectively. The;
advantage of this is that we are able to manipulate and check the previous and
next command. Which is especially useful when identifying a command that will
need to access the `stdout` of the previous command and provide the `stdin` to
the next command.

### Syscall Performance

We have separated our execution into four functions that create processes to
ensure that the command will execute correctly based on the arguments they are
given.

#### Simple commands: Create Process

With simple commands, our program will determine any builtin commands and run
them and before then calling `createProcess` which makes the `execvp` call
required to run a non-builtin simple command.

#### File Manipulation with Input and Output Redirection

With input and output redirection, we call the function  
`createProcessWithOutputRedirection` if the command contains a `>` character,
or `createProcessWithInputRedirection` if the command contains a `<` character
and stores the next argument into our `redirectFileName`. We then will be able
to open the file, if it is valid, and redirect the input or output depending on
the command. Once all commands have completed, we then return to `stdin` and 
`stdout` before allowing the user to enter more commands to ensure
functionality regardless of previous command.

#### Create Process with Piping

We begin our piping by calling `createProcessWithPipes` which iterates through
the number of commands by traversing our doubly linked list through access to
the next node. This decides if it is the first command, one of the two possible
middle commands, or the last command which then determines how we redirect 
`stdin` and `stdout`. We then execute the commands and once they have all run
and returned to their waitpid calls, we then retrieve the status values and
print them to `stderr`.

##### File manipulation with piping

In order to ensure that file manipulation is possible when piping, output and
input redirection is identified in the child processes of 
`createProcessWithPipes` and will then open the file. This is adheres to the 
limitation that there is only supposed to be input manipulation in the first 
command and output manipulation in the last command.

### Limitations and Peculiarities

Our program is limited as noted in the Project1.html document to only work with
a maximum of sixteen arguments. Furthermore, when creating commands to be used
in piping in `parseInPipes` we have a hardcoded max of 4 commands to be run 
together but `createProcessWithPipes` can handle as many commands as the space
allocated for the linked list. In addition, our error handling primarily
focused on the error test cases provided in the Project1.html and sporadically
implemented throughout the code with no central error handling catches or
dedicated functions. Finally, we recognize the repetition and inefficiency in
condensing and refactoring our code as well as potential memory leaks that went
unaddressed, but as full time college students running on coffee we are 
prepared to face the sacrifices in order to submit a fully functional program. 

### Sources 
General c function functionality:
https://cplusplus.com/reference/cstring/strchr/
https://man7.org/linux/man-pages/
https://www.tutorialspoint.com/c_standard_library/index.htm

Create process and piping:
Lecture 3.syscalls -slide 20 and 39

Referenced project 0 and modified in our program to be a doubly linked list:<br>
Giving credit to Joël Porquet-Lupine Project 0: sgrep.c from ECS150 Fall 2022 

Referenced to create our pushd and popd stack:
https://stackoverflow.com/questions/1919975/creating-a-stack-of-strings-in-c
