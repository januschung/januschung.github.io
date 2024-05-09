# Steps to troubleshoot when home server is down after power loss

1. Connect a laptop to the router and check for internet connection

1. Visit the ESXi UI to see if the server is up

  - If ther server is not up 

    - Connect the server with monitor, keyboard and network cable
    - Power on/Force restart of the server

1. Start up the Firewall and PiHole VMs

1. Check server 2 ESXi UI and restart it if needed

1. Start up VM

1. Bring up all docker services


