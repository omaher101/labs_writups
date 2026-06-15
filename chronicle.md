Stuff That I Learned: 
1. Fuzz More
2. command: git log to see commit history
3. scp tool 
4. .mozilla folder is IMPORTANT



found port 80(old) and 8081(new)

i can retrieve passwords by entering usernames but i doesn't work because of missing API Key

Fuzz showed gold .git on the old 80 port website

need to find username and API Key
![[Pasted image 20260614161729.png|557]]


seeing commits, one of them showed the hidden API Key
![[Pasted image 20260614161515.png|554]]

trying the username and API key i found, IT WORKED! it returned the username and password
![[Pasted image 20260614161935.png|572]]

i can also look for more usernames with ffuf
![[Pasted image 20260614162600.png]]

login page doesn't seem to work so we try SSH mayber?
![[Pasted image 20260614162208.png|549]]

using tommy username and password to login via SSH, it worked and i'm insied tommy's machine
![[Pasted image 20260614163310.png|538]]![[Pasted image 20260614163352.png|437]]

BOOM! The first flag, user.txt
![[Pasted image 20260614163447.png]]

found another user called carlJ and his folder contain a hidden .mozzila folder
![[Pasted image 20260614174857.png|550]]

.mozzila files can contain sensitive data and i will use a tool called firefox-decrypt and downloading the firefox folder inside the .mozilla on my local machine via scp tool

![[Pasted image 20260614175438.png]]


choosing option one will show that this isn't a valid profile 
![[Pasted image 20260614175706.png]]

option 2 is the right choice  but requires a password but how to fuzz this? Automation is the answer
![[Pasted image 20260614175555.png]]

made a script using python where when the tool runs and ask for a choice it choses option 2 automatically and when it asks for a password it uses a wordlist until it hits a bingo.
 
 Script:![[cracker.py]]
 ![[Pasted image 20260614180223.png]]

BOOM AGAIN! found the password
![[Pasted image 20260614180349.png|451]]

Let's try SSH carlJ using this password, Success!
![[Pasted image 20260614180519.png|609]]

Finally entered the mailing folder and found smail app
![[Pasted image 20260614180751.png|517]]

opening it and seeing the option we can see that it can be a buffer overflow vulnerability
![[Pasted image 20260614182542.png|479]]

using string we can get more information and know more about the program
![[Pasted image 20260614182721.png|528]]
it's a 64 bit program and it's clear that it has a setuid binary that can be exploited

let's craft an exploit for this program via ret2libc notes
![[Pasted image 20260614183018.png|532]]
