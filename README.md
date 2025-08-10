# Oxy Macro Format

> [!NOTE]
> I've decided that I will release the documentation first and later complete the reference decoder/encoder for both [Hydro](https://github.com/EESports7/Hydro) and Oxy.   
> I might make a FAQ section if I feel the need to.   
> If you have any questions, feel free to make an issue request.   

This format focuses entirely on size, but the performance isn't *the worse*. This format uses a combination of [Zstandard compression](https://github.com/facebook/zstd), [vu128 variable integers](https://john-millikin.com/vu128-efficient-variable-length-integers), and grouped input types to optimize for size. An added bonus of grouped inputs is the ability to multithread parsing each input in parallel. Ideally, this format **should be implemented along side [Hydro](https://github.com/EESports7/Hydro)** because this format does not save inputs that happen multiple times in a single frame in a lossless manner. (See [this section](https://github.com/EESports7/Oxy/edit/main/README.md#sub-frame-logic) for more info)
## Structure

Version 1.0.0

Files should end in ".oxy"
### Header (Metadata)
```
std::string magicNumber = "OXYGMD";                // 
oxy::SemVar oxyVersion = oxy::SemVer(1,0,0);       // 
uint32_t errorCheck;                               // Uses CRC32 on macro data for integrity checks

std::string macroName;                             // 
std::string macroCreator;                          // 
std::string macroDescription;                      // Optional

uint32_t gameVersion;                              // 2.2704 -> 22704

std::string botName;                               // 
oxy::SemVer botVersion;                            // 

std::string levelName;                             // 
uint32_t levelID;                                  // 

uint64_t replayTime;                               // Contains the time it takes to replay the entire macro
uint64_t totalFrames;                              // Contains the total amount of frames elapsed

uint32_t deathCount;                               // Contains how many times the intentional death feature is used

uint32_t p1InputCount;                             // Shouldn't contain TPS changes, deaths, or physic changes
uint32_t p2InputCount;                             // Swift clicks count as 2 inputs

uint32_t bitmask;                                  // Information can be found below
```

### Macro Data (Compressed)

This should be the only data compressed if compression is used

```
std::vector<int32_t> seeds;                  // If there is only one seed, that seed will be use for all attempts
std::vector<PlayerPos> playerSpawn;          // First position is the starting position, contains where the player spawns for each attempt on replay
std::vector<double> tps;                     // First TPS is the starting TPS

// All of these vectors in this block contain frame deltas encoded using vu128
std::vector<uint32_t> P1JUMPCLICK;            //
std::vector<uint32_t> P1JUMPREL;              //
std::vector<uint32_t> P2JUMPCLICK;            //
std::vector<uint32_t> P2JUMPREL;              //
std::vector<uint32_t> TPSCHANGE;              //
std::vector<uint32_t> DEATH;                  //
std::vector<uint32_t> SETPHYSICS;             //
std::vector<uint32_t> RESERVED;               //
std::vector<uint32_t> P1LEFTCLICK;            //
std::vector<uint32_t> P1LEFTREL;              //
std::vector<uint32_t> P1RIGHTCLICK;           //
std::vector<uint32_t> P1RIGHTREL;             //
std::vector<uint32_t> P2LEFTCLICK;            //
std::vector<uint32_t> P2LEFTREL;              //
std::vector<uint32_t> P2RIGHTCLICK;           //
std::vector<uint32_t> P2RIGHTREL;             //
std::vector<uint32_t> P1JUMPSWIFT;            //
std::vector<uint32_t> P2JUMPSWIFT;            //
std::vector<uint32_t> P1LEFTSWIFT;            //
std::vector<uint32_t> P1RIGHTSWIFT;           //
std::vector<uint32_t> P2LEFTSWIFT;            //
std::vector<uint32_t> P2RIGHTSWIFT;           //

// These variables should only be read if indicated by the bitmask
uint8_t physicsFreq = 1;                     // Defaults to 1
std::vector<PhysicsState> player1States;     //
std::vector<PhysicsState> player2States;     //
```

## Data Structures

