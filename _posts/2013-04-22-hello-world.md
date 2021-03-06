---
layout: post
title: "CODE"
description: "저희 팀코드의 변천사입니다."
categories: Coding
tags: [random, jekyll]
redirect_from:
  - /2017/12/17/
---
저는 개발 초기부터 코드를 개발하는데 주력했습니다. 라즈베리파이 코딩을 익힌 후 로컬 내에서 음악을 재생하는 코드를 개발했습니다.

이 코드는 초기에 로컬용 코드 입니다.

```python
import smbus
import time
import vlc
import os
import RPi.GPIO as GPIO


# Define some device parameters
I2C_ADDR  = 0x27 # I2C device address
LCD_WIDTH = 16   # Maximum characters per line

# Define some device constants
LCD_CHR = 1 # Mode - Sending data
LCD_CMD = 0 # Mode - Sending command

LCD_LINE_1 = 0x80 # LCD RAM address for the 1st line
LCD_LINE_2 = 0xC0 # LCD RAM address for the 2nd line
LCD_LINE_3 = 0x94 # LCD RAM address for the 3rd line
LCD_LINE_4 = 0xD4 # LCD RAM address for the 4th line

LCD_BACKLIGHT  = 0x08  # On
#LCD_BACKLIGHT = 0x00  # Off

ENABLE = 0b00000100 # Enable bit

# Timing constants
E_PULSE = 0.0005
E_DELAY = 0.0005

#Open I2C interface
#bus = smbus.SMBus(0)  # Rev 1 Pi uses 0
bus = smbus.SMBus(1) # Rev 2 Pi uses 1

def lcd_init():
  # Initialise display
  lcd_byte(0x33,LCD_CMD) # 110011 Initialise
  lcd_byte(0x32,LCD_CMD) # 110010 Initialise
  lcd_byte(0x06,LCD_CMD) # 000110 Cursor move direction
  lcd_byte(0x0C,LCD_CMD) # 001100 Display On,Cursor Off, Blink Off 
  lcd_byte(0x28,LCD_CMD) # 101000 Data length, number of lines, font size
  lcd_byte(0x01,LCD_CMD) # 000001 Clear display
  time.sleep(E_DELAY)

def lcd_byte(bits, mode):
  # Send byte to data pins
  # bits = the data
  # mode = 1 for data
  #        0 for command

  bits_high = mode | (bits & 0xF0) | LCD_BACKLIGHT
  bits_low = mode | ((bits<<4) & 0xF0) | LCD_BACKLIGHT

  # High bits
  bus.write_byte(I2C_ADDR, bits_high)
  lcd_toggle_enable(bits_high)

  # Low bits
  bus.write_byte(I2C_ADDR, bits_low)
  lcd_toggle_enable(bits_low)

def lcd_toggle_enable(bits):
  # Toggle enable
  time.sleep(E_DELAY)
  bus.write_byte(I2C_ADDR, (bits | ENABLE))
  time.sleep(E_PULSE)
  bus.write_byte(I2C_ADDR,(bits & ~ENABLE))
  time.sleep(E_DELAY)

def lcd_string(message,line):
  # Send string to display

  message = message.rjust(LCD_WIDTH," ")

  lcd_byte(line, LCD_CMD)

  for i in range(LCD_WIDTH):
    lcd_byte(ord(message[i]),LCD_CHR)

def main():
  # Main program block

  # Initialise display
  lcd_init()
  GPIO.setmode(GPIO.BCM)

  GPIO.setup(20, GPIO.IN)
  GPIO.setup(21, GPIO.IN)
  GPIO.setup(23, GPIO.IN)
  GPIO.setup(24, GPIO.IN)


  path = "/home/pi/Music/"

  mlist = os.listdir(path)

  instance = vlc.Instance()

  player=instance.media_player_new()

  sunse=0
  #txt STORAGE

  #txt file save

  media=instance.media_new("/home/pi/Music/"+mlist[sunse])

  player.set_media(media)

  player.play()

  time.sleep(1)
     
  a = player.get_state()
  k=0
  while True:
      if(k==12):
          k=0
      a = player.get_state()
      k=k+1
      time.sleep(0.2)
      
      lcd_string(mlist[sunse]+ str(" ")*k,LCD_LINE_1)
      lcd_string(str(a)+ str(" ")*k,LCD_LINE_2)
      if a== 6:
          sunse=sunse+1
          if sunse==len(mlist):
              sunse=0
          media=instance.media_new("/home/pi/Music/"+mlist[sunse])
          player.set_media(media)
          lcd_string(mlist[sunse]+ str(" ")*k,LCD_LINE_1)
          lcd_string(str(a)+ str(" ")*k,LCD_LINE_2)
          player.play()
          
          time.sleep(1)
          k=k+1
          
           
      if a == 5:
          break
            
      if GPIO.input(20) == 0 :
        #back
         k=0
         player.stop()
         sunse=sunse-1
         if sunse==-1:
             sunse=len(mlist)-1
         media=instance.media_new("/home/pi/Music/"+mlist[sunse])

         player.set_media(media)

         player.play()
         lcd_string(mlist[sunse] + str(" ")*k ,LCD_LINE_1)
         lcd_string(str(a) + str(" ")*k ,LCD_LINE_2)
            
                 
         a = player.get_state()
         time.sleep(1)
         k=k+1
      if GPIO.input(21) == 0 :
          k=0
          #go
          player.stop()
          sunse=sunse+1
          if sunse==len(mlist):
              sunse=0
          media=instance.media_new("/home/pi/Music/"+mlist[sunse])
          player.set_media(media)
          player.play()
          lcd_string(mlist[sunse] + str(" ")*k,LCD_LINE_1)
          lcd_string(str(a) + str(" ")*k ,LCD_LINE_2)
          
          time.sleep(1)
          k=k+1

      if GPIO.input(23) == 0 :
          k=0
          #pause
          player.pause()
          a = player.get_state()
          lcd_string(mlist[sunse] + str(" ")*k ,LCD_LINE_1)
          lcd_string(str(a) + str(" ")*k ,LCD_LINE_2)
          
          time.sleep(1)
          k=k+1
          
      if GPIO.input(24) == 0 :
          #shutdown
          a = player.get_state()
          if a == 3:
              player.stop()



  

if __name__ == '__main__':

  try:
    main()
  except KeyboardInterrupt:
    pass
  finally:
    lcd_byte(0x01, LCD_CMD)
```


