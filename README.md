# General #
All reversing documented here is currently based on the client beta 0.9.
### Sb\_client.exe arguments ###
--show\_console: show the console displaying the log (you can also take control of the console and use some functions/load packages)  
--packet_log: enable logging of the packets sended et received  
--world universeID: bypass universe selection and ask for the universe with ID universeID  
-- uc\_debugger: (check syntax) attach unreal engine debugger to the process. Sadly scripts files are not reachable.  
--unreal\_log: seems to do nothing. Test

### Encoding ###
TCoS client encodes its strings in UTF16-LE.

### Connect to custom login server ###
In etc/client/sb\_client.rc, modifie the line with
**client/net/login_addr** with desired IP and port.

# Packet Acquisition: approximative flow #
*Note*: Beginning of guessing part based on observations of the dll assembly code and debug step by step.
d\_mmos packet acquisition loop detect new message.
According to the ID , calls a method in SB\_client.exe.
*end of guessing*  
The methods called in SB\_client makes the work of calling the ReadMessage<struct MessageType> and the "onBlablabla" callback related to it.  
So by studying these methods, we should be able to understand the date wanted in the packet, and maybe even the *struct MessageType* layout.

# Network #
## d\_mmos.dll ##
Contains low level functions for sending packets.  
Interesting places: 
 
-  d\_message class with read (void\* dataOut, uint dataSize) and write (const void\* dataIn, uint dataSize)  
-  d\_address class: represent IP+Port   

### d\_message class ###
#### d\_message::read (void\* dataOut, uint dataSize) ####
**void\* dataOut** is a pointer to a memory location where the value read will be stored. Sometimes this memory is in stack, sometime in heap. It totally depends of the caller. For instance if the client tries to read a string, dataOut will be a pointer to the std::string::data () (or equivalent).

**uint dataSize** is the size in bytes of the value you want to read. For instance if
you want to read a DWORD (double word = 4 bytes) its value will be 4.

The d\_message object keeps track of the data you've read in it. So each time you call its *read* method, it updates an internal value with the size of data already read (it simply adds *dataSize* to its current value). This is also an offset allowing it to know *where to begin the reading* of the data in the message.
Each time you call read, you actually ask **to read the next data** in the d\_message object. If there is no more data, or not enough (there is still 2 bytes and you ask to read 4 bytes), the method will raise an exception.

#### d\_message::write (const void\* dataIn, uint dataSize) ####
Work the same way than *read* but write **dataSize** bytes of data pointed by **dataIn** inside the d\_message. Same mechanism checking that not too much data is written.

## SBPacket.dll ##
Contains packet high level stuff. Use templates a lot, which means that the window 'Names' in IDA won't show you all the methods because the same actual method can be exported with multiples names (not apparing in 'Names' but only in 'Exports'). So, look in 'Exports' instead.

The interesting classes are d\_packet<struct PACKET> and d\_serializable<struct PACKET> where 'struct PACKET' is the template parameter.
Interesting methods in d\_packet:  

 - GetID ()  
 - GetName () (but actually the name is exactly the same as the template parameter name)

Interesting methods in d\_message:

 - ReadMessage (const struct d\_message&)  
 - WriteMessage (struct d\_message&)  

**/!\\ /!\\ /!\\ ==>WriteMessage and ReadMessage are very interesting to study for each message because it's there you can understand what is put inside the packet.<== /!\\ /!\\ /!\\**

This is how I try to reverse packets, by studying these two methods and their context (who calls them, what structure do they fill etc).

To find quickly which message has a specific ID, in OllyDBG use "find command" and type **move ax, ID** where ID is a short number. It will point you the good GetID () method. The template parameter gives you the message name.

*Note*: packet's names follow a simple convention. They have a prefix specifying if its a packet sended by the client to the server or by the server to the client.  
**C2L**: Client to Login server  
**L2C**: Login server to Client  
**C2S**: Client to Server  
**S2C**: Server to Client  
**S2R**: I think for server broadcast  

There are other prefix but currently they are not involved in the phases I study.


## Packets layout ##
Every packet I have seen in the login process (login+ select universe) have this header: 


    struct PacketHeader (32 bits)
    {
      WORD PacketID;  
      WORD PacketSize;
    }  


If the PacketSize attribute does not match the actual packet size, the client will not trigger the ReadMessage or will trigger an error. If the client wants to read more data than you give to him, it will sayin the logs: "**PACKET\_NAME READ BUFFER OVERFLOW**".
 
