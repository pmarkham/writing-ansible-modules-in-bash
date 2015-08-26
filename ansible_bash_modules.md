# Writing Ansible Modules in Bash


## License

This work is licensed under a Creative Commons Attribution-NonCommercial 4.0 International License.

http://creativecommons.org/licenses/by-nc/4.0/

## Introduction
One of the strengths of Ansible is that you can write modules for it in any language. Whilst there are advantages to writing modules in Python, such as a bunch of helper routines, the only thing you miss out on by using another language is the ability for your module to support dry run mode.  

In this guide I'm going to look at writing modules in bash, although the same principles can be used in any scripting language. 

### Why Bash?
Although bash lacks a lot of features compared to languages such as Python, Ruby or Perl, a large number of people have some experience with it. Turning a bash script into an Ansible module doesn't take much additional knowledge. 

###  Input
Ansible will run your module and pass it one argument: the name of a file containing the module arguments you specified in your playbook. For example, if you had 

```
bashmod: dest=/tmp/hello state=present
```

you would get passed the name of a file with the contents

```
dest=/tmp/hello state=preset
```

You can get the name of this file from the `$1` variable and, handily, this file is valid bash syntax so you can `source` it to convert these arguments to bash variables, e.g.

```
source $1
```

This will create the variables `$dest` and `$state`.

### Output

The output from your module must be in JSON format. If you return any other output, Ansible will treat it as a failure. This means you need to capture stdout and stderr from any commands you run. 

Ansible looks for the following variables in the output:

* `changed`: Return this if your module was successful. Set it to `true` if it made any changes or `false` if everything was already in the correct state. 

* `failed`: set this to `true` if your modules failed. You can set it to `false` if you module worked, or you can leave it out and Ansible will assume it worked. 

* `msg`: Return an error message if your module failed. You can also set this to an information message on success. 

Any other variables can be returned and will be displayed by Ansible in the output for the task.

For example:
```
echo '{"changed": true, "msg": "Updated stuff"}'
```

The JSON variable names must be enclosed in double quotes. String values must also be enclosed in double quotes, but numbers, booleans (true or false), lists and dictionaries don't need double quotes.  

Because of the double quotes, I've surrounded the whole string in single quotes. This is ok if you're returning fixed strings, but doesn't work if you want to use variables as bash won't do variable expansion in a single quoted string. 

One way around this is to escape the double quotes, e.g.
```
echo "{\"changed\": true, \"msg\": \"$msg\"}"
```

I find this hard to read and error prone, so I prefer to use `printf` instead:

```
printf '{"changed": true, "msg": "%s"}' "$msg"
```

Some characters have special meaning to JSON and need to be escaped. If you return any output from a command, you're better off escaping it just in case. There are probably numerous ways to do this, but I like to pipe it through a Python one-liner; if you're using Ansible, you're already going to have Python installed. I don't know who came up with this Python command, so I can't give them attribution; thanks, though!

```
$msg = $(echo "$msg" | python -c 'import json,sys; print json.dumps(sys.stdin.read())')
```

This might be a good thing to put in a function so you can easily use it in multiple places. One thing to be careful of if you're using this method is that the Python `json.dumps` command puts starting and ending double quotes around the string. This means you need to modify the above `printf` command to remove the quotes from around the `%s`, as follows:

```
printf '{"changed": true, "msg": %s}' "$msg"
```

###  Example module

This is a simple example module which writes some text to a file and can convert it to upper or lower case. 

