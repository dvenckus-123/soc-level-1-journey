
## perform a controlled test

This tests whether Hydra can establish an RDP session and authenticate using the domain-qualified account.
`hydra -l bob -p 'YOUR_KNOWN_PASSWORD' rdp://172.16.10.2/lab.local -t 1 -V`
<img width="1402" height="335" alt="image" src="https://github.com/user-attachments/assets/d394700b-caf1-4a2d-9a19-a6210f11e97b" />

successful remote logon detected rule will be triggered in Wazuh (eventid 4624):
<img width="2533" height="249" alt="image" src="https://github.com/user-attachments/assets/7cf8493c-c824-4a20-9d0b-cd0de5933bb2" />

valid password found:
<img width="1925" height="412" alt="image" src="https://github.com/user-attachments/assets/495769af-a228-4a1f-b4fc-94c75bcd76a6" />