> [!IMPORTANT] 
> Vectors should be encoded starting with their length in 32 bits, followed by their contents.   
> Strings should be null-terminated.   
### Bitmask

| Size    | Data                                                                             |
| ------- | -------------------------------------------------------------------------------- |
| 1 Bit   | Compression Mode (None or Zstandard)                                             |
| 2 Bits  | Physics Mode (None, every n frames, every input, or explicitly stated only)      |
| 1 Bit   | Use Sub-frame Logic (More information below) (Included for parsing optimization) |
| 1 Bit   | Platform Mode                                                                    |
| 1 Bit   | Low Detail Mode                                                                  |
| 3 Bits  | Coin Collected                                                                   |
| 22 Bits | Inputs Activated (See input values for correct order)                            |
| 1 Bit   | Reserved                                                                         |

### oxy::SemVer
```
uint16_t major;
uint16_t minor;
uint16_t patch;
std::string prNote;
```

### oxy::PlayerPos
```
double xPos;
double yPos;
```

### oxy::PhysicsState
```
double xPos;
double yPos;
double xVel;
double yVel;
double rot;
```

## Input Values

If anyone knows if platformer swift clicks actually do anything, message me or make an issue request

| Number | Action                      |
| ------ | --------------------------- |
| 0      | Player 1 Jump Click         |
| 1      | Player 1 Jump Release       |
| 2      | Player 2 Jump Click         |
| 3      | Player 2 Jump Release       |
| 4      | TPS Change                  |
| 5      | Intentional Death           |
| 6      | Set Player Physics/Position |
| 7      | Reserved                    |
| 8      | Player 1 Left Click         |
| 9      | Player 1 Left Release       |
| 10     | Player 1 Right Click        |
| 11     | Player 1 Right Release      |
| 12     | Player 2 Left Click         |
| 13     | Player 2 Left Release       |
| 14     | Player 2 Right Click        |
| 15     | Player 2 Right Release      |
| 16     | Player 1 Jump Swift         |
| 17     | Player 1 Left Swift         |
| 18     | Player 1 Right Swift        |
| 19     | Player 2 Jump Swift         |
| 20     | Player 2 Left Swift         |
| 21     | Player 2 Right Swift        |
# Sub-Frame Logic

In order to mitigate most issues concerning this method of storing macro data, addition logic is needed. First and foremost, the first step is to include an additional swift click input type. (It should be known that swift click shouldn't trigger the sub-frame logic check.) Importantly, macros should contain an equal amount of clicks and releases, and if this isn't true, a release can be safely added on the same frame before the next click. In theory, this would limit the first input to be a release on a frame, followed by any number of swift clicks, and ending in a click. (See table below) Concerning the order of platformer and player clicks, this shouldn't matter too much so to standardize, it should be in the order of the input values listed above.

This only concerns decoding, not encoding.

## Singular Frame Example

Correct

| Input 1 | Input 2 | Input 3 | Input 4 |
| ------- | ------- | ------- | ------- |
| Release | Swift   | Swift   | Click   |

Incorrect   

| Input 1 | Input 2 | Input 3 | Input 4 |
| ------- | ------- | ------- | ------- |
| Click   | Swift   | Swift   | Click   |

Incorrect   

| Input 1 | Input 2 | Input 3 | Input 4 |
| ------- | ------- | ------- | ------- |
| Click   | Swift   | Swift   | Release |


# Special Thanks

[GDR](https://github.com/maxnut/GDReplayFormat/tree/gdr2) - Despite its flaws, it is the first successful universal format with lots of metadata   
[SLC](https://github.com/silicate-bot/slc) - Very amazing format that balances size and performance   
any format that used compression (e.g. [Rush - LZMA](http://discord.gg/Pj9nyfTMWh), [OBot - LZMA](https://discord.com/invite/2PaSqR92Dv), RBot - Gzip) - Honestly the most underrated feature I have ever seen used   
[ZCB](https://github.com/zeozeozeo/zcb3) - Great source of public domain macro encoders/decoders   
[Crystal Client](https://github.com/ninXout/Crystal-Client) - First time I've ever seen the use of input grouping   