Note: For what I've seen so far, the client does not check the size in its read algorithm to verify if its is consistent with the type of packet you send.  
 The client will just try to read everything it has to read and if you send not enough data, triggers the **BUFFER OVERFLOW** error seen above. This behavior is due to the d\_mmos.dll::d_message::read (const void* outData, uint dataSize) method which check the message size everytime you ask it to read a value in the packet. If the size alreday read + the size of the data you ask to read is greater than the message size: error.

*Note*: The following sizes does not take the packet's header into account, because the size contained in the packet's header do the same.

    
    struct CONNECT (4 bytes) 
    {
      struct PacketHeader header;
      DWORD unknownDword; //statuscode?
    };
    
    struct DISCONNECT (various size) 
    {
      struct PacketHeader header;
      DWORD unknownDword; //statuscode?
      DWORD reasonNumChars;//num chars in the reason string
      char [reasonNumChars*2] reason;//in UTF16-LE
    };
    
    struct C2L_USER_LOGIN (various size) 
    {
      struct PacketHeader header;
      DWORD clientVersion;//svn build version
      DWORD loginNumCharacters;
      BYTE[loginNumCharacters*2] loginString;//in UTF-16LE
      DWORD passwordNumCharacters;
      BYTE[passwordNumCharacters * 2] passwordString;//in UTF16-LE
    };
    
    struct L2C_USER_LOGIN_ACK (2 DWORDS = 8 bytes) 
    {
      struct PacketHeader header;
      DWORD zeroDword;//unknown data, its value does nothing
      DWORD statusCode;
    };    


statusCode possible values:

 - 0: Login OK
 - 1: Wrong client version
 - 2: Bad login or password
 - 3: Bad login or password (yes, the same)
 - above 3: to investigate!        
    
###  

    struct C2L_QUERY_UNIVERSE_LIST (empty) 
    {
      struct PacketHeader header;
    };  
    
    struct Universe (various size)
    {
      DWORD universeID;
      DWORD universeNameNumChars;
      BYTE[universeNameNumChars*2] universeName; //UTF16-LE
      DWORD universeLanguageNumChars;
      BYTE[universeLanguageNumChars*2] universeLanguage; //UTF16-LE
      DWORD universeTypeNumChars;
      BYTE[universeTypeNumChars*2] universeType; //UTF16-LE
      DWORD universePopulationNumChars;
      BYTE[universePopulationNumChars*2] universePopulation; //UTF16-LE
    };
    
    struct L2C_QUERY_UNIVERSE_LIST_ACK (various size) 
    {
      struct PacketHeader header;
      DWORD unknownDWORD; // MUST BE 0 to work, integrity check?
      DWORD universesNumber; //number of universes available 
      struct Universe[universesNumber] universesList; // universesNumber times the attributes in the struct Universe
    };
    
    struct C2L_UNIVERSE_SELECTED (size = 1 DWORD) 
    {
      struct PacketHeader header;
      DWORD universeID; //id of the selected universe
    };
    
    struct L2C_UNIVERSE_SELECTED_ACK (size = 1 DWORD) 
    {
      struct PacketHeader header;
      DWORD unknownDword#1; //MUST BE 0
      DWORD unknownDword#2;
      DWORD universePackageNameNumChars;
      BYTE[universePackageNameNumChars*2] universePackageName;
      DWORD tkey;//to investigate
      DWORD gameServerIP;
      WORD gameServerPort;
    };


*Note*: universePackageName is the name of the file in *data/universe* with the *SBU* sufixe removed from its name (I am not speeking of the extension but the sufix), and without the extension.  
The *tkey* may be for "transport key" seems to be (not sure) a security allowing the game world to detect if a client did the login part before connecting to it. I think it works this way: the Login Server generate a *tkey* for  a given player when it tries to login, send this key to the game world (by storing it in a commune DB or whatever) and to the client through this packet. When the client log into the game world, it sends back the key, the game world check if the key is correct and allow connection or not.

    struct C2S_TRAVEL_CONNECT (size = 1 DWORD) 
    {
      struct PacketHeader header;
      DWORD tkey; //the tkey the login server sended in the L2C_UNIVERSE_SELECTED_ACK packet
    };
    
    struct S2C_WORLD_PRE_LOGIN (size = 2 DWORD) 
    {
      struct PacketHeader header;
      DWORD unknownDWord;//MUST BE 0
      DWORD worldId; //must be 1 for character selection
    };


