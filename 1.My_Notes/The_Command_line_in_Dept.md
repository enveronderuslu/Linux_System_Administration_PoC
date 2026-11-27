# Standard Streams

## Here Documents
```bash
cat > greeting.txt << EOF
> Hello from the terminal!
> EOF 
```
EOF --> end of file

## Chaining commands
The shell runs each command in turn, regardless of whether the previous one succeeded, with the output of each command shown on separate lines:
```bash 
ls; pwd; whoami
```
the second command runs only if the first one succeeded:

```bash 
cd /tmp && ls
snap-private-tmp  systemd-private-[...]
```
The opposite of && is ||, which runs the second command only if the first one failed:
```bash 
cd /nonexistent || echo "Could not change directory"
bash: cd: /nonexistent: No such file or directory
Could not change directory
```

```bash
ping -c1 localhost && echo "Host is up" || echo "Host is unreachable"
```

## Background execution; &
```bash
sleep 10 &
sleep 10 
```
## Command grouping; {}

{ echo "System Report"; date; uptime; } > report.txt


## Subshell execution; () 
Run commands in a subshell, where the commands are forgotten the moment they finish running. This is useful for isolating side effects like a cd – the directory change does not affect

(export NAME="Alice"; echo "Name is: $NAME")
echo "Name is: $NAME"

# Shell environment variables
## Built-in variables
```bash
env ;  echo $SHELL ;  echo $USER; echo $HOME 

echo §PATH # It tells the shell which directories to search when we type a command name:
```

## sETTING AND  UNSETTING VARIABLES
```BASH
GREETING="Hello, world!"
echo $GREETING
unset GREETING
```