```
  1 #!/bin/bash
  2 
  3 function create_file
  4     {
  5     if [ -f "$dest" ]; then
  6         changed="false"
  7         msg="file already exists"
  8     else
  9         echo 'Hello, "world!"' >> $dest
 10         changed="true"
 11         msg="file created"
 12     fi   
 13     contents=$(cat "$dest" 2>&1 | python -c 'import json,sys; print json.dumps(sys.stdin.read())')
 14     }
 15 
 16 function delete_file
 17     {
 18     if [ -f "$dest" ]; then
 19         changed="true"
 20         msg="file deleted"
 21         contents=$(cat "$dest" 2>&1 | python -c 'import json,sys; print json.dumps(sys.stdin.read())')
 22         output=$(rm -f $dest 2>&1 | python -c 'import json,sys; print json.dumps(sys.stdin.read())')
 23         if [ $? -ne 0 ]; then
 24             printf '{"failed": "true", "msg": "error deleting file", "output": %s}' "$output"
 25             exit 1
 26         fi
 27     else
 28         changed="false"
 29         msg="file not present"
 30         contents='""'
 31     fi   
 32     }
 33 
 34 function convert_to_upper
 35     {
 36     if [ ! -f "$dest" ]; then
 37         create_file
 38         msg="$msg, "
 39     fi
 40     current=$(cat $dest)
 41     new=$(echo "$current" | tr '[:lower:]' '[:upper:]')
 42     if [ "$current" = "$new" ]; then
 43         changed="false"
 44         msg="${msg}file not changed"
 45         contents=$(printf "$current" | python -c 'import json,sys; print json.dumps(sys.stdin.read())')
 46     else
 47         echo "$new" > $dest
 48         changed="true"
 49         msg="${msg}file converted to upper case"
 50         contents=$(printf "$new" | python -c 'import json,sys; print json.dumps(sys.stdin.read())')
 51     fi
 52     }
 53 
 54 function convert_to_lower
 55     {
 56     if [ ! -f "$dest" ]; then
 57         create_file
 58         msg="$msg, "
 59     fi
 60     contents=$(ls -l "$dest" 2>&1 | python -c 'import json,sys; print json.dumps(sys.stdin.read())')
 61     current=$(cat $dest)
 62     new=$(echo "$current" | tr '[:upper:]' '[:lower:]')
 63     if [ "$current" = "$new" ]; then
 64         changed="false"
 65         msg="${msg}file not changed"
 66         contents=$(printf "$current" | python -c 'import json,sys; print json.dumps(sys.stdin.read())')
 67     else
 68         echo "$new" > $dest
 69         changed="true"
 70         msg="${msg}file converted to lower case"
 71         contents=$(printf "$new" | python -c 'import json,sys; print json.dumps(sys.stdin.read())')
 72     fi
 73     }
 74 
 75 source $1
 76 
 77 if [ -z "$dest" ]; then
 78     printf '{"failed": "true", "msg": "missing required arguments: dest"}'
 79     exit 1
 80 fi
 81 if [ -z "$state" ]; then
 82     printf '{"failed": "true", "msg": "missing required arguments: state"}'
 83     exit 1
 84 fi
 85 
 86 changed="false"
 87 msg=""
 88 contents=""
 89 
 90 case $state in
 91     present)
 92         create_file
 93         ;;
 94     absent)
 95         delete_file
 96         ;;
 97     upper)
 98         convert_to_upper
 99         ;;
100     lower)
101         convert_to_lower
102         ;;
103     *)
104         printf '{"failed": true, "msg": "invalid state: %s"}' "$state"
105         exit 1
106         ;;
107 esac
108 
109 printf '{"changed": "%s", "msg": "%s", "contents": %s}' "$changed" "$msg" "$contents"
110 
111 exit 0
```

###  Testing Your Module
#### Testing Using `test-module`
If you've installed Ansible from the GitHub repository, you'll have a directory called "hacking". In there is a program called `test-module`. 

`test-module` will properly handle your module outputting lines other than JSON and will display them in the `RAW OUTPUT` section. Your JSON output will be displayed under `PARSED OUTPUT`.

If you've install Ansible via a package, you may not have this directory; you'll either need to do a clone from GitHub or use one of the other methods below to test your module. However, `test-module` is the best option.

Setting up your environment to use it:

* Change directory into the `ansible/hacking` directory. 
* Run `source env-setup`. This will update your environment variable to enable Anduble to run out of this directory.
 * Add the current directory to your path so you can run  `test-module` without having to specify its path all the time:
    ```
    export PATH=$PATH:`pwd` 
    ```

#### Example Run: create the file

This example creates the file `test.txt` with the default text of `Hello, "world!"`

```
 1  $ test-module -m bashmod -a 'dest=test.txt state=present'
 2  * including generated source, if any, saving to: /Users/paul/.ansible_module_generated
 3  * this may offset any line numbers in tracebacks/debuggers!
 4  ***********************************
 5  RAW OUTPUT
 6  {"changed": "true", "msg": "file created", "contents": "Hello, \"world!\"\n"}
 7  ***********************************
 8  PARSED OUTPUT
 9  {
10      "changed": "true",
11      "contents": "Hello, \"world!\"\n",
12      "msg": "file created"
13  } 
```


#### Example Run: file already exists

Here's what happens if you rerun the previous example. As the file already exists, it returns `changed: false`, which indicates that it's already in the correct state.

```
 1  $ test-module -m bashmod -a 'dest=test.txt state=present'
 2  * including generated source, if any, saving to: /Users/paul/.ansible_module_generated
 3  * this may offset any line numbers in tracebacks/debuggers!
 4  ***********************************
 5  RAW OUTPUT
 6  {"changed": "false", "msg": "file already exists", "contents": "Hello, \"world!\"\n"}
 7  ***********************************
 8  PARSED OUTPUT
 9  {
10      "changed": "false",
11      "contents": "Hello, \"world!\"\n",
12      "msg": "file already exists"
13  }
```


