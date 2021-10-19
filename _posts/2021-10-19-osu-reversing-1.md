---
layout: post
title: Reverse Engineering osu! (part 1)
---

I've always been interested in custom osu! server for some reason. It's a bit exciting to play on something different with a smaller community.  Less than a couple of years back I got interested in making my own but I failed numerous times until lately when I decided to go back in time and make a server for the 2012 client instead. Why? Because back then it was much simpler, not even any HTTPS. This also means that

**This only applies to the 2012 osu! infrastructure, modern osu! is consideribly different which means this probably won't help you defeat the security and cheat or whatever.**

You can find my custom server here: <a href="https://osu.nobbele.dev" target="_blank">osu!classic</a>

With that out of the way, let's start with some terminology and explanations.

## osu!Bancho vs osu!Web

The distinction is important to understanding the architecture.

### osu!Bancho

The bancho bot handles the multiplayer part such as the chat, player listing, spectating and logging in.

Used to be located at 50.23.74.93:13382 but is now cho.ppy.sh in modern osu! (or rather anything after 2013).
osu!Bancho operates on TCP sockets, no HTTP here.

### osu!Web

This one is split into multiple parts, located at different domains. All are communicated with exclusively over HTTP.

#### a.ppy.sh

This is where the user avatar, also known as their profile picture, is stored.

| Route      | Explanation                            |
| ---------- | -------------------------------------- |
| /{user_id} | Gets the avatar for the specified user |

#### b.ppy.sh

This is where beatmap related assets are stored.

| Route                 | Explanation                                                 |
| --------------------- | ----------------------------------------------------------- |
| /thumb/{set_id}.jpg   | Gets the thumbnail image for the specified beatmap set      |
| /preview/{set_id}.mp3 | Gets the preview audio track for the specified beatmap set. |

#### s.ppy.sh

This stores miscellaneous assets. The 's' stands for static.


| Route                              | Explanation                                    |
| ---------------------------------- | ---------------------------------------------- |
| /images/achievements/{achievement} | Gives the image for the specified achievement. |

#### osu.ppy.sh

This is where the rest of the APIs and frontend is hosted. I won't list any endpoints here as they require multiple sections to explain. I will have this in another part.

## Connecting to osu!bancho

<sub>
Note: The client code samples have been pulled from my private osu! client decompilation repository.</sub>

Now we can finally get to the actual code and communication. In the client, after clicking the login button, osu! tries to open a connection to 50.23.74.93:13382 which is osu!bancho (Taiwan and Mainland China use 223.100.98.69:13381 though it seems).

```csharp
IPEndPoint ipendPoint_ = new IPEndPoint(1565136690L, 13382);
tcpClient = ClientHelper.Connect(ipendPoint_, 10000);
```




It then instantly begins to write the login data line by line, flushing each time, in the following format, in plain text.

| Data                                                   | Example                                           |
| ------------------------------------------------------ | ------------------------------------------------- |
| {Username}                                             | nobbele                                           |
| {Password MD5}                                         | 967ef2eb34634ba418db94dab610ba6f                  |
| {BuildName}\|{UTC Offset}\|{DisplayCity}\|{ClientHash} | b20121203\|2\|1\|75a97a5cf087c3dce41479536ebd2466 |

```csharp
GameBase.BanchoStatus.Text = "Logging in...";
writer.AutoFlush = true;
writer.WriteLine(ConfigManager.username);
writer.Flush();
writer.WriteLine(ConfigManager.password);
writer.Flush();
writer.WriteLine("{0}|{1}|{2}|{3}", General.BuildName(), TimeZone.CurrentTimeZone.GetUtcOffset(DateTime.Now).Hours, ConfigManager.DisplayCityLocation ? "1" : "0", GameBase.ClientHash);
writer.Flush();
```

After sending this the game will say "Logging in..." at the bottom of the screen.
The client will then wait for bancho to send a LoginReply, which is a good time to talk about how packets work.

### Bancho packets

Packets are the data bundles sent over the network between the client and server. They start with a header and then the data used when handling the packet.

#### Data Types

I will be using rust type notation here where the prefix i = Signed and u = Unsigned followed by the number of bits. 

u8 = Unsigned Byte. 

u16 = Unsigned 16-bit Integer.

i32 = Signed 32-bit Integer.

##### .NET Strings

As many of you probably know, osu! is written in C# which means it will be using the .NET standard library. This contains the BinaryWriter and BinaryReader that is frequently used here, these classes allows you to automatically convert your data into their binary data representation. Unfortunately the string implementation unecessarily complicated so it needs it's own explanation.

The first part will be the length of the string but, this length is encoded in a variable width data type, very similar to UTF-8. This means the length will be encoded as either 1, 2, 3 or 4 bytes. The way this is handled is that you read one byte at a time, if the MSB (Most Significant Byte / 8th bit) is 1 then that means there's another byte, if it's 0 then this is the end of the length data. The rest of the 7 bits of the byte are added to the length variable from the LSB / from the right. Once the length has been properly decoded, that many bytes will be read to form the string data.

Example of strings.

```c
00000110 01001000 01100101 01101100 01101100 01101111
(5 end)  ('H')    ('e')    ('l')    ('l')    ('o')


10000110 10000111 ...
(5 cont) (6 end)  ...
This will become 00001100000111 = 775
```



##### osu! Strings

Luckily this is simpler. osu! prepends a byte before the .NET string to signify whether it's null or not.

#### Packet Format

The format of the packet header is explained in the following table. 

| Data Type | Usage                     |
| --------- | ------------------------- |
| u16       | Type of packet to read    |
| u8        | Unknown                   |
| u32       | Length of the packet data |

### Login Transaction

Bancho will send the client a LoginReply packet which looks like this.

| Data Type | Usage       |
| --------- | ----------- |
| i32       | The user id |

Simple enough but why is it an i32 and not u32? Because that's how the game reads user IDs normally but also because in the case of LoginReply, it's used to return errors about the login. Anything not listed will be treated as sucessful. The various login error states are as follows.

| User ID | Meaning                                |
| ------- | -------------------------------------- |
| -1      | Incorrect username or password         |
| -2      | Client's osu! version is too old       |
| -3      | User is banned                         |
| -4      | Account is not yet activated           |
| -5      | Server-side error                      |
| -6      | Using test build without osu!supporter |

After successful login, the game will now say "Retreiving data..." in the bottom of the screen.

The client now expects a ChannelJoinSuccess packet which looks this.

| Data Type | Usage        |
| --------- | ------------ |
| NetString | Channel name |

The first ChannelJoinSuccess will be used to print the "Welcome to osu!Bancho, nobbele!", etc text.

Now the client will have a proper connection to the server but you won't be able to do much of anything without implementing a few more packets. That will be for part 2.

## Afterword

I hope you found this blog post somewhat interesting. I am very new to writing this blog thing but I wanted to share my findings with everyone and I also needed some content for my website. If you find any errors or misinformation please contact me anywhere. Thank you for reading!