*Note*: the *worldId* is the ID identifying the map to load. 1 is CharacterSelection.sbw, and then beginning to 100 (PT_Hawksmouth) are the others ID. To find new ids you can send in a loop this message with a lot of world ID and see in the logs which ones actually load a map.

    struct C2S_WORLD_PRE_LOGIN_ACK (size = 1 DWORD) 
    {
      struct PacketHeader header;
      DWORD unknownDWord;// = 0; to investigate
    };  

    struct S2C_CS_LOGIN (various size) 
    {
      struct PacketHeader header;
      DWORD unknownDWord;//num characters I think
      DWORD unknownDword;//must be 0 if previous one is 0
      //Character data stuff if characters to send
      //This packet has to be reversed but does not seem to be very complicated
      //after having reversed the character creation phase.
    };  

    struct C2S_CS_CREATE_CHARACTER (various size) 
    {
      struct PacketHeader header;
      DWORD lod0size;
      byte[lod0size] lod0;
      DWORD lod1size;
      byte[lod1size] lod1;
      DWORD lod2size;
      byte[lod2size] lod2;
      DWORD lod3size;
      byte[lod3size] lod3;
      DWORD charNameNumChars;
      BYTE[charNameNumChars*2] characterName;
      DWORD classID;
      DWORD fixedSkill1ID;//Hack/slash/shoot
      DWORD fixedSkill2ID;//Hack/slash/shoot
      DWORD fixedSkill3ID;//Hack/slash/shoot
      DWORD customSkill1ID;//choosen by player
      DWORD customSkill2ID;//choosen by player
      DWORD unknwownDword;//=41 it changes when the character has a shield (=43)
    };  

*Note*: Lod0 Lod1 Lod2 Lod3 are byte arrays containing appearance information.  
**WARNING**: I am not 100% sure of their layout, so it has to be verified.  

	/*LOD0 size = 13
	 * [00] = glove left color 1
	 * [01] = glove left color 2
	 * [02] = glove right color 1
	 * [03] = glove right color 2
	 * [04] = gauntlet left color 1
	 * [05] = gauntlet left color 2
	 * [06] = gauntlet right color 1
	 * [07] = gauntlet right color 2
	 * [08] = tattoo chest + left arm (power of 16)
	 * [09] = tattoo left arm + tatoo right arm
	 * [10] = unknown (reserved for hood?)
	 * [11] = unknown (reserved for hood?)
	 * [12] = voice id
	 */
	
	/*LOD1 size = 20
	 * [00] = Pants color 1
	 * [01] = pants colour 2 
	 * [02] = shooes color 1
	 * [03] = shooes color 2
	 * [04] = helmet color 1
	 * [05] = helmet color 2
	 * [06] = left shoulder color 1
	 * [07] = left shoulder color 2
	 * [08] = right shoulder color 1
	 * [09] = right shoulder color 2
	 * [10] = belt color 1
	 * [11] = belt color 2
	 * [12] = Thigh left color 1
	 * [13] = Thigh left color 2
	 * [14] = thigh right color 1
	 * [15] = thigh right color 2
	 * [16] = shin left color 1
	 * [17] = shin left color 2
	 * [18] = shin right color 1
	 * [19] = shin right color 2
	 */
	
	
	/*LOD2 size = 15
	 * [00] = glove left type
	 * [01] = glove right type + pants type
	 * [02] = shooes type
	 * [03] = helmet type
	 * [04] = shoulder left + right type part 1
	 * [05] = shoulder right type part 2 + gauntlet left part 1
	 * [06] = gauntlet left part 2 + gauntlet right
	 * [07] = belt type + thigh left type part 1
	 * [08] = Thigh left type part 2 + thigh right part 1
	 * [09] = thigh right part 2 + shin left
	 * [10] = shin right + melee weapon part 1
	 * [11] = melee weapon part 2
	 * [12] = ranged weapon part 1
	 * [13] = ranged weapon part 2
	 * [14] = unknown dword (reserved for hood?)
	 */
	
	/*LOD3 size = 10
	 * [0] = Race+Gender+body 
	 * 0 to 3 = skinny (human male, daevi male, human female, daevi female)
	 * 4 to 7 = athletic (idem)
	 * 8 to 11 = fat (idem)
	 * [01] = skin color1 + headTypeID
	 * [02] = skin color2 + hairType (power of 16)
	 * [03] = hair type part 2 + hair color 1
	 * [04] = hair color 2 + torso cloth
	 * [05] = torso cloth color 1
	 * [06] = torso cloth color 2
	 * [07] = armor chest type + armor chest color 1
	 * [08] = armor chest color 1 + armor chest color 2
	 * [09] = armor chest color 2
	 */
