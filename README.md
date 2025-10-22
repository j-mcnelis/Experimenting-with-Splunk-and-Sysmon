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
- This is a placeholder section title that may get changed in the future depending on how I proceed with the project
