
## setup & output/screenshots

perform brute force with a word list containing 100 passwords then investigation with alert → pivot → timeline → conclusion.
* the test will use 100 passwords to brute force account lab.local\bob. passwords taken from rockyou 2009 data breach, which come with kali linux. `head -n 100 /usr/share/wordlists/rockyou.txt > /home/kali/password_list_100.txt`
* 

 brute force from terminal: `hydra -l bob -P /home/kali/password_list_100.txt rdp://172.16.10.2/lab.local`



Hydra output:
 <img width="1372" height="792" alt="image" src="https://github.com/user-attachments/assets/75179b34-dbc7-47b5-8151-85ded7f626e5" />

Suricata log (ET INFO RDP - Response To External Host (sid 2001330)):
<img width="1732" height="97" alt="image" src="https://github.com/user-attachments/assets/d4c51aaa-a0b1-48e6-b9b8-18fcf941a46f" />
Suricata rule (ET INFO RDP - Response To External Host (sid 2001330)):
<img width="1045" height="364" alt="image" src="https://github.com/user-attachments/assets/78ebe2b1-f494-4f95-bca5-ade337c13079" />


Triggered custom 100100 rule from Wazuh in local_rules.xml:
<img width="1625" height="346" alt="image" src="https://github.com/user-attachments/assets/282f2493-e6d6-45d1-a19e-d4f64304f129" />


Wazuh alerts:
<img width="2523" height="912" alt="image" src="https://github.com/user-attachments/assets/f4284e06-a108-4da5-b50f-b78cce345e3b" />
<img width="2520" height="830" alt="image" src="https://github.com/user-attachments/assets/61e0023d-33d4-4bf2-9b77-fcb7f330f017" />
<img width="2523" height="831" alt="image" src="https://github.com/user-attachments/assets/1cedc1de-1822-412b-b71c-c0da3479336a" />
<img width="2520" height="826" alt="image" src="https://github.com/user-attachments/assets/be96aee4-7499-4928-a203-f203a8e8908c" />