Next packet.  

    struct Item
    {
      DWORD
      DWORD
      BYTE
      DWORD x 4
      BYTE
      BYTE
      DWORD
    };

    struct S2C_CS_CREATE_CHARACTER_ACK (various size) 
    {
      struct PacketHeader header;
      DWORD status;//0 = OK
      DWORD characterId;
      BYTE  isCharacterDead;
      DWORD accountID;
      DWORD charNameNumChars;
      BYTE[charNameNumChars*2] characterName;
      DWORD x 5; //WIP
      QWORD characterAppareance; //Lod stuff layouted in a different way, WIP
      DWORD x 5; // WIP
      DWORD classId;
      DWORD fameLevel;//To be verified
      DWORD x 3;//WIP
      BYTE x 4; //WIP
      DWORD numItems;
      struct[numItems] Item;
    }; 

    struct C2L_CS_SELECT_CHARACTER
    {
      struct PacketHeader header;
      DWORD characterId;
    };

    struct S2C_CS_SELECT_CHARACTER_ACK
    {
      struct PacketHeader header;
      DWORD status;//mysterious 32 bits
    };

*Note*: The status in the selected ack packet is strange. Can't find what to put in it. It may be a checksum... The value '2' display a specific error messages but all the other values I have tried display the same generic error message.

## Protocol ##
Here is the dialog between client and server, don't know if it's very useful. For detailed info about the packets structure, see the section above.  

**Client** sends *CONNECT* **to Login Server** (status code?) 
 
Client sends *C2L\_USER\_LOGIN* (login + password)  
Login Server answers *L2C\_USER\_LOGIN\_ACK* (status code)

Client sends *C2L\_QUERY\_UNIVERSE\_LIST* (no data)   
Login Server answers *L2C\_QUERY\_UNIVERSE\_LIST\_ACK* (status code?, universes list with infos)

Client sends *C2L\_UNIVERSE\_SELECTED* (universe selected id)  
Login Server answers *L2C\_UNIVERSE\_SELECTED\_ACK* (universe package's name, universe (game server) IP + port, and tkey)

Client sends *CONNECT* to **Game Server** 
Beginning of the conversatiom between **Client** and **Game Server**     
Client sends *C2L\_TRAVEL\_CONNECT* to Game Server (with tkey)  
Client sends *DISCONNECT* to **Login Server** (disconnection reason "connect to game world")  
Conversation between **Client** and **Login Server** is now over.

**Game Server** sends *S2C\_WORLD\_PRE\_LOGIN* (world setup id, must be 1 to load character selection screen, then world ID begin to 100 with PT_Hawksmouth etc)  
Client answers *C2S\_WORLD\_PRE\_LOGIN\_ACK*

Game Server sends *S2C\_CS\_LOGIN* (sends two DWORDS with '0' as value to load the **character creation screen**, but here you can also sends character data to display already existing characters) *Note: I think here we can cheat and send directly a world login, but as it is not obvious what to put in it, looking through the data in the character selection/creation phase should help a lot*  

After validating a character, client sends *C2S\_CS\_CREATE\_CHARACTER* (lots of data)  
Server answers *S2C\_CS\_CREATE\_CHARACTER\_ACK*

After clicking on "Enter World", Client sends *C2S\_CS\_SELECT\_CHARACTER* (character id)  
Server answers *S2C\_CS\_SELECT\_CHARACTER\_ACK* (**MYSTERIOUS DWORD**)

## List of Packets ID ##

CONNECT = 0xFFFD;  
DISCONNECT = 0xFFFE;
  
###Client (login phase)
C2L\_USER\_LOGIN = 0;  
C2L\_QUERY\_UNIVERSE\_LIST = 2;  
C2L\_UNIVERSE\_SELECTED = 4;  

### Login Server (login phase)
L2C\_USER\_LOGIN\_ACK = 1;     
L2C\_QUERY\_UNIVERSE\_LIST\_ACK = 3;  
L2C\_UNIVERSE\_SELECTED\_ACK = 5;

### Client (game world phase)
C2S\_TRAVEL\_CONNECT = 0;  
C2S\_WORLD\_PRE\_LOGIN\_ACK = 2;  
C2S\_WORLD\_LOGIN\_ACK = 4;  
C2S\_WORLD\_LOGOUT = 5;  
C2S\_TRAVEL\_WORLD = 7;  
C2S\_CS\_CREATE\_CHARACTER = 0x1C;//28  
C2S\_CS\_SELECT\_CHARACTER = 0x20;//32  

### Game Server (game world phase)
S2C\_WORLD\_PRE\_LOGIN = 1;  
S2C\_WORLD\_LOGIN = 3;
S2C\_WORLD\_LOGOUT\_ACK = 6;  
S2C\_TRAVEL\_WORLD\_ACK = 8;  
S2C\_CS\_LOGIN = 27;  
S2C\_CS\_CREATE\_CHARACTER\_ACK = 0x1D;//29  
S2C\_CS\_SELECT\_CHARACTER\_ACK = 0x21;//33  
