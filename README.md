# Apple Home Key

<p float="left">
 <img src="./assets/HOME.KEY.DEMO.webp" alt="![Reading a Home Key with a PN532]" width=250px>
</p>
<sub>No, it's not Photoshop, although there's a trick involved</sub>
<br>

# Notes

⚠️Reverse-engineering of a Home Key protocol has not yet been completed and thus this repository does not provide complete specification as to how to replicate the protocol. The main goal of this document is to provide a starting point for people that plan on researching this topic to save their time and to help with collaboration.  
Information already available inclues:  
- What standard Home Key is based on;
- HomeKit:
    - How the lock is configured via HomeKit;
    - Primitives, operations;
    - Common data structures;
- NFC:
    - ECP;
    - Applets are used;
    - Which commands are used;
    - What is the content of command payloads;
    - What is the expected command response format.

There are two problems solving which could help complete the reverse-engineering:
1. Analysing home kit traffic on a real lock:  
    Right now we know which data is being written and read from a lock as HomeKit does it even with a virtual lock, being able to look at communication with a real device (I assume it can be done via a BLE HomeKit MITM) could help us to understand data structures more in-depth;
2. Decrypting NFC data:  
    Data transferred in NFC protocol is encrypted using a common derived ECDH key. As of now we know how to get the keys, but a problem remains regarding the KDF, as Apple have switched up the static values in order to make reverse-engineering more difficult. This is the main issue regarding the protocol. Possible solutions could involve:  
    - Brute forcing possible static shared info part variants, knowing all keys and an approximate decrypted data format. If a combination of words is used this is doeable, otherwise pure luck or SOL.
    - [A divine intervention](https://pastebin.com).


If you can help with any of the issues, I'm ready to cooperate and provide sample data. Feel free to create an issue or a PR.

**Information published here is not in final form. Many ammendments are planned to be done throughout this week to make this document fully self-sufficient**

# Overview

Apple Home Key is an NFC protocol used by select HomeKit certified locks to authenticate a user using a virtual key provisioned into their Apple device.  

This protocol is largely based on a REDACTED standard (look into links), although there are some changes in regards to initial provisioning/pairing to adapt to Home Kit, plus static shared info used during KDF has been changed, presumably to protect against people like the ones writing this text right now.

Just like tha parent protocol, Home Key has following advantages:
- This protocol provides reader authentication, data encryption, forward secrecy;
- It provides a "Mailbox" mechanism that allows to sync lock configuration data even in situations when a lock is not connected to the internet:
    - Key revocations;
    - Invitations;
    - Attestations.
- It allows to freely share keys (although this functionality is disabled due to some reason);
- In theory, future locks could implement UWB-based access as it's supported by specification.


# HomeKit

Information in this section is based on info found by [@KhaosT](https://github.com/KhaosT) for [HAP-NodeJS](https://github.com/homebridge/HAP-NodeJS);
HomeKit data is encoded in a [TLV8](https://pypi.org/project/tlv8/) format;

## Device configuration

A lock that implements a Home Key has to have an NFCAccess(`00000266-0000-1000-8000-0026BB765291`) service that has following characteristics:
- ConfigurationState(`00000263-0000-1000-8000-0026BB765291`):  
    Format: UINT16  
    Operations: NOTIFY, READ  
    Usage: Unknown
- NFCAccessControlPoint(`00000264-0000-1000-8000-0026BB765291`):  
    Format: TLV8  
    Operations: READ, WRITE, WRITE_RESPONSE  
    Usage: This characteristic is the first one being written to when a new lock is provisioned into a home. This data presumably includes:
    - Reader private key for Home Key authentication;
    - Public keys of devices that belong to members of a home;
- NFCAccessSupportedConfiguration(`00000265-0000-1000-8000-0026BB765291`):  
    Format: TLV8  
    Operations: READ  
    Usage: Unknown, presumably to read lock information, such as:
    - Maximum amount of key slots;
    - Key slot configuration/state;
    - Extra features?

## Characteristics operations

This section will describe data structures being written to NFCAccessControlPoint characteristic;

Prematurely, following are known:
- HomeKey Home private key provisioning;
- Buddy device provisioning;
- Invitation provisioning.

**TODO Provide examples and explanation** 

## Key Color

A home device may have a AccessoryInformation(`0000003E-0000-1000-8000-0026BB765291`) service, which has a RO HardwareFinish(`0000026C-0000-1000-8000-0026BB765291`) characteristic. 

Hardware finish is a TLV8 with a following format:
```
    01[04]:    # Field 1
      000000   # Color HEX
      00       # Unknown (opacity?)
```

**HardwareFinish of the first lock added to your home installation influences the color of the lock art for the whole home.**

Following finish variations are mentioned in IOS system files:

   | Color  | Art                                                                                  | Value      | Notes                                                                                                |
   | ------ | ------------------------------------------------------------------------------------ | ---------- | ---------------------------------------------------------------------------------------------------- |
   | Black  | <img src="./assets/HOME.KEY.COLOR.BLACK.webp" alt="![Black Home Key]" width=200px>   | `00 00 00` |                                                                                                      |
   | Gold   | <img src="./assets/HOME.KEY.COLOR.GOLD.webp" alt="![Gold Home Key]" width=200px>     | ?? ?? ??   |                                                                                                      |
   | Silver | <img src="./assets/HOME.KEY.COLOR.SILVER.webp" alt="![Silver Home Key]" width=200px> | ?? ?? ??   | Not `FF FF FF`                                                                                                     |
   | Tan    | <img src="./assets/HOME.KEY.COLOR.TAN.webp" alt="![Tan Home Key]" width=200px>       | ?? ?? ??   | The default color. If an unexisting color combo is chosen, this color will be selected as a fallback |
    
<sub>I've tried finding the colors by taking the hex values from the original pass image, but there is no direct correlation between them, so don't bother</sub>

# NFC

## ECP

[ECP](https://github.com/kormax/apple-enhanced-contactless-polling) allows Home Keys to work via express mode and is required to be implemented by certified locks.

Full Home Key ECP frame looks like this

```
   6a 02 cb 02 06 021100 deadbeefdeadbeef
```

Following characteristics can be noted:
- It belongs to the Access(`02`) reader group  with a dedicated subtype HomeKey(`06`);
- It contains a single 3-byte TCI with a value of `02 11 00`, no other variations have been met. IOS file system contains multiple copies of a Home Key pass json with this TCI;
- The final 8 bytes of an ECP Home Key frame contain reader group identifier, which allows IOS to differentiate between keys for different Home installations.  

(NOTE) Actual reader identifier is 16 bytes long, first 8 bytes are the same for all locks in a single home, while the latter 8 are unique to each one. First 8 are used for ECP.  
(BUG) If more than one Home Key is added to a device, ECP stops working correctly, as a device responds to any reader emitting Home Key ECP frame even if the reader identifier is not known to it.  
(BUG) If you disable express mode for a single key while having multiple with enabled express mode, it won't appear on a screen when brought near to a reader. Disabling express mode for all home keys fixes this issue.

## Applets and Application Identifiers

Home Key uses two application identifiers:
1. Home Key Primary:  
    `A0000008580101`. Primary, used for most commands;
2. Home Key Configuration:  
    `A0000008580102`. Used for mailbox synchronization, can only be selected after a successful authentication with a primary applet.

In most situations a reader will only use the Primary applet, Configuration will be selected only:
- When a secondary device (a watch) is authenticated for the first time;
- If a new person is invited and a key data hasn't been provisioned into a lock prior to that;
- After key revocation;

There might be more instances when mailbox is used. This information might change as we are able to research the protocol more in-depth.

## Command overview

   | Command name                  | CLA | INS | P1    | P2   | DATA                       | LE  | Notes                       |
   | ----------------------------- | --- | --- | ----- | ---- | -------------------------- | --- | --------------------------- |
   | SELECT Home Key               | 00  | A4  | 04    | 00   | Home Key AID               | 00  |                             |
   | FAST                          | 80  | 80  | FLAGS | TYPE | TLV encoded data           | 00  | Data format described below |
   | STANDARD                      | 80  | 81  | 00    | 00   | TLV encoded data           |     | Data format described below |
   | EXCHANGE                      | 80  | 84  |       |      | Encrypted TLV encoded data |     | Data format described below |
   | CONTROL FLOW                  | 80  | 3c  | STEP  | INFO |                            |     | No data, used purely for UX |
   | Select Home Key Configuration | 00  | A4  | 04    | 00   | Home Key Configuration AID | 00  |                             |

Commands are executed in a following sequence:
1. At first lock SELECTs a Key applet, verifies protocol version;
2. Lock issues a FAST command to authenticate itself and establish a secure channel via ephemeral keys.  
    A device, if sees that a lock is trusted (via signature check), generates a cryptogram over shared data for a lock to decrypt.  
    A) If a lock had previously had a successful STANDARD transaction there is enough info to verify cryptogram authenticity and we can close the session;  
    B) Otherwise a STANDARD command has to be issued;  
    C) STANDARD is also called if mailbox sync has to happen. 
3.  A successful STANDARD transaction establishes long term shared data to be used later in FAST command.  
4. The existing channel can be used to execute EXCHANGE command (for light mailbox sync).  
5. If more data has to be written, a configuration applet is selected during the same session, and using the previously established channel the data is WRITtEn and READ there. 
- To indicate a transaction status, reader notifies the device about the current transaction state in between regular commands using the CONTROL FLOW command,  
   which also serves as a trigger for a checkmark or an error appearing on a screen.

**TODO Add command overview for configuration applet**

## Commands

This section will describe following aspects of each command separately:
- command data;
- p1, p2;
- status words;
- response data.
- nested data structures;


**TODO Add command format explanation**

## Communication examples

This section will provide full real transaction examples with verbose data and explanations

**TODO Add communication examples**

# Setting up test environment

You can start playing with Home Keys even without having a real lock. To do that:
1. First, you have to download [HAP-NodeJS](https://github.com/homebridge/HAP-NodeJS) onto your system;
2. Copy the example code [provided by @KhaosT here](https://github.com/KhaosT/HAP-NodeJS/commit/80cdb1535f5bee874cc06657ef283ee91f258815) into a new file inside the HAP directory, call it "lock.js";
3. Run the code using `node lock.js` command. Wait for initialization to complete;
4. Open the Home app on your device that resides in the same network, click "+" to add an accessory. Wait for Lock to appear (should do this really fast, otherwise verify connectivity); 
5. Add the lock, take the pairing code from the bottom of example code. During the provisioning proccess, agree to set up the home key;
6. After the initialization is complete, your Home Key should appear inside of the Wallet app.
* When adding multiple locks, don't forget to change MAC, ID, Pairing info of the new instances, as similar info might confuse HomeKit.

**TODO Expand this tutorial**

# References

* Resources that helped with research:
  - Original inspiration:
    - [KhaosT - Home Key demo](https://twitter.com/followhomekit/status/1478489402028531712) [(Archive)](https://web.archive.org);
  - General:
    - [TLV8 format](https://pypi.org/project/tlv8/);
    - [List of digital keys in mobile wallets](https://en.wikipedia.org/wiki/List_of_digital_keys_in_mobile_wallets);
    - [IPSW.me](https://ipsw.me/product/iPhone) - used to get IOS IPSW to look for clues in a file system;
  - Apple resources:
    - [Apple Developer Documentation](https://developer.apple.com/documentation/);
    - [WWDC Introducting "Home Keys" (I'm not kidding)](https://developer.apple.com/videos/play/wwdc2020/10006/);
    - [Unlock your door with a home key on iPhone](https://support.apple.com/en-gb/guide/iphone/iph0dc255875/ios);
  - Device brochures:  
    - [Aqara A100 Zigbee](https://www.aqara.com/en/product/smart-door-lock-a100-zigbee);
    - [Aqara D100 Zigbee](https://www.aqara.com/en/product/smart-door-lock-d100-zigbee);
    - [Aqara Smart Lock U100](https://www.aqara.com/en/product/smart-lock-u100);
    - [Schlage encode plus](https://www.schlage.com/en/home/smart-locks/encode-plus.html);
    - [Level Lock +](https://level.co/shop/level-lock-plus);
    - [TUO Smart Lock](https://findtuo.com/pages/smart-lock).
* Devices and software used for analysis:
  - [HAP-NodeJS](https://github.com/homebridge/HAP-NodeJS) - used to create virtual lock instances in order to look at communication and data structures;
  - Proxmark3 Easy - used to sniff Home Key transactions. Proxmark3 RDV2/4 can also be used;
  - [Proxmark3 Iceman Fork](https://github.com/RfidResearchGroup/proxmark3) - firmware for Proxmark3.
