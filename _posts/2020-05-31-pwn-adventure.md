---
title: "Pwn Adventure 3: Walkthrough"
header:
  teaser: /assets/images/pwn-adventure/teaser.png
  image: /assets/images/pwn-adventure/pwnie-island.png
  caption: "[Pwn Adventure 3: Pwnie Island](https://www.pwnadventure.com/)"
toc: true
toc_sticky: true
tags:
  - reverse engineering
  - network protocols
  - windows
  - video game
  - MMORPG
---

# What is Pwn Adventure?

By discovering the videos from [LiveOverflow](https://www.youtube.com/channel/UClcE-kVhqyiHCcjYwcpfj9w/featured), I landed on his series of videos about "[Pwn Adventure 3: Pwnie Island](https://www.pwnadventure.com/)". This is a video game created by Vector 35 for the Ghost in the Shellcode CTF that took place in 2015. This MMORPG is intentionally vulnerable, replacing classical quests with exploits, featuring network communications tampering with the game server, and reverse engineering techniques against the game client.

Since video game is not a prevalent hacking category among the CTF competitions nor the existing training platforms, I was curious to see how many flags I would be able to find by playing this game myself.

The game client is available on Linux and Windows systems. I chose the Windows version for two reasons:
- Nowadays, PC video games are more common on Windows systems than on Linux systems;
- LiveOverflow leveraged the Linux game client for his walkthrough, and I wanted to adopt a different approach.

# Setup and tools

After installing the Windows game client, we will need to set up our own private game server, since the public ones do not exist anymore (and even if they did, this would be a nightmare to mess with the server at the same time as other hackers). In addition to that, we will need tools for the client and the server communication analysis.

## Private server

A simple way to set up a local private server is to use this [Docker installation](https://github.com/LiveOverflow/PwnAdventure3) by LiverOverflow. Optionally, this could be deployed on the cloud (e.g. Amazon EC2). One important remark here is that if the server process is interrupted, all world and player's progression will be lost. By following the instructions, you will need to adapt your Windows *hosts* file and the *server.ini* of the client.

## Binary Ninja

For the client analysis, I decided to use the reverse engineering tool by the same team that created this game: [Binary Ninja](https://binary.ninja/). Many other tools are available, like the excellent free tool [Ghidra](https://ghidra-sre.org/).

## Server proxy

In order to intercept and tamper the network communications to and from the server, the easiest solution is to set up a [proxy](/assets/files/PwnAdventure-proxy.zip) server between the client and the server. I adapted the Python code written by LiveOverflow from the same repository evoked above. Note that the code is far from optimised and still contains research comments from both LiveOverflow and myself.

# Client-side exploits

We will first explore the Windows client and its custom files: The game executable relies on a Dynamic-Link Library named *GameLogic.dll*, that is provided with a file named *GameLogic.pdb*.

![Windows client files](/assets/images/pwn-adventure/windows-files.png)

This is a Program DataBase file that contains debugging information about the GameLogic DLL. A PDB file is never supposed to be released and is only useful for the developers, but since we are in a context of a CTF, this will considerably speeds up our analysis as we will have symbol names associated with the functions.

![Functions without PDB](/assets/images/pwn-adventure/binary-ninja.png)

We just open the DLL file in Binary Ninja and import the PDB file in **Tools > PDB > Load**. Once the analysis is done, we now have function names on which we can perform research.

![Functions with PDB information](/assets/images/pwn-adventure/binary-ninja-pdb.png)

## Speed acceleration

Before starting any quest, it would be practical to spend less time exploring the island because we do not have any way to fly nor teleport. Since flying is not possible (at least not easily) and teleporting is limited by the Travel System (see below), an easy solution would be to increase our moving speed. By searching on *Player* in the symbols, we find a `Player::GetSprintMultiplier` function.

![Player search](/assets/images/pwn-adventure/sprint-search.png)

This funtion returns a certain number (an integer) that is equals to 1 077 936 128 (in decimal).

![Player::GetSprintMultiplier function](/assets/images/pwn-adventure/sprint-function.png)

We increase that number to 1 **9**77 936 128, and write it in hexadecimal (we must be careful to write it in Little Endian order where it is stored).

![Player::GetSprintMultiplier value](/assets/images/pwn-adventure/sprint-value.png)

This value allows us to greatly accelerate as long as we run, while we are still able to move quite normally on short distances.

![Player sprint increased](/assets/images/pwn-adventure/sprint.gif)

## Until the Cows Come Home

Flags on the Pwnie Island are often associated with quests, and this quest concerns missing cows that are nowhere to be found on the island.

![The missing cows quest](/assets/images/pwn-adventure/cow-quest.png)

*To be honest, I did not search the island in depth, I landed on the cows after I decided to start hacking the Fast travel system. This consists in several spots scattered around the island from which we can teleport once they have been discovered manually.*

![Fast travel system](/assets/images/pwn-adventure/fast-travel-system.png)

By searching on the library symbols, we find an explicit function: `Player::GetFastTravelDestinations`. This function contains all the destinations that we can find as a player. The code blocks are the same and replicated for all destinations, only two important changes (highlighted in red) occur each time:
- Two numbers that indicate the position of the destination in the UI presented to the player (these do not need to be changed and could be interpreted as 4 slots available);
- A small number;
- The destination name (pointer to the destination name string).

![Fast travel function](/assets/images/pwn-adventure/fast-travel-function.png)

The first obvious step would be to replace a destination by another one (by respecting the exact name and integer of the destination to insert), to observe that it is succesful. For example, we can add twice the same known destination.

![Fast travel hacked](/assets/images/pwn-adventure/fast-travel-hacked.png)

Going further, we wonder if some secret destination is available and hidden in the code. This seems to be the case, as we can observe *Cowabungalow* and *CowLevel* not far from the destination strings that we could already locate in the library.

![Fast travel secret destination](/assets/images/pwn-adventure/fast-travel-secret.png)

We can insert these two desintation names in the function to see if it is working. The only missing information is this small number associated with each destination. By comparing each value to the known destination, we understand that it corresponds to the length of the destination name (in hexadecimal). We can then deduce the value that we need to provide for our two secret destination attempts.

| Value		| Destination		|
| -----		| -----------		|
| 0x4    	| Town			|
| 0xf		| UnbearbaleWoods	|
| 0xd		| TailMountains		|
| 0x8		| GoldFarm		|
| **0xc**	| **Cowabungalow**	|
| **0x8**	| **CowLevel**		|

However, this will not work as we must first discover a destination before being able to teleport to it. So the next step consists in modifying the destinations displayed to the player. This seems to be verified in our function by the `FastTravelDestination::AddToListIfValid`, after each code block featuring the destinations discovered above.

![Fast travel destination validation](/assets/images/pwn-adventure/fast-travel-validation.png)

We guess that this function will either validate or invalidate the destination at some point, depending if we have discovered it or not. This means that if we invert the right condition, we could obtain the list of unknown destinations instead of the known destinations. After several tries (some conditions inverted have no effect while others crash the game), we find the relevant condition to invert:

![Fast travel validation inversion](/assets/images/pwn-adventure/fast-travel-inversion.png)

Patching the condition line from `je` to `jne` (inversion of the condition), we end up with unknown destination listed in the game, and we observe that **Cowabungalow** is a valid destination that can be used that way. By completing the quest on this secret island, we kill the Cow King and end up with a new legendary weapon (*Static link*) and our first flag.

![Cow level](/assets/images/pwn-adventure/cow-level.gif)

# Server-side exploits

For the next flags we will exploit the communications to and from the server. This is because it is not possible to perform all actions from the client alone, one example is the player position. Although it is possible to hack the player position where we want locally, it needs to be sent and validated by the server.

First we need to understand the communication protocol: by using our proxy or a tool like Wireshark, we observe that hexadecimal data are sent between the client and the server. Event though our understanding of the protocol implies some guesses, the options are limited: the data are represented in formats such as Char, String, Integer, Float, etc. and are probably ordered following a typical methodology like the [TLV](https://en.wikipedia.org/wiki/Type-length-value).

![Network protocol](/assets/images/pwn-adventure/communication-protocol.png)

Our proxy is able to print all or a part of the communications in hexadecimal, and the goal is to build functions to translate the different types of communications once we understand them. The methods consists in performing isolated actions in the game and see which communications result from these actions, e.g. move forward, on the right, jump, etc.

![Network communications](/assets/images/pwn-adventure/communication-unknown.png)

## Unbearable Revenge

The Unbearable Revenge is a quest where the player must survive against waves of attacking bears in a small zone around a chest for five minutes.

![Bear quest](/assets/images/pwn-adventure/bear-1-p1.gif)

Although the number of bears can be managed with the player weapons for a certain time, they become more and more numerous, after a while with *Angry Bears*, that are stronger bears more difficult to fight. Our first attempt is to try to *kite* the bears with the help of our speed hacking.

> Kiting is a method that consists of keeping enemies or other players at a close distance and running whenever the enemy comes near.

![Bear quest: 1st attempt](/assets/images/pwn-adventure/bear-1-p2.gif)
*Unbearable Revenge: 1st attempt*

Unfortunately, we forgot that bears could use AK Rifles to kill the player at distance if it is necessary - this is what is happening after 3min30s.

From there it seems necessary to hack the player position to somewhere safe, out of range from both the bear claws and their AK Rifles. By applying the technique above, we isolate the communications about the player movement: this concerns only data sent from the client to the server. First we need to understand the movement protocol composition. We deduce two useful parts: An ID that will identify the data as movement, and the coordinates of the player. The last two parts seem not relevant for us (where the camera look and empty data).

![Movement data](/assets/images/pwn-adventure/communication-movement.png)
*Movement data structure*

The part that we want to tamper with is the position. We know that in a 3D environment, 3 coordinates (X, Y, Z) are enough to define a position. From there, what remains to do is guess the number format. There are only a few options, since we have 12 bytes in total and we would need floating point numbers in this context. After some tries, we confirm that a translation using the float data type gives us the right coordinates.

![Data type of the movement data](/assets/images/pwn-adventure/communication-data-types.png)

Now that we understand how coordinates are defined, we can tamper with the communications and send our own coordinates in place of the game client. The proxy command *bear* will automate this, by sending the coordinates we specify for 5min. We must now decide which coordinates to send, by keeping in mind that we have to stay in the quest zone for 5 minutes. By observing the quest zone environment, we see a tree. We could now try to place the player high in that tree.

```python
cmd = raw_input('PwnProxy$ ')
    if cmd[:4] == 'quit' or cmd[:4] == 'exit':
        os._exit(0)
    elif cmd[:4] == 'bear':
	bear_quest = True
	timeout = time.time() + 60*5 # 5min timer
        # Hexadecimal movement data to send
	cmd = '6d76ae91e2c5214e7c4714362845d1f78d3a00000000'
	# Attempt 2: 6d76ae 33f9f6c56e397a471ff17945 d1f78d3a00000000
	# Attempt 3: 6d76ae 33f9f6c56e397a471f211945 d1f78d3a00000000
	# Attempt 4: 6d76ae ae91e2c5214e7c4714362845 d1f78d3a00000000
	while True:
	    #print cmd
    	    if time.time() > timeout:
                bear_quest = False
                break
            for server in game_servers:
                if server.running:
                    parser.SERVER_QUEUE.append(cmd.decode('hex'))
	    time.sleep(.010)
```

![Bear quest: 2nd attempt](/assets/images/pwn-adventure/bear-2.gif)
*Unbearable Revenge: 2nd attempt*

By the way, we can observe that even though our player is still on the ground in our game (the client), the bears see our character in the tree since this data is changed on the server side and not on the client side, which is a key aspect here. Unluckily, hiding high in the tree is not enough to be out of range of the weapons.

Since we are still in the zone when we "climb" in the tree, we could guess that the zone is delimited by a sphere around the chest. We could then fake our position beneath the ground and thus be physically protected againt all types of attacks.

![Bear quest: 3rd attempt](/assets/images/pwn-adventure/bear-3.gif)
*Unbearable Revenge: 3rd attempt*

But after a few seconds, we are out of the zone. How is that possible? Well, it must come from the server behavior: if a character is not standing on any floor or some physical object, it is considered as falling. Also, when a player is falling, the fall becomes faster with time, until the position refresh rate is not enough to compensate the fall speed, we are then considered out of the quest area.

![Bear quest: 3rd attempt: falling behavior](/assets/images/pwn-adventure/bear-3-falling.gif)

In our next attempt, we will keep the idea of a physical protection while remaining on the floor, combining the ideas from the second and the third attempt: we will hide inside the tree.

![Bear quest: 4th attempt](/assets/images/pwn-adventure/bear-4.gif)
*Unbearable Revenge: 4th attempt*

This last attempt is succesful, we can even observe that the bears close to the tree are disturbed, they constantly switch between going for our position and then going away because of the obstacle (the tree) encountered where they are supposed to reach our character. By opening the chest we obtain another flag.

## Egg Hunter

While exploring the island, it is possible to find a golden egg (like the one below in the Pirate zone) that starts a quest, asking the player to find all of them (10 in total).

![A golden egg](/assets/images/pwn-adventure/egg.png)

To solve this quest, we will focus on another network communication that occurs each time we connect to the game or we change island zones.

![Unknown data delivered by the server on connection](/assets/images/pwn-adventure/communication-unknown-start.png)

We will first try to see if any text can be read from the hexadecimal translation. It is the case, and the names suggest that information about world objects are transmitted to the client (including the eggs).

![A preview of the locations](/assets/images/pwn-adventure/locations-preview.png)

Analysing the data format for each of these objects and leveraging the format we already understood for the movement data, here is the structure that we are able to deduce:

![Object location data format](/assets/images/pwn-adventure/object-data.png)
*Object location data structure*

From there, we have two options:
- Go get all the eggs manually;
- Be lazy and find a solution to avoid searching for the eggs scattered around the island.

By going for the first option, we will quickly realise that the second option will become necessary, as some eggs cannot be reached without getting around the physical limits of the game. Some eggs will require at least a mechanism that allows us to fly or teleport.

The good news is that we are already able to teleport where we want on the map, even to places we cannot reach normally (cfr. the Unbearable Revenge, where we were able to get inside a tree). However, once at the correct location we still need to trigger the "Pick up" action to collect the egg. So, let's see the data structure of a pick-up action. We can intercept such a communication by picking up a first egg that we can reach normally:

![Pick up data format](/assets/images/pwn-adventure/pickup-data.png)
*Pick-up action data structure*

This structure is not difficult to understand since it contains a movement data structure that we already saw before. It appears that we just need to add a constant value (certainly the pick-up action) and the object ID that we want to pick up, ended by some empty data. Since the pick-up action encompasses a movement data, such a network communication sent to the server will both teleport the player to the right place and then pick up the designated object.

We can now collect all eggs locations and automate the pick-up actions in our proxy as following with the command *egg*:

```python
elif cmd[:3] == 'egg':
    """
    11 - GoldenEgg1 -25045.00 18085.00 260.00
    12 - GoldenEgg2 -51570.00 -61215.00 5020.00
    13 - GoldenEgg3 24512.00 69682.00 2659.00
    14 - GoldenEgg4 60453.00 -17409.00 2939.00
    15 - GoldenEgg5 1522.00 14966.00 7022.00
    16 - GoldenEgg6 11604.00 -13131.00 411.00
    17 - GoldenEgg7 -72667.00 -53567.00 1645.00
    18 - GoldenEgg8 48404.00 28117.00 704.00
    19 - GoldenEgg9 65225.00 -5740.00 4928.00
    20 - BallmerPeakEgg -2778.00 -11035.00 10504.00
    """
    eggs = ["6d76" + struct.pack("fff", -25045.00, 18085.00, 260.00).encode('hex') + "d1f78d3a00000000" + ("ee"+struct.pack("I", 11)).encode('hex'),
        "6d76" + struct.pack("fff", -51570.00, -61215.00, 5020.00).encode('hex') + "d1f78d3a00000000" + ("ee"+struct.pack("I", 12)).encode('hex'),
        "6d76" + struct.pack("fff", 24512.00, 69682.00, 2659.00).encode('hex') + "d1f78d3a00000000" + ("ee"+struct.pack("I", 13)).encode('hex'),
        "6d76" + struct.pack("fff", 60453.00, -17409.00, 2939.00).encode('hex') + "d1f78d3a00000000" + ("ee"+struct.pack("I", 14)).encode('hex'),
	"6d76" + struct.pack("fff", 1522.00, 14966.00, 7022.00).encode('hex') + "d1f78d3a00000000" + ("ee"+struct.pack("I", 15)).encode('hex'),
        "6d76" + struct.pack("fff", 11604.00, -13131.00, 411.00).encode('hex') + "d1f78d3a00000000" + ("ee"+struct.pack("I", 16)).encode('hex'),
        "6d76" + struct.pack("fff", -72667.00, -53567.00, 1645.00).encode('hex') + "d1f78d3a00000000" + ("ee"+struct.pack("I", 17)).encode('hex'),
	"6d76" + struct.pack("fff", 48404.00, 28117.00, 704.00).encode('hex') + "d1f78d3a00000000" + ("ee"+struct.pack("I", 18)).encode('hex'),
        "6d76" + struct.pack("fff", 65225.00, -5740.00, 4928.00).encode('hex') + "d1f78d3a00000000" + ("ee"+struct.pack("I", 19)).encode('hex'),
        "6d76" + struct.pack("fff", -2778.00, -11035.00, 10504.00).encode('hex') + "d1f78d3a00000000" + ("ee"+struct.pack("I", 20)).encode('hex')]
    for egg in eggs:
        parser.SERVER_QUEUE.append(egg.decode('hex'))
        time.sleep(5)
```

We add 5 seconds of delay between each egg collected as the server seems to perform a validation of our position, preventing us to teleport too far too quickly.

![Almost all eggs collected](/assets/images/pwn-adventure/egg-1.gif)

This worked pretty well, except that one of the egg was not collected succesfully. It is the egg that has a different name: the *BallmerPeakEgg*. By going manually to its position, we are supposed to find it at a cabin in the mountains, but it is nowhere to be found.

The solution here consists in going back to the game client analysis, and search for "ballmer" in the library symbols. We find a curious function named `BallmerPeakPoster::Damage`, which contains code that include the **CowboyCoder**, one of the weapons available in the game.

![Ballmer research](/assets/images/pwn-adventure/ballmer-1.png)

![BallmerPeakPoster function](/assets/images/pwn-adventure/ballmer-2.png)

Without more information, a wild guest suggests that damaging the poster in the cabin (the BallmerPeak poster) with the Cowboy Coder would trigger something related to the egg.

![BallmerPeakEgg collected](/assets/images/pwn-adventure/egg-2.gif)

This action makes the last egg appear, that we can then collect to receive another flag.

# Bonus hack: Overachiever

With all the flags unlocked above, we are now able to easily get all achievements (shortcut L in-game) by exploring the game a little more:
- Obtain the ice spell on the spider boss in the cavern level;
- Reach the final puzzle room, part of the Blocky quest;
- Kill other players in PvP, which simply requires another account connected at the same time.

![Overachiever](/assets/images/pwn-adventure/overachiever.gif)

# Results and lessons learnt

So far I was able to get 750 / 1950 points available for this video game CTF. The three more difficult flags remain to be collected, as you can see in the list of the flags available:

| Challenge        			| Flag           			| Technique  			|
| ------------				| ----					| ---------			|
| (200 pts) Unbearable Revenge    	| They couldnt bear the sight of you 	| Network communication 	|
| (100 pts) Until the Cows Come Home	| I shouldve used dynamic link		| Reverse engineering		|
| (200 pts) Overachiever		| Achievement Unlocked Red Ding of Death| Unlocked with other flags	|
| (250 pts) Egg Hunter			| The fortress of Anorak is all yours	| Network communication		|
| (500 pts) Pirate's Treasure		| **Locked**				| 				|
| (300 pts) Fire and Ice		| **Locked**				| 				|
| (400 pts) Blocky's Revenge		| **Locked**				| 				|

From the exploits presented above, we can already understand why video game developers must consider security as a very important aspect, and this is achieved with several principles and mechanisms that we can observe in video games nowadays, especially when most of them integrate an online aspect (more or less critical according to the game category):
- Never trust the client
- Store secret information on the server
- Establish a patch management
- Detect deviations from normal player behavior
- Set up a reporting system

These mitigation mechanisms are critical especially for games that are only played online, like the MMORPGs, featured with this CTF.

I may consider this hacking exploration continued in the future if I come back to attack the three remaining challenges, either with the Windows or the Linux approach :)

