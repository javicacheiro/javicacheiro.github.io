## Access the WebUI using a proxy
Create a connection that routes traffic to local port 9876 to the cluster over SSH:

    ssh -C2qTnNf -D 9876 <username>@login.hdp.cesga.es

Now use the tunnel from your browser:
- Configure the browser to use localhost:9876 as a SOCKS v5 proxy
- Select Remote DNS so DNS requests will be resolved in the remote login node not in your local computer. If this is unselected, DNS will be resolved locally and you will have to manually replace the names of the nodes with their corresponding IP addresses
- Browser extensions such as FoxyProxy support pattern matching for URL requests (FoxyProxy Standard or Plus only), so that only requests for specific URLs will be sent over the tunnel

You can also create the tunnel in Windows using PuTTY:

Reference:
  https://azure.microsoft.com/en-us/documentation/articles/hdinsight-linux-ambari-ssh-tunnel/