그리고 이후 서버 구축에 성공한후 서버에서 기기로 음악을 다운받아 구동하는 코드입니다.


```python
#!/usr/bin/env python

import smbus
import time
import RPi.GPIO as GPIO
import os
import vlc
from ftplib import FTP
import random
comlist=os.listdir("/home/pi/Music")
ftp = FTP("192.168.0.107","root","raspberry")

def down():
    i=0
    newone=False
    #ftp.retrlines("LIST")
     
    ftp.cwd("/music2")
     
    listing=[]
    ftplist=[]
     
    ftp.retrlines("LIST",listing.append)

    print(listing)
     
    while i<len(listing):
        word = listing[i].split(None,8)
        print(word)
        filename=word[-1].lstrip()
        print(filename)
        ftplist.append(filename)
        i+=1
    return ftplist

# Define some device parameters
I2C_ADDR  = 0x27 # I2C device address
LCD_WIDTH = 16   # Maximum characters per line

# Define some device constants
LCD_CHR = 1 # Mode - Sending data
LCD_CMD = 0 # Mode - Sending command

LCD_LINE_1 = 0x80 # LCD RAM address for the 1st line
LCD_LINE_2 = 0xC0 # LCD RAM address for the 2nd line
LCD_LINE_3 = 0x94 # LCD RAM address for the 3rd line
LCD_LINE_4 = 0xD4 # LCD RAM address for the 4th line

LCD_BACKLIGHT  = 0x08  # On
#LCD_BACKLIGHT = 0x00  # Off

ENABLE = 0b00000100 # Enable bit

# Timing constants
E_PULSE = 0.0005
E_DELAY = 0.0005

#Open I2C interface
#bus = smbus.SMBus(0)  # Rev 1 Pi uses 0
bus = smbus.SMBus(1) # Rev 2 Pi uses 1

def lcd_init():
  # Initialise display
  lcd_byte(0x33,LCD_CMD) # 110011 Initialise
  lcd_byte(0x32,LCD_CMD) # 110010 Initialise
  lcd_byte(0x06,LCD_CMD) # 000110 Cursor move direction
  lcd_byte(0x0C,LCD_CMD) # 001100 Display On,Cursor Off, Blink Off 
  lcd_byte(0x28,LCD_CMD) # 101000 Data length, number of lines, font size
  lcd_byte(0x01,LCD_CMD) # 000001 Clear display
  time.sleep(E_DELAY)

def lcd_byte(bits, mode):
  # Send byte to data pins
  # bits = the data
  # mode = 1 for data
  #        0 for command

  bits_high = mode | (bits & 0xF0) | LCD_BACKLIGHT
  bits_low = mode | ((bits<<4) & 0xF0) | LCD_BACKLIGHT

  # High bits
  bus.write_byte(I2C_ADDR, bits_high)
  lcd_toggle_enable(bits_high)

  # Low bits
  bus.write_byte(I2C_ADDR, bits_low)
  lcd_toggle_enable(bits_low)

def lcd_toggle_enable(bits):
  # Toggle enable
  time.sleep(E_DELAY)
  bus.write_byte(I2C_ADDR, (bits | ENABLE))
  time.sleep(E_PULSE)
  bus.write_byte(I2C_ADDR,(bits & ~ENABLE))
  time.sleep(E_DELAY)

def lcd_string(message,line):
  # Send string to display
  

  message = message.rjust(LCD_WIDTH," ")
  

  lcd_byte(line, LCD_CMD)

  for i in range(LCD_WIDTH):
    lcd_byte(ord(message[i]),LCD_CHR)

def main():
  index = open('index2.txt','r')
  indexcontent = int(index.read())
  index.close()
  
  # Main program block
  downloading=False
  ftplist=down()
  print(ftplist)

  # Initialise display
  lcd_init()
  GPIO.setmode(GPIO.BCM)

  GPIO.setup(20, GPIO.IN)
  GPIO.setup(21, GPIO.IN)
  GPIO.setup(23, GPIO.IN)
  GPIO.setup(24, GPIO.IN)
  GPIO.setup(16, GPIO.IN)


  path = "ftp://192.168.0.107/"

  instance = vlc.Instance()

  player=instance.media_player_new()

  sunse=indexcontent
  #txt STORAGE

  #txt file save
 
  
  media=instance.media_new("ftp://192.168.0.107/music2/"+ftplist[sunse])

  player.set_media(media)

  player.play()

  time.sleep(1)
     
  a = player.get_state()
  k=0
  while True:
      
      if(k==12):
          k=0
      a = player.get_state()
      k=k+1
      time.sleep(0.5)
      
      lcd_string(ftplist[sunse]+ str(" ")*k,LCD_LINE_1)
      lcd_string(str(a)+ str(" ")*k,LCD_LINE_2)
      
      if (a==3):
          if GPIO.input(16) == 0 :
              sunse=random.randint(0,len(ftplist)-1)
              media=instance.media_new("ftp://192.168.0.107/music2/"+ftplist[sunse])
              player.set_media(media)
              
              player.play()
             
              
      if a== 6:
          sunse=sunse+1
          if sunse==len(ftplist):
              sunse=0
          media=instance.media_new("ftp://192.168.0.107/music2/"+ftplist[sunse])
          player.set_media(media)
          
          player.play()
          k=k+1 
      if a == 5:
          break
            
      if GPIO.input(20) == 0 :
        #back
         k=0
         player.stop()
         sunse=sunse-1
         if sunse==-1:
             sunse=len(ftplist)-1
         media=instance.media_new("ftp://192.168.0.107/music2/"+ftplist[sunse])

         player.set_media(media)

         player.play()
            
                 
         a = player.get_state()
         
         k=k+1
      if GPIO.input(21) == 0 :
          k=0
          #go
          player.stop()
          sunse=sunse+1
          if sunse==len(ftplist):
              sunse=0
          media=instance.media_new("ftp://192.168.0.107/music2/"+ftplist[sunse])
          player.set_media(media)
          player.play()

          k=k+1

      if GPIO.input(23) == 0 :
          k=0
          #pause
          player.pause()
          a = player.get_state()
          
          
          k=k+1
          
      if GPIO.input(24) == 0 :
          #shutdown
          a = player.get_state()
          if a == 3:
              player.stop()
  os.remove("/home/pi/Desktop/index2.txt")
  indexnum=str(sunse)
  index = open('index2.txt','w')
  index.write(indexnum)
  index.close()

if __name__ == '__main__':

  try:
    main()
  except KeyboardInterrupt:
    pass
  finally:
    lcd_byte(0x01, LCD_CMD)
```

