# RE-WA


## Introduction

This project intends to provide a complete description of WhatsApp internals.

I'll be adding more content as I have more free time.

## Index

- [Key generation](#key-generation)
- [Registration](#registration)
  - [Registration. Scenario 1](#registration-scenario-1)
  - [Registration. Scenario 2](#registration-scenario-2)
  - [Check if number was already registered](#check-if-number-was-already-registered)
  - [Request registration code](#request-registration-code)
  - [Send registration code](#send-registration-code)
  - [Registration token](#registration-token)
    - [iOS Token](#ios-token)
  	- [Android Token](#android-token)
  - [Recovery token](#recovery-token)
    - [iOS Recovery token](#ios-recovery-token)
    - [Android Recovery token](#android-recovery-token)



## Key generation

At install time, a libsignal client needs to generate its identity keys, registration id, and prekeys:

```java
IdentityKeyPair    identityKeyPair = KeyHelper.generateIdentityKeyPair();
int                registrationId  = KeyHelper.generateRegistrationId();
List<PreKeyRecord> preKeys         = KeyHelper.generatePreKeys(startId, 100);
SignedPreKeyRecord signedPreKey    = KeyHelper.generateSignedPreKey(identityKeyPair, 5);

// Store identityKeyPair somewhere durable and safe.
// Store registrationId somewhere durable and safe.

// Store preKeys in PreKeyStore.
// Store signed prekey in SignedPreKeyStore.
```

## Registration

Depending on the number was already registered or not with a concrete device, there will be two possible scenarios.

### Registration. Scenario 1

The number wasn't registered before in that device. It will generate an identity with pseudo random bytes and then ask to WhatsApp server. Since WhatsApp server doesn't know your identity, the client will start the registration process. The client will generate a token which is different from iOS and Android. I'll provide documentation about it later. With this token and other information, we are requesting WhatsApp server to send us a 6 digit code that can be received via SMS or voice call. Finally, we send WhatsApp server the received code and the registration process is completed. The password that will be used to authenticate in the login process is generated by the client.

<img src="https://i.imgur.com/hzHta7n.png">

### Registration. Scenario 2

The number was already registered using that device, therefore the client has the identity and WhatsApp server knows the client identity. WhatsApp server will return a new password for the client.

<img src="https://i.imgur.com/BeqfVFH.png">

### Check if number was already registered

For this purpose, it uses a parameter called `id` which is the identity or as I like to call it, recovery token. It is generated using pseudo random bytes (20 bytes), and then is stored encrypted in the device.


Parameters:

- `cc`: Country code
- `in`: Phone number without country code
- `lg`: Language
- `lc`: Language code
- `authkey`: User public key (b64 encoded)
- `e_regid`: Libsignal registration id (b64 encoded)
- `e_keytype`: [DJB Curve25519 key type](https://github.com/signalapp/libsignal-protocol-java/blob/master/java/src/main/java/org/whispersystems/libsignal/ecc/Curve.java#L17) (`0x05`) (b64 encoded)
- `e_ident`: Serialized public key (b64 encoded)
- `e_skey_id`: Signed prekey id (b64 encoded)
- `e_skey_val`: Signed prekey value (b64 encoded)
- `e_skey_sig`: Signed prekey signature (b64 encoded)
- `id`: Identity or recovery token

All these parameters are encrypted and sent to WhatsApp in a GET request. The parameters are formatted as a query before encryption:

`cc=xx&in=xxxxxx&lg=...`

The encryption is based on Diffie-Hellman key exchange:

1. Get user private key
2. Get server public key
3. Calculate agreement
4. Use agreement to encrypt communication using symmetric key cipher.

```java
ECPublicKey userPrivateKey = Curve.decodePoint(userPrivate, 0);
ECPrivateKey serverPublicKey = Curve.decodePrivatePoint(serverPublic);

byte[] sharedOne = Curve.calculateAgreement(userPrivateKey, serverPublicKey);
```

WhatsApp server public key: 
`\x8e\x8c\x0f\x74\xc3\xeb\xc5\xd7\xa6\x86\x5c\x6c\x3c\x84\x38\x56\xb0\x61\x21\xcc\xe8\xea\x77\x4d\x22\xfb\x6f\x12\x25\x12\x30\x2d`

The symmetric cipher WhatsApp uses is AES 256 GCM. They use [Spongy Castle library](https://rtyley.github.io/spongycastle/):

- IV: `\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00` (96 bits)
- Tag length: 96 bit

Sample output (hex string): 
```
206ad31866a695d70b42eab9361bf95a47ab2c999a81aec1f40d85b524511894ceedb408866c7c4cbb82a7ae98ad2b36a6329040243e943b3e76059d6bcb9e0d7a41a0c34d2b99feb3ef30161f480c69343f298781ca6a32b6fc6e4a3c36b6f680dd95255642e1ceaa8b4476cde020b84ae6a998ba2a587d79a732b3ce19124ff47f888dcf06dd29209257f4d854b07ec8dd65da57366e4432a67216cbdc8087662c6d4eb3304c35488b2164b270a9b3fda5bcd0ab3a995705ec476d5edd87b56a5a7ed88cf559d09c9b41088f1a918a919331533f9aa0211f9f789cde0214e11ef3ed19f6653eb5666b9c8f34fd9d5c68ec75bcc861c1f4c852d496a3bda06259eef360ab3c3acc80fe737eb4e3d22ebfa0fe4481ce5233fd05ee0a0efe7139573bddca3834af1186406ae904c80ce96f69d6954c35883175f27018845cc8eac55af377884bd8028937e3564258a130adb9dcd7602497474204c893974babcabdf6ccf31b284c511b519b5f93da5f84c57849ff6a20a437277817e75855cfc1946beb7e42d25b485b1bac2128e1587edcda00d2b0a6b9
```

**Note**: Auth tag is added.

To this output, user public key is appended at the beginning. Final output:

- `<user public key> +  <Encrypted data>`

Request URL: `https://v.whatsapp.net/v2/exist`

GET Param: 

- `ENC`: Base64 encoded value of final output (URL safe).

Sample:

```
https://v.whatsapp.net/v2/exist?ENC=4maAaXTGZ4aeSql_Gu-QmGdxlJVjKdhafxx0tTfEc1EgatMYZqaV1wtC6rk2G_laR6ssmZqBrsH0DYW1JFEYlM7ttAiGbHxMu4KnrpitKzamMpBAJD6UOz52BZ1ry54NekGgw00rmf6z7zAWH0gMaTQ_KYeBymoytvxuSjw2tvaA3ZUlVkLhzqqLRHbN4CC4SuapmLoqWH15pzKzzhkST_R_iI3PBt0pIJJX9NhUsH7I3WXaVzZuRDKmchbL3ICHZixtTrMwTDVIiyFksnCps_2lvNCrOplXBexHbV7dh7VqWn7YjPVZ0JybQQiPGpGKkZMxUz-aoCEfn3ic3gIU4R7z7Rn2ZT61ZmucjzT9nVxo7HW8yGHB9MhS1JajvaBiWe7zYKs8OsyA_nN-tOPSLr-g_kSBzlIz_QXuCg7-cTlXO93KODSvEYZAaukEyAzpb2nWlUw1iDF18nAYhFzI6sVa83eIS9gCiTfjVkJYoTCtudzXYCSXR0IEyJOXS6vKvfbM8xsoTFEbUZtfk9pfhMV4Sf9qIKQ3J3gX51hVz8GUa-t-QtJbSFsbrCEo4Vh-3NoA0rCmuQ%3D%3D
```

### Request registration code

Parameters are encrypted the same way as explained in [Check if number was already registered](#check-if-number-was-already-registered).

Parameters:


- `cc`: Country code
- `in`: Phone number without country code
- `lg`: Language
- `lc`: Language code
- `mcc`: Mobile Country Code
- `mnc`: Mobile Network Code
- `sim_mcc`: MCC from the SIM
- `sim_mnc`: MNC from the SIM
- `method`: `sms` or `voice`.
- `reason`: `jailbroken` if device is jailbroken, else empty
- `token`: Token
- `authkey`: User public key (b64 encoded)
- `e_regid`: Libsignal registration id (b64 encoded)
- `e_keytype`: [DJB Curve25519 key type](https://github.com/signalapp/libsignal-protocol-java/blob/master/java/src/main/java/org/whispersystems/libsignal/ecc/Curve.java#L17) (`0x05`) (b64 encoded)
- `e_ident`: Serialized public key (b64 encoded)
- `e_skey_id`: Signed prekey id (b64 encoded)
- `e_skey_val`: Signed prekey value (b64 encoded)
- `e_skey_sig`: Signed prekey signature (b64 encoded)
- `network_radio_type`: Network radio type
- `simnum`: `1` if [MSISDN](http://www.qtc.jp/3GPP/Specs/23003-3e0.pdf) length > 6, else `0`
- `hasinrc`: `1` if rc file exists, else `0`
- `pid`: Process ID
- `rc`: Release type
- `id`: Identity or recovery token


#### Network radio type

Detects which type of radio network the client is using. The client uses `NetworkInfo` to detect network type and subtype, once detected the [type of network](https://developer.android.com/reference/android/telephony/TelephonyManager.html) is using, it assigns a specific value:

- WiFi: `1`
- GPRS: `104`
- EDGE: `100`
- UMTS: `102`
- CDMA: `108`
- EVD0: `103`
- 1xRTT: `109`
- HSDPA: `105`
- HSUPA: `106`
- HSPA: `107`
- IDEN: `101`
- LTE: `111`
- EHRPD: `110`
- HSPAP: `112`
- Unkown: `0`

#### rc

- release: `0`
- beta: `1`
- alpha: `2`
- debug: `3`



Request URL: `https://v.whatsapp.net/v2/code`

GET Param: 

- `ENC`: Base64 encoded value of final output (URL safe).

### Send registration code

Parameters:

- `cc`: Country code
- `in`: Phone number without country code
- `lg`: Language
- `lc`: Language code
- `authkey`: User public key (b64 encoded)
- `e_regid`: Libsignal registration id (b64 encoded)
- `e_keytype`: [DJB Curve25519 key type](https://github.com/signalapp/libsignal-protocol-java/blob/master/java/src/main/java/org/whispersystems/libsignal/ecc/Curve.java#L17) (`0x05`) (b64 encoded)
- `e_ident`: Serialized public key (b64 encoded)
- `e_skey_id`: Signed prekey id (b64 encoded)
- `e_skey_val`: Signed prekey value (b64 encoded)
- `e_skey_sig`: Signed prekey signature (b64 encoded)
- `id`: Identity or recovery token
- `code`: Received code (xxxxxx)
- `entered`: `1`

Request URL: `https://v.whatsapp.net/v2/register`

GET Param: 

- `ENC`: Base64 encoded value of final output (URL safe).


### Registration token

#### iOS Token

- `waString`: `0a1mLfGUIBVrMKF1RdvLI5lkRBvof6vn0fD2QRSM`

Token: `md5(waString + md5(app package) + Phone number without country code)`

Example:

```python
import hashlib

waString = "0a1mLfGUIBVrMKF1RdvLI5lkRBvof6vn0fD2QRSM" # Static
packageMD5 = "a200e8c6b58fda4c7d569aacfa2119a7" # Changes
number = "000000000"

token = hashlib.md5((waString+packageMD5+number).encode('utf-8')).hexdigest()
```

#### Android Token

- waPrefix: `Y29tLndoYXRzYXBw`
- Signature: `MIIDMjCCAvCgAwIBAgIETCU2pDALBgcqhkjOOAQDBQAwfDELMAkGA1UEBhMCVVMxEzARBgNVBAgTCkNhbGlmb3JuaWExFDASBgNVBAcTC1NhbnRhIENsYXJhMRYwFAYDVQQKEw1XaGF0c0FwcCBJbmMuMRQwEgYDVQQLEwtFbmdpbmVlcmluZzEUMBIGA1UEAxMLQnJpYW4gQWN0b24wHhcNMTAwNjI1MjMwNzE2WhcNNDQwMjE1MjMwNzE2WjB8MQswCQYDVQQGEwJVUzETMBEGA1UECBMKQ2FsaWZvcm5pYTEUMBIGA1UEBxMLU2FudGEgQ2xhcmExFjAUBgNVBAoTDVdoYXRzQXBwIEluYy4xFDASBgNVBAsTC0VuZ2luZWVyaW5nMRQwEgYDVQQDEwtCcmlhbiBBY3RvbjCCAbgwggEsBgcqhkjOOAQBMIIBHwKBgQD9f1OBHXUSKVLfSpwu7OTn9hG3UjzvRADDHj+AtlEmaUVdQCJR+1k9jVj6v8X1ujD2y5tVbNeBO4AdNG/yZmC3a5lQpaSfn+gEexAiwk+7qdf+t8Yb+DtX58aophUPBPuD9tPFHsMCNVQTWhaRMvZ1864rYdcq7/IiAxmd0UgBxwIVAJdgUI8VIwvMspK5gqLrhAvwWBz1AoGBAPfhoIXWmz3ey7yrXDa4V7l5lK+7+jrqgvlXTAs9B4JnUVlXjrrUWU/mcQcQgYC0SRZxI+hMKBYTt88JMozIpuE8FnqLVHyNKOCjrh4rs6Z1kW6jfwv6ITVi8ftiegEkO8yk8b6oUZCJqIPf4VrlnwaSi2ZegHtVJWQBTDv+z0kqA4GFAAKBgQDRGYtLgWh7zyRtQainJfCpiaUbzjJuhMgo4fVWZIvXHaSHBU1t5w//S0lDK2hiqkj8KpMWGywVov9eZxZy37V26dEqr/c2m5qZ0E+ynSu7sqUD7kGx/zeIcGT0H+KAVgkGNQCo5Uc0koLRWYHNtYoIvt5R3X6YZylbPftF/8ayWTALBgcqhkjOOAQDBQADLwAwLAIUAKYCp0d6z4QQdyN74JDfQ2WCyi8CFDUM4CaNB+ceVXdKtOrNTQcc0e+t`
- Salt: `PkTwKSZqUfAUyR0rPQ8hYJ0wNsQQ3dW1+3SCnyTXIfEAxxS75FwkDf47wNv/c8pP3p0GXKR6OOQmhyERwx74fw1RYSU10I4r1gyBVDbRJ40pidjM41G1I1oN`
- `about_logo.png`: Image inside apk. You can find it [here](https://github.com/mgp25/RE-WhatsApp/blob/master/Android/about_logo.png).
- classesMD5: MD5 hash of `classes.dex`. You can use a tool I made to get this value. [classesMD5-64](https://github.com/mgp25/classesMD5-64).

Example:

```python
import hashlib
import pbkdf2
import base64

waStringDecoded = base64.b64decode(waString)
saltDecoded = base64.b64decode(salt)

with open("about_logo.png", "rb") as imageFile:
  f = imageFile.read()
  imageBytes = bytearray(f)

password = waStringDecoded + imageBytes

key = pbkdf2.pbkdf2(hashlib.sha1, password, saltDecoded, 128, 80)

keyDecoded = bytearray(base64.b64decode(key))
sigDecoded = base64.b64decode(signature)
clsDecoded = base64.b64decode(classesMD5)

data = sigDecoded + clsDecoded + number

opad = bytearray()
ipad = bytearray()
for i in range(0, 64):
    opad.append(0x5C ^ key[i])
    ipad.append(0x36 ^ key[i])
hash = hashlib.sha1()
subHash = hashlib.sha1()

subHash.update(ipad + data)
hash.update(opad + subHash.digest())
subHash.update(bytes(ipad + data))
hash.update(bytes(opad + subHash.digest()))

token = base64.b64encode(hash.digest())
```

To save some steps, you can use the precalculated key (you need to b64 decode it):

key: `eQV5aq/Cg63Gsq1sshN9T3gh+UUp0wIw0xgHYT1bnCjEqOJQKCRrWxdAe2yvsDeCJL+Y4G3PRD2HUF7oUgiGo8vGlNJOaux26k+A2F3hj8A=`


### Recovery token

See [Registration. Scenario 2](#registration-scenario-2).

The recovery token is used to identify an already registed user in a specific device, and it changes if the account is activated in other device.

In case the user needs to reinstall the app in the device for whatever reason, it won't go for all registration steps (no token and no SMS). WhatsApp server will give the client a new password directly just by knowing the number and the recovery token.

The recovery token is stored in:

- Android: `/data/data/com.whatsapp/files/rc2`
- iOS: `/var/mobile/Containers/Data/Application/<App UUID>/Library/rc.dat`

#### iOS Recovery token

- `rc.dat` content (hex):

`253338253130253842254531253344254231254235253732253146253944254445253945253630253232253837253438`

Which in ascii is just the recovery token, no encryption, just as easy as this:

Recovery token: `%38%10%8B%E1%3D%B1%B5%72%1F%9D%DE%9E%60%22%87%48`

#### Android Recovery token

Steps:

1. Read rc2 and read Java Object.
2. From the unserialized data, we obtain: header, iv, salt and encrypted data values.
3. Get the password:

`Password = RC_SECRET + regex(number) + account_name`

4. We obtain the key to decrypt the data by deriving the password using pbkdf2 (sha1).
5. Finally, we can decrypt the data using the derived key and using AES 128 OFB.
6. The recovery token is the decrypted data urlencoded.

Example: 

```python
import re
import base64
import hashlib
from Crypto.Cipher import AES
from binascii import hexlify, unhexlify
import urllib.parse

class WhatsAppSecurity:

    RECOVERY_TOKEN_HEADER = b"\x00\x02"

    def __init__(self, phone_number, google_play_email = ""):
        self.pn = phone_number
        self.email = google_play_email

    def get_recovery_token(self, rc_data):
        secret = WhatsAppData().get_rc_secret() + self.get_recovery_jid_from_jid(self.pn) + self.email;
        return self.get_encrypted_data(secret, rc_data)

    def get_rc_file_data(self, recovery_token_file):
        with open(recovery_token_file, 'rb') as f:
            read_data = f.read()
        return read_data

    def get_encrypted_data(self, secret, data):
        data = data[27:]
        header = data[:2]
        salt = data[2:6]
        iv = data[6:22]
        encrypted_data = data[22:42]

        if header != self.RECOVERY_TOKEN_HEADER:
            raise Exception('Header mismatch')

        key = self.get_key(secret, salt)
        cipher = AES.new(key, AES.MODE_OFB, iv)

        return cipher.decrypt(encrypted_data)

    def get_key(self, secret, salt):
        return hashlib.pbkdf2_hmac('sha1', bytes(secret, 'utf-8'), salt, 16, 16)

    def get_recovery_jid_from_jid(self, phone_number):
        c = re.compile("^([17]|2[07]|3[0123469]|4[013456789]|5[12345678]|6[0123456]|8[1246]|9[0123458]|\d{3})\d*?(\d{4,6})$")
        g = c.match(phone_number)

        if g is not None:
            return g.group(1) + g.group(2)
        else:
            return ""

class WhatsAppData:

    def __init__(self):
        self.RC_SECRET = self.decode("A\u0004\u001d@\u0011\u0018V\u0091\u0002\u0090\u0088\u009f\u009eT(3{;ES")

    def decode(self, s):
        sb = []
        for i in range(len(s)):
            sb.append(self.sxor("\u0012", s[i]))
        return ''.join(sb)


    def sxor(self, s1, s2):
        # convert strings to a list of character pair tuples
        # go through each tuple, converting them to ASCII code (ord)
        # perform exclusive or on the ASCII code
        # then convert the result back to ASCII (chr)
        # merge the resulting array of characters as a string
        return ''.join(chr(ord(a) ^ ord(b)) for a,b in zip(s1,s2))

    def get_rc_secret(self):
        return self.RC_SECRET


phone_number = '34123456789' # country_code + number
account_name = '' # Google Play Email. If not set, don't change this value.

ws = WhatsAppSecurity(phone_number, '')
rc_data = ws.get_rc_file_data('rc2') # Opening and reading rc2 file data

recovery_token = ws.get_recovery_token(rc_data)

print(urllib.parse.quote_plus(recovery_token))
```

## Shared Data

### iOS Shared Data

Data stored locally in `NSUserDefaults` are encrypted using AES 128 CBC. 

- String key: `s.whatsapp.net`
- IV: `\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00`

Example:

```python
from Crypto.Cipher import AES
from binascii import hexlify, unhexlify
import hashlib
import base64

country_code = "34"
waString = "s.whatsapp.net"

md5 = hashlib.md5(waString.encode('utf-8')).hexdigest()
key = bytes(md5[:16], 'utf-8')

BLOCK_SIZE = 16 # AES 128 CBC

data = bytes(country_code, 'utf-8')
padding_length = BLOCK_SIZE - len(data)
pad_data = data + bytes(chr(padding_length), 'utf-8') * padding_length
iv = "\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00";
cipher = AES.new(key, AES.MODE_CBC, iv)

encrypted_data = base64.b64encode(cipher.encrypt(pad_data))
print(encrypted_data)
```

