
## perform a controlled test

This tests whether Hydra can establish an RDP session and authenticate using the domain-qualified account.
`hydra -l bob -p 'YOUR_KNOWN_PASSWORD' rdp://172.16.10.2/lab.local -t 1 -V`
<img width="1402" height="335" alt="image" src="https://github.com/user-attachments/assets/d394700b-caf1-4a2d-9a19-a6210f11e97b" />

successful remote logon detected rule will be triggered in Wazuh (eventid 4624):
<img width="2533" height="249" alt="image" src="https://github.com/user-attachments/assets/7cf8493c-c824-4a20-9d0b-cd0de5933bb2" />

valid password found:
<img width="1925" height="412" alt="image" src="https://github.com/user-attachments/assets/495769af-a228-4a1f-b4fc-94c75bcd76a6" />

## perform brute force with a word list containing 100 passwords
* the first test will use 100 passwords to brute force account lab.local\bob. passwords taken from rockyou 2009 data breach, which come with kali linux. `head -n 100 /usr/share/wordlists/rockyou.txt > /home/kali/password_list_100.txt`
* 

 brute force from terminal: `hydra -l bob -P /home/kali/password_list_100.txt rdp://172.16.10.2/lab.local`