그리고 이후 중간 발표가 있었고 그후에 서버 스트리밍에 성공했고 최종발표까지 서버스트리밍과 채널구현까지 가능한 코드를 개발했습니다. 하지만 이전까지 개발했던 코드로 채멀구현을 하기에는 무리가있었고 다른 친구가 그 코드를 기반으로 재구성하여 최종코드를 개발했습니다.


```python
#!/usr/bin/env python
import smbus
import time
import RPi.GPIO as GPIO
import os
import random
import vlc
from ftplib import FTP

IP = "192.168.0.22"
ftp = FTP(IP, "root", "raspberry")
ftp.cwd("/storage")
PATH = "ftp://"+IP+"/"
instance = vlc.Instance()
player=instance.media_player_new()

def downlist():
    templist = []
    musiclist = []
    channellist = []
    musicfolderlist=[]
    folderlist=[]
    ftp.retrlines("LIST", templist.append)
    i=0

    while i<len(templist):
            word = templist[i].split(None,8)
            filename=word[-1].lstrip()
            musiclist.append(filename)
            i+=1
    i=0
    templist = []
    templist.append("Non Folder Music")

    while i<len(musiclist):
        musicname = musiclist[i]
        if(len(musicname)>4):
            if(musicname[-4]!="." and musicname[-5]!="."):
                channellist.append(musicname)
            else:
                templist.append(musicname)
        else:
            channellist.append(musicname)
        i+=1
    musicfile = templist
    folderlist.append(musicfile)
    print(channellist)
    templist=[]
    i=0
    j=0
    while i<(len(channellist)):
        foldername = channellist[i]
        ftp.cwd("/storage/"+foldername)
        ftp.retrlines("LIST", templist.append)
        musiclist=[]
        musiclist.append(foldername)
        while j<len(templist):
            word = templist[j].split(None,8)
            filename = word[-1].lstrip()
            musiclist.append(filename)
            j+=1
        i+=1

        musicfolderlist.append(musiclist)
    musicfolderlist.append(folderlist[-1])
    return musicfolderlist

serverlist = downlist()

#LCD start
# Define some device parameters
I2C_ADDR  = 0x27 # I2C device address
LCD_WIDTH = 16   # Maximum characters per line

# Define some device constants
LCD_CHR = 1 # Mode - Sending data
LCD_CMD = 0 # Mode - Sending command

LCD_LINE_1 = 0x80 # LCD RAM address for the 1st line
LCD_LINE_2 = 0xC0 # LCD RAM address for the 2nd line
LCD_LINE_3 = 0x94 # LCD RAM address for the 3rd line
LCD_LINE_4 = 0xD4 # LCD RAM address for the 4th line

LCD_BACKLIGHT  = 0x08  # On
#LCD_BACKLIGHT = 0x00  # Off

ENABLE = 0b00000100 # Enable bit

# Timing constants
E_PULSE = 0.0005
E_DELAY = 0.0005

#Open I2C interface
#bus = smbus.SMBus(0)  # Rev 1 Pi uses 0
bus = smbus.SMBus(1) # Rev 2 Pi uses 1

def lcd_init():
  # Initialise display
    lcd_byte(0x33,LCD_CMD) # 110011 Initialise
    lcd_byte(0x32,LCD_CMD) # 110010 Initialise
    lcd_byte(0x06,LCD_CMD) # 000110 Cursor move direction
    lcd_byte(0x0C,LCD_CMD) # 001100 Display On,Cursor Off, Blink Off 
    lcd_byte(0x28,LCD_CMD) # 101000 Data length, number of lines, font size
    lcd_byte(0x01,LCD_CMD) # 000001 Clear display
    time.sleep(E_DELAY)

def lcd_byte(bits, mode):
  # Send byte to data pins
  # bits = the data
  # mode = 1 for data
  #        0 for command

    bits_high = mode | (bits & 0xF0) | LCD_BACKLIGHT
    bits_low = mode | ((bits<<4) & 0xF0) | LCD_BACKLIGHT

  # High bits
    bus.write_byte(I2C_ADDR, bits_high)
    lcd_toggle_enable(bits_high)

  # Low bits
    bus.write_byte(I2C_ADDR, bits_low)
    lcd_toggle_enable(bits_low)

def lcd_toggle_enable(bits):
    # Toggle enable
    time.sleep(E_DELAY)
    bus.write_byte(I2C_ADDR, (bits | ENABLE))
    time.sleep(E_PULSE)
    bus.write_byte(I2C_ADDR,(bits & ~ENABLE))
    time.sleep(E_DELAY)

def lcd_string(message,line):
  # Send string to display
  

    message = message.ljust(LCD_WIDTH," ")
  

    lcd_byte(line, LCD_CMD)

    for i in range(LCD_WIDTH):
        lcd_byte(ord(message[i]),LCD_CHR)

def sound(number):
    player.audio_set_volume(number)

def starter():
    index = open('index4.txt','r')
    channelk = index.readline()
    channels = int (channelk)
    listnumk = index.readline()
    listnums = int (listnumk)
    title = index.readline()
    index.close()
    if(serverlist[channels][listnums] == title):
        return channels,listnums
        
    else:
        return 0,1

def sunsetype(listnum, playmode, button,ranlist):
    if(playmode % 3 == 0):
        if(button == 1):
            return listnum+1
        if(button == -1):
            return listnum-1

    if(playmode % 3 == 1):
        return random.randrange(1, len(ranlist)-1)
        

    if(playmode % 3 == 2):
        return listnum

def whattype(playmode):
    if(playmode % 3 == 0):
        return "ORDER"
    if(playmode % 3 == 1):
        return "RANDOM"
    if(playmode % 3 == 2):
        return "REPEAT"
    
def mediaplayer(channel, title):

    media=instance.media_new(PATH+"storage/"+str(channel)+"/"+title)
    player.set_media(media)
    player.play()

def LCDsetter(changetime,Amode,Bmode,channel,listnum,ik):
    title = serverlist[channel][listnum]
    if(changetime % 8 >= 7):
        lcd_string(title[ik*16:(ik+1)*16],LCD_LINE_1)
        changetime=0
        return True
    if(Amode != Bmode):
        lcd_string(str(Amode),LCD_LINE_2)
        Bmode = Amode
    else:
        lcd_string(str(Bmode),LCD_LINE_2)

def timeout(sec):
    mint = sec/60
    sec = sec%60
    strmin=""
    strsec=""
    if(mint<10):
        strmin = "0"+str(mint)
    else:
        strmin = mint
    if(sec<10):
        strsec = "0"+str(sec)
    else:
        strsec = sec
    return str(strmin)+":"+str(strsec)

def main():
    channelmode = True
    musicmode = False
    volumemode = False
    statemode = False
    i=0
    channel, listnum = starter()
    lcd_init()
    lcd_string("   Hello!",LCD_LINE_1)
    time.sleep(1)
    GPIO.setmode(GPIO.BCM)

    GPIO.setup(20, GPIO.IN)#after
    GPIO.setup(21, GPIO.IN)#Before
    GPIO.setup(23, GPIO.IN)#pause
    GPIO.setup(24, GPIO.IN)#turn off
    GPIO.setup(25, GPIO.IN)#mode change
    print(str(channel)+"   "+str(listnum))
    print(serverlist[channel][listnum]+"\n\n")

    mediaplayer(serverlist[channel][0],serverlist[channel][listnum])
    i = len(serverlist[channel][listnum])/16
    time.sleep(1)

    mode = 0
    playmode=0
    volumepower = 50
    changetime =0
    Amode=""
    Bmode=""
    ik=0
    times=0
    sec=0

    while True:
        times+=1
        if(ik==i+1):
            ik=0
        if(LCDsetter(changetime,Amode, Bmode,channel,listnum,ik)):
            ik+=1
        if(times%4==3 and player.get_state() == 3):
            sec+=1
        changetime += 1
        time.sleep(0.25)
            
        
        if(GPIO.input(24)==0):#shutdown
            os.remove("/home/pi/index4.txt")
            index = open('index4.txt','w')
            index.write(str(channel)+"\n"+str(listnum)+"\n"+str(serverlist[channel][listnum]))
            index.close()
            player.stop()
            lcd_string(str("Goodbye!"),LCD_LINE_1)
            time.sleep(2)
            lcd_string(str(" "),LCD_LINE_2)
            time.sleep(2)
            break

        if(mode%4 == 0):
            Amode = "Chennal "+serverlist[channel][0]
        if(mode%4 == 1):
            Amode = whattype(playmode)+" "+timeout(sec)
        if(mode%4 == 2):
            Amode = "Vol : "+str(volumepower)
        if(mode%4 == 3):
            Amode = whattype(playmode)+" setting"
            
        if(GPIO.input(25)==0):
            time.sleep(0.2)
            mode = mode+1
            if(mode%4 == 0):
                print("channelmode")
                channelmode = True
                musicmode = False
                volumemode = False
                statemode = False
                Amode = "channelmode"
                time.sleep(1)
            if(mode%4 == 1):
                print("musicmode")
                channelmode = False
                musicmode = True
                volumemode = False
                statemode = False
                Amode = "musicplay"
                time.sleep(1)
            if(mode%4 == 2):
                print("volumemode")
                channelmode = False
                musicmode = False
                volumemode = True
                statemode = False
                Amode = "volume"
                time.sleep(1)
            if(mode%4 == 3):
                print("statemode")
                channelmode = False
                musicmode = False
                volumemode = False
                statemode = True
                Amode = "playtype mode"
                time.sleep(1)
        if(channelmode):
            Amode = "Chennal "+serverlist[channel][0]
            if(GPIO.input(20)==0):
                time.sleep(0.2)
                channel+=1
                if(channel == len(serverlist)):
                    channel = 0
                Amode = "Chennal "+serverlist[channel][0]
            if(GPIO.input(21)==0):
                time.sleep(0.2)
                channel-=1
                if(channel == -1):
                    channel = len(serverlist)-1
                Amode = "Chennal "+serverlist[channel][0]

            if(GPIO.input(23)==0):
                time.sleep(0.2)
                player.stop()
                mediaplayer(serverlist[channel][0],serverlist[channel][1])
                sec=0
                i = len(serverlist[channel][listnum])/16
                mode+=1
                channelmode = False
                musicmode = True
                volumemode = False
                statemode = False

        if(player.get_state() == 6):
            listnum = sunsetype(listnum, playmode, 1,serverlist[channel])
            if(listnum == len(serverlist[channel])):
                listnum = 1
            player.stop()
            mediaplayer(serverlist[channel][0],serverlist[channel][listnum])
            i = len(serverlist[channel][listnum])/16
            sec=0
            Amode = whattype(playmode)+" "+timeout(sec)

        if(musicmode):
            Amode = whattype(playmode)+" "+timeout(sec)
            if(GPIO.input(20)==0):
                time.sleep(0.2)
                listnum = sunsetype(listnum, playmode, 1,serverlist[channel])
                if(listnum == len(serverlist[channel])):
                    listnum = 1
                player.stop()
                mediaplayer(serverlist[channel][0],serverlist[channel][listnum])
                i = len(serverlist[channel][listnum])/16
                sec=0
                Amode = whattype(playmode)+" "+timeout(sec)
            if(GPIO.input(21)==0):
                time.sleep(0.2)
                listnum = sunsetype(listnum, playmode, -1,serverlist[channel])
                if(listnum == 0):
                    listnum = len(serverlist[channel])-1
                player.stop()
                mediaplayer(serverlist[channel][0],serverlist[channel][listnum])
                sec=0
                i = len(serverlist[channel][listnum])/16
                Amode = whattype(playmode)+" "+timeout(sec)
            if(GPIO.input(23)==0):
                time.sleep(0.2)
                player.pause()
                Amode = whattype(playmode)+" "+timeout(sec)
                


        if(volumemode):
            Amode = "Vol : "+str(volumepower)
            if(GPIO.input(20)==0):
                time.sleep(0.2)
                volumepower += 10
                player.audio_set_volume(volumepower)
                Amode = "Vol : "+str(volumepower)
            if(GPIO.input(21)==0):
                time.sleep(0.2)
                volumepower -= 10
                player.audio_set_volume(volumepower)
                Amode = "Vol : "+str(volumepower)
            if(GPIO.input(23)==0):
                time.sleep(0.2)
                volumepower = 50
                player.audio_set_volume(volumepower)
                Amode = "Vol : "+str(volumepower)
                


        if(statemode):
            Amode = whattype(playmode)+" setting"
            if(GPIO.input(20)==0):
                Amode = whattype(playmode)+" setting"
                time.sleep(0.2)
                playmode+=1
                

            if(GPIO.input(21)==0):
                Amode = whattype(playmode)+" setting"
                time.sleep(0.2)
                playmode-=1
                if(playmode<0):
                    playmode = 5
            if(GPIO.input(23)==0):
                time.sleep(0.2)
                sec=0
                i = len(serverlist[channel][listnum])/16
                mode+=1
                channelmode = False
                musicmode = True
                volumemode = False
                statemode = False

if __name__ == '__main__':

  try:
    main()
  except KeyboardInterrupt:
    pass
  finally:
    lcd_byte(0x01, LCD_CMD)
```
여러가지 우여곡절이 있었지만 문제를 해결하고 개발해가는 과정을 느낄수 있었습니다.
