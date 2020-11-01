# template-daemon
Template for *nix daemon

# Usage
Compile the code: 
```gcc -o firstdaemon daemonize.c```

Start the daemon: 
```./firstdaemon```

Check if everything is working properly: 
```ps -xj | grep firstdaemon```
