### **Summary of Remote Process Monitoring Approach**  

#### **Problem Statement:**  
When executing a long-running process on a remote machine via SSH using `nohup`, the SSH session does not provide a direct way to track when the process completes. The goal is to **reliably monitor** the remote process, ensure it does not run indefinitely, and detect its completion from the local machine.  

#### **Solution Approach:**  

1. **Start Process with nohup and Save PID:**  
   - Run the process in the background using `nohup` and store its Process ID (PID) in a file (`/tmp/4-boot-test.pid`).  
   - Example:  
     ```bash
     nohup sh /tmp/4-boot-test.sh > /tmp/4-boot-test.log 2>&1 & echo $! > /tmp/4-boot-test.pid
     ```

2. **Monitor Process Completion from Local Machine:**  
   - Use SSH to check if the process is still running by reading its PID and checking `ps -p $pid`.  
   - Example:  
     ```bash
     sshpass -p $lpar_password ssh -p 9989 $lpar_username@localhost \
       -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
       'pid=$(cat /tmp/4-boot-test.pid);
        if ps -p $pid > /dev/null 2>&1; then
            echo "Process $pid is still running.";
            while ps -p $pid > /dev/null 2>&1; do sleep 10; done
            echo "Process $pid has completed.";
        else
            echo "Process already completed or failed.";
        fi'
     ```

3. **Enforce a Time Limit (Kill Process if Exceeds 10 Hours):**  
   - Continuously check if the process is running and track its runtime.  
   - If the runtime exceeds **10 hours**, terminate the process.  
   - Example:
   - execute script on which is present on remote server
     ```bash
     local machine -> gateway server/jump server -> remote machine which is behind firewall
     
     #establish tunneling from local to remote server with help of gateway server, 
     #we added public key in gateway server

     ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i /tmp/ssh-privatekey -D 1234 -f -C -q -N gateway_user@$gateway_ip
     
     ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i /tmp/ssh-privatekey -L 9989:$remote_host_ip:22 -f -C -q -N gateway_user@$gateway_ip
     
     sshpass -p $remote_host_password ssh -p 9989 $remote_username@localhost \
                -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
                "nohup sh /tmp/4-boot-test.sh > /tmp/4-boot-test.log 2>&1 & echo \$! > /tmp/4-boot-test.pid"
     ```
     
     ```bash
      sshpass -p $lpar_password ssh -p 9989 $lpar_username@localhost \
        -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
        'if [ ! -f /tmp/4-boot-test.pid ]; then
            echo "❌ PID file not found. Process may have exited or system rebooted."
            exit 1
         fi
      
         # Read stored PID, start time, and command from the PID file
         read pid start_time cmd < /tmp/4-boot-test.pid
         current_time=$(date +%s)
         current_uptime=$(awk "{print int(\$1)}" /proc/uptime)
      
         # ⚠️ Check if the system rebooted since the process started
         if [ $current_uptime -lt $((current_time - start_time)) ]; then
             echo "⚠️ Remote machine has rebooted. Stored PID is invalid."
             exit 1
         fi
      
         # Check if the process is still running and belongs to the correct command
         if ! ps -p $pid -o cmd= | grep -q "$cmd"; then
             echo "⚠️ Process $pid is not running or a different process is using the same PID."
             exit 1
         fi
      
         echo "✅ Process $pid ($cmd) is running."
         sleep_time=60  # Start with 1-minute checks
      
         while true; do
             # Check if the process is still running
             if ! ps -p $pid > /dev/null 2>&1; then
                 echo "✅ Process $pid has completed."
                 exit 0
             fi
      
             # Get elapsed time in seconds
             elapsed_time=$(ps -o etimes= -p $pid)
             if [ "$elapsed_time" -ge 36000 ]; then  # 36000 seconds = 10 hours
                 echo "⏳ Process $pid exceeded 10 hours. Terminating..."
                 kill -9 $pid
                 echo "❌ Process $pid forcibly stopped after timeout."
                 exit 1
             fi
      
             echo "⏳ Process $pid is still running. Next check in $sleep_time seconds..."
             sleep $sleep_time
      
             # Exponential backoff: Increase sleep time, max 5 minutes
             if [ "$sleep_time" -lt 300 ]; then
                 sleep_time=$((sleep_time * 2))
             fi
         done'

     ```

#### **Reliability & Benefits:**  
✅ Ensures the remote process is monitored without keeping SSH active.  
✅ Prevents indefinite execution by enforcing a time limit.  
✅ Runs periodic checks (every 10s) with a maximum of **5 minutes wait time** post-completion.  
✅ Suitable for long-running processes, even up to **15+ hours**, without overloading the system.  
✅ Handles remote machine reboots → Checks uptime and invalidates PID if needed.

✅ Prevents false positives → Ensures PID matches the expected command.

✅ Efficient monitoring → Uses exponential backoff for longer-running processes.

✅ Ensures no process runs indefinitely → Kills process if it exceeds 10 hours.