#### Example Run: convert to upper case

Convert the contents of the file to upper case. 

```
 1  $ test-module -m bashmod -a 'dest=test.txt state=upper'
 2  * including generated source, if any, saving to: /Users/paul/.ansible_module_generated
 3  * this may offset any line numbers in tracebacks/debuggers!
 4  ***********************************
 5  RAW OUTPUT
 6  {"changed": "true", "msg": "file converted to uppercase", "contents": "HELLO, \"WORLD!\""}
 7  ***********************************
 8  PARSED OUTPUT
 9  {
10      "changed": "true",
11      "contents": "HELLO, \"WORLD!\"",
12      "msg": "file converted to upper case"
13  }
```

#### Example Run: convert to lower case

Convert the contents of the file to lower case.

```
 1  $ test-module -m bashmod -a 'dest=test.txt state=lower'
 2  * including generated source, if any, saving to: /Users/paul/.ansible_module_generated
 3  * this may offset any line numbers in tracebacks/debuggers!
 4  ***********************************
 5  RAW OUTPUT
 6  {"changed": "true", "msg": "file converted to uppercase", "contents": "hello, \"world!\""}
 7  ***********************************
 8  PARSED OUTPUT
 9  {
10      "changed": "true",
11      "contents": "hello, \"world!\"",
12      "msg": "file converted to lower case"
13  }
```


#### Example Run: delete the file

```
 1  $ test-module -m bashmod -a 'dest=test.txt state=absent'
 2  * including generated source, if any, saving to: /Users/paul/.ansible_module_generated
 3  * this may offset any line numbers in tracebacks/debuggers!
 4  ***********************************
 5  RAW OUTPUT
 6  {"changed": "true", "msg": "file deleted", "contents": "hello, \"world!\"\n"}
 7  ***********************************
 8  PARSED OUTPUT
 9  {
10      "changed": "true",
11      "contents": "hello, \"world!\"\n",
12      "msg": "file deleted"
13  }
```

#### Example Run: create file and convert to upper case

This example specifies the state as `upper` but the file hadn't been created yet. The module creates the file and converts the contents to upper case. Both actions are reflected in the `msg` variable that's returned. 

```
 1  $ test-module -m bashmod -a 'dest=test.txt state=upper'
 2  * including generated source, if any, saving to: /Users/paul/.ansible_module_generated
 3  * this may offset any line numbers in tracebacks/debuggers!
 4  ***********************************
 5  RAW OUTPUT
 6  {"changed": "true", "msg": "file created, file converted to uppercase", "contents": "HELLO, \"WORLD!\""}
 7  ***********************************
 8  PARSED OUTPUT
 9  {
10      "changed": "true",
11      "contents": "HELLO, \"WORLD!\"",
12      "msg": "file created, file converted to uppercase"
13  }
```

### Testing Using Ansible Command Line

You can use the `ansible` command to run your module. If your module outputs anything other than JSON, it will be treated as a failure. 

```
 1  $ ansible -c local -i 'localhost,'  -M . -m bashmod -a 'dest=test.txt state=present' all
 2  localhost | success >> {
 3      "changed": "true",
 4      "contents": "Hello, \"world!\"\n",
 5      "msg": "file created"
 6  }
```

### Testing Using bash

 You can execute your module directly by writing the arguments to a file in key=value pairs on a single line. Run the module and pass the name of this file. 
 
 This will not check that your output is valid JSON format, but can be handy if you want to add debugging statements to the module. 
 
```
 1  $ echo 'dest=test.txt state=present' > args
 2  $ bash bashmod args
 3  {"changed": "false", "msg": "file already exists", "contents": "Hello, \"world!\"\n"}
```

You can run your module with the `-x` option to trace its execution. 

```
 1  $ echo 'dest=test.txt state=present' > args
 2  $ bash -x bashmod args
 3  + source args
 4  ++ dest=test.txt
 5  ++ state=present
 6  + '[' -z test.txt ']'
 7  + '[' -z present ']'
 8  + changed=false
 9  + msg=
10  + contents=
11  + case $state in
12  + create_file
13  + '[' -f test.txt ']'
14  + echo 'Hello, "world!"'
15  + changed=true
16  + msg='file created'
17  ++ cat test.txt
18  ++ python -c 'import json,sys; print json.dumps(sys.stdin.read())'
19  + contents='"Hello, \"world!\"\n"'
20  + printf '{"changed": "%s", "msg": "%s", "contents": %s}' true 'file created' '"Hello, \"world!\"\n"'
21  {"changed": "true", "msg": "file created", "contents": "Hello, \"world!\"\n"}+ exit 0
```
