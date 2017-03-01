This is a practical guide to using [YubiKey](https://www.yubico.com/faq/yubikey/) as a [SmartCard](https://security.stackexchange.com/questions/38924/how-does-storing-gpg-ssh-private-keys-on-smart-cards-compare-to-plain-usb-drives) for storing GPG encryption and signing keys. An authentication key can also be created for SSH and used with [gpg-agent](https://unix.stackexchange.com/questions/188668/how-does-gpg-agent-work/188813#188813).

Keys stored on a smartcard like YubiKey are more secure than ones stored on disk and are convenient enough for everyday use.

Instructions written for OSX using YubiKey 4 in OTP+CCID mode. Note, older YubiKeys are limited to 2048 bit RSA keys. The following has been tested on OSX 10.11.4.

The information provided below is a combination of work from Keith Swett (keith.swett@wheniwork.com), Dr Duh (https://github.com/drduh/YubiKey-Guide) and the Freenode #yubikey community. Thanks for all the help! :) 

- [Install required software](#install-required-software)
- [Configure smartcard](#configure-smartcard)
  - [Change PINs](#change-pins)
  - [Set card information](#set-card-information)
- [Creating keys](#creating-keys)
  - [Create temporary working directory for GPG](#create-temporary-working-directory-for-gpg)
  - [Create configuration](#create-configuration)
  - [Create master key](#create-master-key)
  - [Save Key ID](#save-key-id)
  - [Create revocation certificate](#create-revocation-certificate)
  - [Back up master key](#back-up-master-key)
  - [Create subkeys](#create-subkeys)
    - [Signing key](#signing-key)
    - [Encryption key](#encryption-key)
    - [Authentication key](#authentication-key)
  - [Check your work](#check-your-work)
  - [Export subkeys](#export-subkeys)
  - [Back up everything](#back-up-everything)
  - [Configure YubiKey](#configure-yubikey)
  - [Transfer keys](#transfer-keys)
    - [Signature key](#signature-key)
    - [Encryption key](#encryption-key-1)
    - [Authentication key](#authentication-key-1)
  - [Check your work](#check-your-work-1)
  - [Export public key](#export-public-key)
  - [Finish](#finish)
- [Using keys](#using-keys)
  - [Create GPG configuration](#create-gpg-configuration)
  - [Import public key](#import-public-key)
  - [Insert YubiKey](#insert-yubikey)
  - [GnuPG](#gnupg)
    - [Trust master key](#trust-master-key)
    - [Encryption](#encryption)
    - [Decryption](#decryption)
    - [Signing](#signing)
    - [Verifying signature](#verifying-signature)
  - [SSH](#ssh)
    - [Update configuration](#update-configuration)
    - [Replace ssh-agent with gpg-agent](#replace-ssh-agent-with-gpg-agent)
    - [Copy public key to server](#copy-public-key-to-server)
    - [Connect with public key authentication](#connect-with-public-key-authentication)
- [Troubleshooting](#troubleshooting)
- [References](#references)

# Install required software

You will need to install the following software:

Homebrew, an alternative package manager for OSX:

    $ /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"

Once homebrew has been installed, run:

    $ brew update

Install the necessary packages:

    $ brew install git-crypt yubikey-personalization socat
    $ brew install gpg-agent gnupg2
    $ brew install Caskroom/cask/yubikey-neo-manager
    $ brew install Caskroom/cask/yubikey-personalization-gui
    $ brew install Caskroom/cask/gpgtools

# Configure the Yubikey

Once the packages have been installed, we need to modify the Yubikey configuration to run in OTP and CCID mode. To do this, run the following command:

    $ ykpersonalize -m82
    Firmware version 4.3.1 Touch level 517 Program sequence 1

    The USB mode will be set to: 0x82

    Commit? (y/n) [n]: y

# Creating keys

## Create temporary working directory for GPG

Create a temporary directory which won't survive a [reboot](https://serverfault.com/questions/377348/when-does-tmp-get-cleared):

    $ export GNUPGHOME=$(mktemp -d) ; echo $GNUPGHOME
    `/var/folders/m_/t3y0ptl111zd0kbs6590gw_40000gn/T/tmp.4dMGGCBi`

## Create configuration

Paste the following [text](https://stackoverflow.com/questions/2500436/how-does-cat-eof-work-in-bash) into a terminal window to create a [recommended](https://github.com/drduh/config/blob/master/gpg.conf) GPG configuration:

    $ cat << EOF > $GNUPGHOME/gpg.conf
    default-preference-list SHA512 SHA384 SHA256 SHA224 AES256 AES192 AES CAMELLIA256 CAMELLIA192 CAMELLIA128 TWOFISH
    cert-digest-algo SHA512
    keyid-format 0xlong
    use-agent
    lock-never
    EOF

## Create master key

Generate a new key with GPG, selecting RSA (sign only), the appropriate keysize (4096), and no expiration date. The following procedure will also prompt to create and confirm a unique passphrase. 

    $ gpg --gen-key

    Please select what kind of key you want:
       (1) RSA and RSA (default)
       (2) DSA and Elgamal
       (3) DSA (sign only)
       (4) RSA (sign only)
    Your selection? 4
    RSA keys may be between 1024 and 4096 bits long.
    What keysize do you want? (2048) 4096
    Requested keysize is 4096 bits
    Please specify how long the key should be valid.
             0 = key does not expire
          <n>  = key expires in n days
          <n>w = key expires in n weeks
          <n>m = key expires in n months
          <n>y = key expires in n years
    Key is valid for? (0) 0
    Key does not expire at all
    Is this correct? (y/N) y

    GnuPG needs to construct a user ID to identify your key.

    Real name: Firstname Lastname
    Email address: firstname.lastname@wheniwork.com
    Comment:
    You selected this USER-ID:
        "Firstname Lastname <firstname.lastname@wheniwork.com>"

    Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? o
    You need a Passphrase to protect your secret key.

    We need to generate a lot of random bytes. It is a good idea to perform
    some other action (type on the keyboard, move the mouse, utilize the
    disks) during the prime generation; this gives the random number
    generator a better chance to gain enough entropy.
    gpg: /var/folders/m_/t3y0ptl111zd0kbs6590gw_40000gn/T/tmp.LXjFxkDy/trustdb.gpg: trustdb created
    gpg: key 0xF932D46EFBBF395C marked as ultimately trusted
    public and secret key created and signed.

    gpg: checking the trustdb
    gpg: 3 marginal(s) needed, 1 complete(s) needed, PGP trust model
    gpg: depth: 0  valid:   1  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 1u
    pub   4096R/0xF932D46EFBBF395C 2016-08-03
          Key fingerprint = AA88 FC68 946B 42FF E1CC  0EBD F932 D46E FBBF 395C
    uid                 [ultimate] Firstname Lastname <firstname.lastname@wheniwork.com>

    Note that this key cannot be used for encryption.  You may want to use
    the command "--edit-key" to generate a subkey for this purpose.

Keep this passphrase handy as you'll need it throughout.

## Save Key ID

Export the key ID as a [variable](https://stackoverflow.com/questions/1158091/defining-a-variable-with-or-without-export/1158231#1158231) for use throughout the configuration process:

    $ KEYID=0xF932D46EFBBF395C

## Add a photo

Next you will want a real picture of you and shouldn't be bigger than 240x288 and limit it to less then 14KB. Use the following commands to add the photo.

    $ gpg --edit-key $KEYID

    gpg (GnuPG/MacGPG2) 2.0.30; Copyright (C) 2015 Free Software Foundation, Inc.
    This is free software: you are free to change and redistribute it.
    There is NO WARRANTY, to the extent permitted by law.

    Secret key is available.

    pub  4096R/0xF932D46EFBBF395C  created: 2016-08-03  expires: never       usage: SC
                                   trust: ultimate      validity: ultimate
    [ultimate] (1). Firstname Lastname <firstname.lastname@wheniwork.com>

    gpg> addphoto

    Pick an image to use for your photo ID.  The image must be a JPEG file.
    Remember that the image is stored within your public key.  If you use a
    very large picture, your key will become very large as well!
    Keeping the image close to 240x288 is a good size to use.

    Enter JPEG filename for photo ID: /Users/firstnamelastname/Desktop/photo.jpg
    Is this photo correct (y/N/q)? y

    You need a passphrase to unlock the secret key for
    user: "Firstname Lastname <firstname.lastname@wheniwork.com>"
    4096-bit RSA key, ID 0xF932D46EFBBF395C, created 2016-08-03


    pub  4096R/0xF932D46EFBBF395C  created: 2016-08-03  expires: never       usage: SC
                                   trust: ultimate      validity: ultimate
    [ultimate] (1). Firstname Lastname <firstname.lastname@wheniwork.com>
    [ unknown] (2)  [jpeg image of size 2750]

    gpg> save

## Create revocation certificate

Create a way to revoke your keys in case of loss or compromise, an explicit reason being optional:

    $ gpg --gen-revoke $KEYID > $GNUPGHOME/revoke.txt

    sec  4096R/0xF932D46EFBBF395C 2016-05-24 Firstname Lastname <firstname.lastname@wheniwork.com>

    Create a revocation certificate for this key? (y/N) y
    Please select the reason for the revocation:
      0 = No reason specified
      1 = Key has been compromised
      2 = Key is superseded
      3 = Key is no longer used
      Q = Cancel
    (Probably you want to select 1 here)
    Your decision? 1
    Enter an optional description; end it with an empty line:
    >
    Reason for revocation: Key has been compromised
    (No description given)
    Is this okay? (y/N) y

    You need a passphrase to unlock the secret key for
    user: "Firstname Lastname <firstname.lastname@wheniwork.com>"
    4096-bit RSA key, ID 0xFF3E7D88647EBCDB, created 2016-05-24

    ASCII armored output forced.
    Revocation certificate created.

    Please move it to a medium which you can hide away; if Mallory gets
    access to this certificate he can use it to make your key unusable.
    It is smart to print this certificate and store it away, just in case
    your media become unreadable.  But have some caution:  The print system of
    your machine might store the data and make it available to others!

## Back up master key

Save a copy of the private key block:

    $ gpg --armor --export-secret-keys $KEYID > $GNUPGHOME/master.key

## Create subkeys

Edit the key to add subkeys:

    $ gpg --expert --edit-key $KEYID

    gpg (GnuPG/MacGPG2) 2.0.30; Copyright (C) 2015 Free Software Foundation, Inc.
    This is free software: you are free to change and redistribute it.
    There is NO WARRANTY, to the extent permitted by law.

    Secret key is available.

    gpg: checking the trustdb
    gpg: 3 marginal(s) needed, 1 complete(s) needed, PGP trust model
    gpg: depth: 0  valid:   1  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 1u
    pub  4096R/0xF932D46EFBBF395C  created: 2016-08-03  expires: never       usage: SC
                                   trust: ultimate      validity: ultimate
    [ultimate] (1). Firstname Lastname <firstname.lastname@wheniwork.com>
    [ultimate] (2)  [jpeg image of size 2750]

    gpg>

### Signing key

First, create a [signing key](https://stackoverflow.com/questions/5421107/can-rsa-be-both-used-as-encryption-and-signature/5432623#5432623), selecting RSA (sign only):

    gpg> addkey
    Key is protected.

    You need a passphrase to unlock the secret key for
    user: "Firstname Lastname <firstname.lastname@wheniwork.com>"
    4096-bit RSA key, ID 0xF932D46EFBBF395C, created 2016-08-03

    Please select what kind of key you want:
       (3) DSA (sign only)
       (4) RSA (sign only)
       (5) Elgamal (encrypt only)
       (6) RSA (encrypt only)
       (7) DSA (set your own capabilities)
       (8) RSA (set your own capabilities)
    Your selection? 4
    RSA keys may be between 1024 and 4096 bits long.
    What keysize do you want? (2048)
    Requested keysize is 2048 bits
    Please specify how long the key should be valid.
             0 = key does not expire
          <n>  = key expires in n days
          <n>w = key expires in n weeks
          <n>m = key expires in n months
          <n>y = key expires in n years
    Key is valid for? (0) 0
    Key expires at Fri Nov 11 11:29:52 2016 CST
    Is this correct? (y/N) y
    Really create? (y/N) y
    We need to generate a lot of random bytes. It is a good idea to perform
    some other action (type on the keyboard, move the mouse, utilize the
    disks) during the prime generation; this gives the random number
    generator a better chance to gain enough entropy.

    pub  4096R/0xF932D46EFBBF395C  created: 2016-08-03  expires: never       usage: SC
                                   trust: ultimate      validity: ultimate
    sub  2048R/0x1E7E95EA22108AE7  created: 2016-08-03  expires: 2016-11-11  usage: S
    [ultimate] (1). Firstname Lastname <firstname.lastname@wheniwork.com>
    [ultimate] (2)  [jpeg image of size 2750]

    gpg>

### Encryption key

Next, create an [encryption key](https://www.cs.cornell.edu/courses/cs5430/2015sp/notes/rsa_sign_vs_dec.php), selecting RSA (encrypt only):

    gpg> addkey
    Key is protected.

    You need a passphrase to unlock the secret key for
    user: "Firstname Lastname <firstname.lastname@wheniwork.com>"
    4096-bit RSA key, ID 0xF932D46EFBBF395C, created 2016-08-03

    Please select what kind of key you want:
       (3) DSA (sign only)
       (4) RSA (sign only)
       (5) Elgamal (encrypt only)
       (6) RSA (encrypt only)
       (7) DSA (set your own capabilities)
       (8) RSA (set your own capabilities)
    Your selection? 6
    RSA keys may be between 1024 and 4096 bits long.
    What keysize do you want? (2048)
    Requested keysize is 2048 bits
    Please specify how long the key should be valid.
             0 = key does not expire
          <n>  = key expires in n days
          <n>w = key expires in n weeks
          <n>m = key expires in n months
          <n>y = key expires in n years
    Key is valid for? (0) 0
    Key expires at Fri Nov 11 11:32:21 2016 CST
    Is this correct? (y/N) y
    Really create? (y/N) y
    We need to generate a lot of random bytes. It is a good idea to perform
    some other action (type on the keyboard, move the mouse, utilize the
    disks) during the prime generation; this gives the random number
    generator a better chance to gain enough entropy.

    pub  4096R/0xF932D46EFBBF395C  created: 2016-08-03  expires: never       usage: SC
                                   trust: ultimate      validity: ultimate
    sub  2048R/0x1E7E95EA22108AE7  created: 2016-08-03  expires: 2016-11-11  usage: S
    sub  2048R/0xAF6A01035CFD7DF0  created: 2016-08-03  expires: 2016-11-11  usage: E
    [ultimate] (1). Firstname Lastname <firstname.lastname@wheniwork.com>
    [ultimate] (2)  [jpeg image of size 2750]

    gpg>

### Authentication key

Finally, create an [authentication key](https://superuser.com/questions/390265/what-is-a-gpg-with-authenticate-capability-used-for), selecting RSA (set your own capabilities):

    gpg> addkey
    Key is protected.

    You need a passphrase to unlock the secret key for
    user: "Firstname Lastname <firstname.lastname@wheniwork.com>"
    4096-bit RSA key, ID 0xF932D46EFBBF395C, created 2016-08-03

    Please select what kind of key you want:
       (3) DSA (sign only)
       (4) RSA (sign only)
       (5) Elgamal (encrypt only)
       (6) RSA (encrypt only)
       (7) DSA (set your own capabilities)
       (8) RSA (set your own capabilities)
    Your selection? 8

    Possible actions for a RSA key: Sign Encrypt Authenticate
    Current allowed actions: Sign Encrypt

       (S) Toggle the sign capability
       (E) Toggle the encrypt capability
       (A) Toggle the authenticate capability
       (Q) Finished

    Your selection? s

    Possible actions for a RSA key: Sign Encrypt Authenticate
    Current allowed actions: Encrypt

       (S) Toggle the sign capability
       (E) Toggle the encrypt capability
       (A) Toggle the authenticate capability
       (Q) Finished

    Your selection? e

    Possible actions for a RSA key: Sign Encrypt Authenticate
    Current allowed actions:

       (S) Toggle the sign capability
       (E) Toggle the encrypt capability
       (A) Toggle the authenticate capability
       (Q) Finished

    Your selection? a

    Possible actions for a RSA key: Sign Encrypt Authenticate
    Current allowed actions: Authenticate

       (S) Toggle the sign capability
       (E) Toggle the encrypt capability
       (A) Toggle the authenticate capability
       (Q) Finished

    Your selection? q
    RSA keys may be between 1024 and 4096 bits long.
    What keysize do you want? (2048)
    Requested keysize is 2048 bits
    Please specify how long the key should be valid.
             0 = key does not expire
          <n>  = key expires in n days
          <n>w = key expires in n weeks
          <n>m = key expires in n months
          <n>y = key expires in n years
    Key is valid for? (0) 0
    Key expires at Fri Nov 11 11:33:21 2016 CST
    Is this correct? (y/N) y
    Really create? (y/N) y
    We need to generate a lot of random bytes. It is a good idea to perform
    some other action (type on the keyboard, move the mouse, utilize the
    disks) during the prime generation; this gives the random number
    generator a better chance to gain enough entropy.

    pub  4096R/0xF932D46EFBBF395C  created: 2016-08-03  expires: never       usage: SC
                                   trust: ultimate      validity: ultimate
    sub  2048R/0x1E7E95EA22108AE7  created: 2016-08-03  expires: 2016-11-11  usage: S
    sub  2048R/0xAF6A01035CFD7DF0  created: 2016-08-03  expires: 2016-11-11  usage: E
    sub  2048R/0x6B41354877FC08DA  created: 2016-08-03  expires: 2016-11-11  usage: A
    [ultimate] (1). Firstname Lastname <firstname.lastname@wheniwork.com>
    [ultimate] (2)  [jpeg image of size 2750]

    gpg> save

## Check your work

List your new secret keys:

    $ gpg --list-secret-keys
    /var/folders/m_/t3y0ptl111zd0kbs6590gw_40000gn/T/tmp.LXjFxkDy/secring.gpg
    -------------------------------------------------------------------------
    sec   4096R/0xF932D46EFBBF395C 2016-08-03
    uid                            Firstname Lastname <firstname.lastname@wheniwork.com>
    uid                            [jpeg image of size 2750]
    ssb   2048R/0x1E7E95EA22108AE7 2016-08-03
    ssb   2048R/0xAF6A01035CFD7DF0 2016-08-03
    ssb   2048R/0x6B41354877FC08DA 2016-08-03

## Export subkeys

Save a copy of your subkeys:

    $ gpg --armor --export-secret-keys $KEYID > $GNUPGHOME/mastersub.key

    $ gpg --armor --export-secret-subkeys $KEYID > $GNUPGHOME/sub.key
    
## Back up everything

Once keys are moved to hardware, they cannot be extracted again (otherwise, what would be the point?), so make sure you have made a backup before proceeding.

Insert the USB thumbdrive you were provided by WIW and backup the temporary GPG working directory:

    $ cp -avi $GNUPGHOME /Volumes/WIW_USB_THUMBDRIVE/
    /var/folders/m_/t3y0ptl111zd0kbs6590gw_40000gn/T/tmp.LXjFxkDy -> /Volumes/WIW_USB_THUMBDRIVE/tmp.LXjFxkDy
    /var/folders/m_/t3y0ptl111zd0kbs6590gw_40000gn/T/tmp.LXjFxkDy/gpg.conf -> /Volumes/WIW_USB_THUMBDRIVE/tmp.LXjFxkDy/gpg.conf
    /var/folders/m_/t3y0ptl111zd0kbs6590gw_40000gn/T/tmp.LXjFxkDy/master.key -> /Volumes/WIW_USB_THUMBDRIVE/tmp.LXjFxkDy/master.key
    /var/folders/m_/t3y0ptl111zd0kbs6590gw_40000gn/T/tmp.LXjFxkDy/mastersub.key -> /Volumes/WIW_USB_THUMBDRIVE/tmp.LXjFxkDy/mastersub.key
    /var/folders/m_/t3y0ptl111zd0kbs6590gw_40000gn/T/tmp.LXjFxkDy/private-keys-v1.d -> /Volumes/WIW_USB_THUMBDRIVE/tmp.LXjFxkDy/private-keys-v1.d
    /var/folders/m_/t3y0ptl111zd0kbs6590gw_40000gn/T/tmp.LXjFxkDy/pubring.gpg -> /Volumes/WIW_USB_THUMBDRIVE/tmp.LXjFxkDy/pubring.gpg
    /var/folders/m_/t3y0ptl111zd0kbs6590gw_40000gn/T/tmp.LXjFxkDy/pubring.gpg~ -> /Volumes/WIW_USB_THUMBDRIVE/tmp.LXjFxkDy/pubring.gpg~
    /var/folders/m_/t3y0ptl111zd0kbs6590gw_40000gn/T/tmp.LXjFxkDy/random_seed -> /Volumes/WIW_USB_THUMBDRIVE/tmp.LXjFxkDy/random_seed
    /var/folders/m_/t3y0ptl111zd0kbs6590gw_40000gn/T/tmp.LXjFxkDy/revoke.txt -> /Volumes/WIW_USB_THUMBDRIVE/tmp.LXjFxkDy/revoke.txt
    cp: /var/folders/m_/t3y0ptl111zd0kbs6590gw_40000gn/T/tmp.LXjFxkDy/S.gpg-agent: Operation not supported on socket
    /var/folders/m_/t3y0ptl111zd0kbs6590gw_40000gn/T/tmp.LXjFxkDy/secring.gpg -> /Volumes/WIW_USB_THUMBDRIVE/tmp.LXjFxkDy/secring.gpg
    /var/folders/m_/t3y0ptl111zd0kbs6590gw_40000gn/T/tmp.LXjFxkDy/sub.key -> /Volumes/WIW_USB_THUMBDRIVE/tmp.LXjFxkDy/sub.key
    /var/folders/m_/t3y0ptl111zd0kbs6590gw_40000gn/T/tmp.LXjFxkDy/trustdb.gpg -> /Volumes/WIW_USB_THUMBDRIVE/tmp.LXjFxkDy/trustdb.gpg

## Configure YubiKey

Plug in your YubiKey and configure it:

    $ ykpersonalize -m82
    Firmware version 4.2.7 Touch level 527 Program sequence 4

    The USB mode will be set to: 0x82

    Commit? (y/n) [n]: y

> The -m option is the mode command. To see the different modes, enter ykpersonalize –help. Mode 82 (in hex) enables the YubiKey as a composite USB device (HID + CCID) and allows OTPs to be emitted while in use as a smart card.  Once you have changed the mode, you need to re-boot the YubiKey – so remove and re-insert it.

https://www.yubico.com/2012/12/yubikey-neo-openpgp/

## Configure smartcard

Use GPG to configure YubiKey as a smartcard:

    $ gpg --card-edit

    Application ID ...: D2760001240102010006055532110000
    Version ..........: 2.1
    Manufacturer .....: unknown
    Serial number ....: 05553211
    Name of cardholder: [not set]
    Language prefs ...: [not set]
    Sex ..............: unspecified
    URL of public key : [not set]
    Login data .......: [not set]
    Private DO 1 .....: [not set]
    Private DO 2 .....: [not set]
    Signature PIN ....: not forced
    Key attributes ...: 2048R 2048R 2048R
    Max. PIN lengths .: 127 127 127
    PIN retry counter : 3 0 3
    Signature counter : 0
    Signature key ....: [none]
    Encryption key....: [none]
    Authentication key: [none]
    General key info..: [none]

### Change PINs

The default PIN codes are: 
admin: `12345678`
user: `123456`

Note: the user PIN code *MUST* be *at LEAST* 6 digits. 

    gpg/card> admin
    Admin commands are allowed

    gpg/card> passwd
    gpg: OpenPGP card no. D2760001240102010006055532110000 detected

    1 - change PIN
    2 - unblock PIN
    3 - change Admin PIN
    4 - set the Reset Code
    Q - quit

    Your selection? 1
    PIN changed.

    1 - change PIN
    2 - unblock PIN
    3 - change Admin PIN
    4 - set the Reset Code
    Q - quit

    Your selection? 3
    PIN changed.

    1 - change PIN
    2 - unblock PIN
    3 - change Admin PIN
    4 - set the Reset Code
    Q - quit

    Your selection? 4
    Reset Code set.

    1 - change PIN
    2 - unblock PIN
    3 - change Admin PIN
    4 - set the Reset Code
    Q - quit

    Your selection? q

    gpg/card>

### Set card information

Set up the optional fields:

    gpg/card> name
    Cardholder's surname: Lastname
    Cardholder's given name: Firstname

    gpg/card> lang
    Language preferences: en

    gpg/card> sex
    Sex ((M)ale, (F)emale or space): m

    gpg/card>

Verify the card information:

    gpg/card> (Press Enter)

    Application ID ...: D2760001240102010006049421930000
    Version ..........: 2.1
    Manufacturer .....: Yubico
    Serial number ....: 04942193
    Name of cardholder: Firstname Lastname
    Language prefs ...: en
    Sex ..............: male
    URL of public key : [not set]
    Login data .......: [not set]
    Signature PIN ....: not forced
    Key attributes ...: 2048R 2048R 2048R
    Max. PIN lengths .: 127 127 127
    PIN retry counter : 3 3 3
    Signature counter : 0
    Signature key ....: [none]
    Encryption key....: [none]
    Authentication key: [none]
    General key info..: [none]

    gpg/card> quit

## Transfer keys

Transfering keys to YubiKey hardware is a one-way operation only, so make sure you've made a backup before proceeding:

    $ gpg --edit-key $KEYID
    gpg (GnuPG/MacGPG2) 2.0.30; Copyright (C) 2015 Free Software Foundation, Inc.
    This is free software: you are free to change and redistribute it.
    There is NO WARRANTY, to the extent permitted by law.

    Secret key is available.

    pub  4096R/0xF932D46EFBBF395C  created: 2016-08-03  expires: never       usage: SC
                                   trust: ultimate      validity: ultimate
    sub  2048R/0x1E7E95EA22108AE7  created: 2016-08-03  expires: 2016-11-11  usage: S
    sub  2048R/0xAF6A01035CFD7DF0  created: 2016-08-03  expires: 2016-11-11  usage: E
    sub  2048R/0x6B41354877FC08DA  created: 2016-08-03  expires: 2016-11-11  usage: A
    [ultimate] (1). Firstname Lastname <firstname.lastname@wheniwork.com>
    [ultimate] (2)  [jpeg image of size 2750]

Toggle to the secret key listings:

    gpg> toggle

    sec  4096R/0xF932D46EFBBF395C  created: 2016-08-03  expires: never
    ssb  2048R/0x1E7E95EA22108AE7  created: 2016-08-03  expires: never
    ssb  2048R/0xAF6A01035CFD7DF0  created: 2016-08-03  expires: never
    ssb  2048R/0x6B41354877FC08DA  created: 2016-08-03  expires: never
    (1)  Firstname Lastname <firstname.lastname@wheniwork.com>
    (2)  [jpeg image of size 2750]

    gpg>

### Signature key

Select the signature key.

    gpg> key 1

    sec  4096R/0xF932D46EFBBF395C  created: 2016-08-03  expires: never
    ssb* 2048R/0x1E7E95EA22108AE7  created: 2016-08-03  expires: never
    ssb  2048R/0xAF6A01035CFD7DF0  created: 2016-08-03  expires: never
    ssb  2048R/0x6B41354877FC08DA  created: 2016-08-03  expires: never
    (1)  Firstname Lastname <firstname.lastname@wheniwork.com>
    (2)  [jpeg image of size 2750]

    gpg>

Move the signature key (you will be prompted for the key passphrase and admin PIN):

    gpg> keytocard
    Signature key ....: [none]
    Encryption key....: [none]
    Authentication key: [none]

    Please select where to store the key:
       (1) Signature key
       (3) Authentication key
    Your selection? 1

    You need a passphrase to unlock the secret key for
    user: "Firstname Lastname <firstname.lastname@wheniwork.com>"
    2048-bit RSA key, ID 0x1E7E95EA22108AE7, created 2016-08-03


    sec  4096R/0xF932D46EFBBF395C  created: 2016-08-03  expires: never
    ssb* 2048R/0x1E7E95EA22108AE7  created: 2016-08-03  expires: never
                         card-no: 0006 04942193
    ssb  2048R/0xAF6A01035CFD7DF0  created: 2016-08-03  expires: never
    ssb  2048R/0x6B41354877FC08DA  created: 2016-08-03  expires: never
    (1)  Firstname Lastname <firstname.lastname@wheniwork.com>
    (2)  [jpeg image of size 2750]

    gpg>

Type `key 1` again to deselect the signature key.

    gpg> key 1

    sec  4096R/0xF932D46EFBBF395C  created: 2016-08-03  expires: never
    ssb  2048R/0x1E7E95EA22108AE7  created: 2016-08-03  expires: never
                         card-no: 0006 04942193
    ssb  2048R/0xAF6A01035CFD7DF0  created: 2016-08-03  expires: never
    ssb  2048R/0x6B41354877FC08DA  created: 2016-08-03  expires: never
    (1)  Firstname Lastname <firstname.lastname@wheniwork.com>
    (2)  [jpeg image of size 2750]

    gpg>

### Encryption key

Type `key 2` to select the next key:

    gpg> key 2

    sec  4096R/0xF932D46EFBBF395C  created: 2016-08-03  expires: never
    ssb  2048R/0x1E7E95EA22108AE7  created: 2016-08-03  expires: never
                         card-no: 0006 04942193
    ssb* 2048R/0xAF6A01035CFD7DF0  created: 2016-08-03  expires: never
    ssb  2048R/0x6B41354877FC08DA  created: 2016-08-03  expires: never
    (1)  Firstname Lastname <firstname.lastname@wheniwork.com>
    (2)  [jpeg image of size 2750]

    gpg>

Move the encryption key to the card (you will be prompted for the key passphrase):

    gpg> keytocard
    Signature key ....: C488 2B4E 29E2 95D8 66D2  37DF 1E7E 95EA 2210 8AE7
    Encryption key....: [none]
    Authentication key: [none]

    Please select where to store the key:
       (2) Encryption key
    Your selection? 2

    You need a passphrase to unlock the secret key for
    user: "Firstname Lastname <firstname.lastname@wheniwork.com>"
    2048-bit RSA key, ID 0xAF6A01035CFD7DF0, created 2016-08-03


    sec  4096R/0xF932D46EFBBF395C  created: 2016-08-03  expires: never
    ssb  2048R/0x1E7E95EA22108AE7  created: 2016-08-03  expires: never
                         card-no: 0006 04942193
    ssb* 2048R/0xAF6A01035CFD7DF0  created: 2016-08-03  expires: never
                         card-no: 0006 04942193
    ssb  2048R/0x6B41354877FC08DA  created: 2016-08-03  expires: never
    (1)  Firstname Lastname <firstname.lastname@wheniwork.com>
    (2)  [jpeg image of size 2750]

    gpg>

Type `key 2` to deselect the encryption key.

    gpg> key 2

    sec  4096R/0xF932D46EFBBF395C  created: 2016-08-03  expires: never
    ssb  2048R/0x1E7E95EA22108AE7  created: 2016-08-03  expires: never
                         card-no: 0006 04942193
    ssb  2048R/0xAF6A01035CFD7DF0  created: 2016-08-03  expires: never
                         card-no: 0006 04942193
    ssb  2048R/0x6B41354877FC08DA  created: 2016-08-03  expires: never
    (1)  Firstname Lastname <firstname.lastname@wheniwork.com>
    (2)  [jpeg image of size 2750]

    gpg>

### Authentication key

Type `key 3` to select the next key:

    gpg> key 3

    sec  4096R/0xF932D46EFBBF395C  created: 2016-08-03  expires: never
    ssb  2048R/0x1E7E95EA22108AE7  created: 2016-08-03  expires: never
                         card-no: 0006 04942193
    ssb  2048R/0xAF6A01035CFD7DF0  created: 2016-08-03  expires: never
                         card-no: 0006 04942193
    ssb* 2048R/0x6B41354877FC08DA  created: 2016-08-03  expires: never
    (1)  Firstname Lastname <firstname.lastname@wheniwork.com>
    (2)  [jpeg image of size 2750]

    gpg>

Move the authentication key to card (you will be prompted for the key passphrase):

    gpg> keytocard
    Signature key ....: C488 2B4E 29E2 95D8 66D2  37DF 1E7E 95EA 2210 8AE7
    Encryption key....: 5CE0 F585 0226 DF12 8813  05CF AF6A 0103 5CFD 7DF0
    Authentication key: [none]

    Please select where to store the key:
       (3) Authentication key
    Your selection? 3

    You need a passphrase to unlock the secret key for
    user: "Firstname Lastname <firstname.lastname@wheniwork.com>"
    2048-bit RSA key, ID 0x6B41354877FC08DA, created 2016-08-03


    sec  4096R/0xF932D46EFBBF395C  created: 2016-08-03  expires: never
    ssb  2048R/0x1E7E95EA22108AE7  created: 2016-08-03  expires: never
                         card-no: 0006 04942193
    ssb  2048R/0xAF6A01035CFD7DF0  created: 2016-08-03  expires: never
                         card-no: 0006 04942193
    ssb* 2048R/0x6B41354877FC08DA  created: 2016-08-03  expires: never
                         card-no: 0006 04942193
    (1)  Firstname Lastname <firstname.lastname@wheniwork.com>
    (2)  [jpeg image of size 2750]

    gpg>

Save and quit:

    gpg> save

## Check your work

`ssb>` indicates a stub to the private key on smartcard:

    $ gpg --list-secret-keys
    /var/folders/m_/t3y0ptl111zd0kbs6590gw_40000gn/T/tmp.LXjFxkDy/secring.gpg
    -------------------------------------------------------------------------
    sec   4096R/0xF932D46EFBBF395C 2016-08-03
    uid                            Firstname Lastname <firstname.lastname@wheniwork.com>
    uid                            [jpeg image of size 2750]
    ssb>  2048R/0x1E7E95EA22108AE7 2016-08-03
    ssb>  2048R/0xAF6A01035CFD7DF0 2016-08-03
    ssb>  2048R/0x6B41354877FC08DA 2016-08-03


## Back up the remaining stubs.

    $ gpg -a --export-secret-keys $KEYID > $GNUPGHOME/masterstubs.txt
    $ gpg -a --export-secret-subkeys $KEYID > $GNUPGHOME/subkeystubs.txt
    $ gpg -a --export $KEYID > $GNUPGHOME/publickey.txt

## Export public key

This file should be publicly shared:

    $ gpg2 --armor --export firstname.lastname@wheniwork.com > firstname.lastname@wheniwork.com--master.asc

## Back up the new exports

    $ cp -an $GNUPGHOME /Volumes/WIW_USB_THUMBDRIVE/

## Finish

If all went well, you should now reboot or remove `$GNUPGHOME`.

    $ unset $GNUPGHOME

# Using keys

## Create GPG configuration 

Paste the following text into a terminal window to create the recommended GPG configuration.

gpg.conf:

    $ cat << EOF > ~/.gnupg/gpg.conf
    default-preference-list SHA512 SHA384 SHA256 SHA224 AES256 AES192 AES CAMELLIA256 CAMELLIA192 CAMELLIA128 TWOFISH
    cert-digest-algo SHA512
    use-agent
    lock-never
    keyid-format 0xlong
    EOF

gpg-agent.conf:

    $ cat << EOF > ~/.gnupg/gpg-agent.conf
    pinentry-program /usr/local/MacGPG2/libexec/pinentry-mac.app/Contents/MacOS/pinentry-mac
    enable-ssh-support
    write-env-file
    use-standard-socket
    default-cache-ttl 600
    max-cache-ttl 7200
    debug-level advanced
    log-file /var/log/gpg-agent.log
    EOF

## Import public key

Import it from a file:

    $ gpg --import < /Volumes/WIW_USB_THUMBDRIVE/tmp.LXjFxkDy/publickey.txt
    gpg: /Users/firstnamelastname/.gnupg/trustdb.gpg: trustdb created
    gpg: key FBBF395C: public key "Firstname Lastname <firstname.lastname@wheniwork.com>" imported
    gpg: Total number processed: 1
    gpg:               imported: 1  (RSA: 1)

## Insert YubiKey

Check the card's status:

    $ gpg --card-status
    Application ID ...: D2760001240102010006049421930000
    Version ..........: 2.1
    Manufacturer .....: Yubico
    Serial number ....: 04942193
    Name of cardholder: Firstname Lastname
    Language prefs ...: en
    Sex ..............: male
    URL of public key : [not set]
    Login data .......: [not set]
    Signature PIN ....: not forced
    Key attributes ...: 2048R 2048R 2048R
    Max. PIN lengths .: 127 127 127
    PIN retry counter : 3 3 3
    Signature counter : 0
    Signature key ....: C488 2B4E 29E2 95D8 66D2  37DF 1E7E 95EA 2210 8AE7
          created ....: 2016-08-03 17:29:31
    Encryption key....: 5CE0 F585 0226 DF12 8813  05CF AF6A 0103 5CFD 7DF0
          created ....: 2016-08-03 17:32:14
    Authentication key: 2172 7812 1277 2743 DB3E  CD68 6B41 3548 77FC 08DA
          created ....: 2016-08-03 17:33:06
    General key info..: sub  2048R/22108AE7 2016-08-03 Firstname Lastname <firstname.lastname@wheniwork.com>
    sec#  4096R/FBBF395C  created: 2016-08-03  expires: never
    ssb>  2048R/22108AE7  created: 2016-08-03  expires: 2016-11-11
                          card-no: 0006 04942193
    ssb>  2048R/5CFD7DF0  created: 2016-08-03  expires: 2016-11-11
                          card-no: 0006 04942193
    ssb>  2048R/77FC08DA  created: 2016-08-03  expires: 2016-11-11
                          card-no: 0006 04942193

`sec#` indicates master key is not available (as it should be stored encrypted offline).

## GnuPG

### Trust master key

Edit the imported key to assign it ultimate trust. the KEYID is the one listed under 'General card info':

    $ gpg --edit-key 0x1E7E95EA22108AE7
    gpg (GnuPG/MacGPG2) 2.0.30; Copyright (C) 2015 Free Software Foundation, Inc.
    This is free software: you are free to change and redistribute it.
    There is NO WARRANTY, to the extent permitted by law.

    Secret key is available.

    pub  4096R/0xF932D46EFBBF395C  created: 2016-08-03  expires: never       usage: SC
                                   trust: unknown       validity: unknown
    sub  2048R/0x1E7E95EA22108AE7  created: 2016-08-03  expires: 2016-11-11  usage: S
    sub  2048R/0xAF6A01035CFD7DF0  created: 2016-08-03  expires: 2016-11-11  usage: E
    sub  2048R/0x6B41354877FC08DA  created: 2016-08-03  expires: 2016-11-11  usage: A
    [ unknown] (1). Firstname Lastname <firstname.lastname@wheniwork.com>
    [ unknown] (2)  [jpeg image of size 2750]

    gpg> trust
    pub  4096R/0xF932D46EFBBF395C  created: 2016-08-03  expires: never       usage: SC
                                   trust: unknown       validity: unknown
    sub  2048R/0x1E7E95EA22108AE7  created: 2016-08-03  expires: 2016-11-11  usage: S
    sub  2048R/0xAF6A01035CFD7DF0  created: 2016-08-03  expires: 2016-11-11  usage: E
    sub  2048R/0x6B41354877FC08DA  created: 2016-08-03  expires: 2016-11-11  usage: A
    [ unknown] (1). Firstname Lastname <firstname.lastname@wheniwork.com>
    [ unknown] (2)  [jpeg image of size 2750]

    Please decide how far you trust this user to correctly verify other users' keys
    (by looking at passports, checking fingerprints from different sources, etc.)

      1 = I don't know or won't say
      2 = I do NOT trust
      3 = I trust marginally
      4 = I trust fully
      5 = I trust ultimately
      m = back to the main menu

    Your decision? 5
    Do you really want to set this key to ultimate trust? (y/N) y

    pub  4096R/0xF932D46EFBBF395C  created: 2016-08-03  expires: never       usage: SC
                                   trust: ultimate      validity: unknown
    sub  2048R/0x1E7E95EA22108AE7  created: 2016-08-03  expires: 2016-11-11  usage: S
    sub  2048R/0xAF6A01035CFD7DF0  created: 2016-08-03  expires: 2016-11-11  usage: E
    sub  2048R/0x6B41354877FC08DA  created: 2016-08-03  expires: 2016-11-11  usage: A
    [ unknown] (1). Firstname Lastname <firstname.lastname@wheniwork.com>
    [ unknown] (2)  [jpeg image of size 2750]
    Please note that the shown key validity is not necessarily correct
    unless you restart the program.

    gpg> quit

### Encryption

Encrypt some sample text:

    $ echo "$(uname -a)" | gpg --encrypt --armor --recipient 0xFF3E7D88647EBCDB
    -----BEGIN PGP MESSAGE-----

    hQIMA1kSp5XpDdLPAQ/+JyYfLaUS/+llEzQaKDb5mWhG4HlUgD99dNJUXakm085h
    PSSt3I8Ac0ctwyMnenZvBEbHMqdRnfZJsj5pHidKcAZrhgs+he+B1tdZ/KPa8inx
    NIGqd8W1OraVSFmPEdC1kQ5he6R/WCDH1NNel9+fvLtQDCBQaFae/s3yXCSSQU6q
    HKCJLyHK8K9hDvgFmXOY8j1qTknBvDbmYdcCKVE1ejgpUCi3WatusobpWozsp0+b
    6DN8bXyfxLPYm1PTLfW7v4kwddktB8eVioV8A45lndJZvliSqDwxhrwyE5VGsArS
    NmqzBkCaOHQFr0ofL91xgwpCI5kM2ukIR5SxUO4hvzlHn58QVL9GfAyCHMFtJs3o
    Q9eiR0joo9TjTwR8XomVhRJShrrcPeGgu3YmIak4u7OndyBFpu2E79RQ0ehpl2gY
    tSECB6mNd/gt0Wy3y15ccaFI4CVP6jrMN6q3YhXqNC7GgI/OWkVZIAgUFYnbmIQe
    tQ3z3wlbvFFngeFy5IlhsPduK8T9XgPnOtgQxHaepKz0h3m2lJegmp4YZ4CbS9h6
    kcBTUjys5Vin1SLuqL4PhErzmlAZgVzG2PANsnHYPe2hwN4NlFtOND1wgBCtBFBs
    1pqz1I0O+jmyId+jVlAK076c2AwdkVbokKUcIT/OcTc0nwHjOUttJGmkUHlbt/nS
    iAFNniSfzf6fwAFHgsvWiRJMa3keolPiqoUdh0tBIiI1zxOMaiTL7C9BFdpnvzYw
    Krj0pDc7AlF4spWhm58WgAW20P8PGcVQcN6mSTG8jKbXVSP3bvgPXkpGAOLKMV/i
    pLORcRPbauusBqovgaBWU/i3pMYrbhZ+LQbVEaJlvblWu6xe8HhS/jo=
    =pzkv
    -----END PGP MESSAGE-----

### Decryption

Decrypt the sample text by running the following command:

    $ gpg --decrypt --armor

Paste in the encrypted message generated above:

    -----BEGIN PGP MESSAGE-----

    hQIMA1kSp5XpDdLPAQ/+JyYfLaUS/+llEzQaKDb5mWhG4HlUgD99dNJUXakm085h
    PSSt3I8Ac0ctwyMnenZvBEbHMqdRnfZJsj5pHidKcAZrhgs+he+B1tdZ/KPa8inx
    NIGqd8W1OraVSFmPEdC1kQ5he6R/WCDH1NNel9+fvLtQDCBQaFae/s3yXCSSQU6q
    HKCJLyHK8K9hDvgFmXOY8j1qTknBvDbmYdcCKVE1ejgpUCi3WatusobpWozsp0+b
    6DN8bXyfxLPYm1PTLfW7v4kwddktB8eVioV8A45lndJZvliSqDwxhrwyE5VGsArS
    NmqzBkCaOHQFr0ofL91xgwpCI5kM2ukIR5SxUO4hvzlHn58QVL9GfAyCHMFtJs3o
    Q9eiR0joo9TjTwR8XomVhRJShrrcPeGgu3YmIak4u7OndyBFpu2E79RQ0ehpl2gY
    tSECB6mNd/gt0Wy3y15ccaFI4CVP6jrMN6q3YhXqNC7GgI/OWkVZIAgUFYnbmIQe
    tQ3z3wlbvFFngeFy5IlhsPduK8T9XgPnOtgQxHaepKz0h3m2lJegmp4YZ4CbS9h6
    kcBTUjys5Vin1SLuqL4PhErzmlAZgVzG2PANsnHYPe2hwN4NlFtOND1wgBCtBFBs
    1pqz1I0O+jmyId+jVlAK076c2AwdkVbokKUcIT/OcTc0nwHjOUttJGmkUHlbt/nS
    iAFNniSfzf6fwAFHgsvWiRJMa3keolPiqoUdh0tBIiI1zxOMaiTL7C9BFdpnvzYw
    Krj0pDc7AlF4spWhm58WgAW20P8PGcVQcN6mSTG8jKbXVSP3bvgPXkpGAOLKMV/i
    pLORcRPbauusBqovgaBWU/i3pMYrbhZ+LQbVEaJlvblWu6xe8HhS/jo=
    =pzkv
    -----END PGP MESSAGE-----

You should now be prompted to enter your PIN. Output:

    gpg: encrypted with 4096-bit RSA key, ID 0x5912A795E90DD2CF, created
    2016-05-24
          "Firstname Lastname <firstname.lastname@wheniwork.com>"

Press Control-D twice. Output:

    Darwin Your-MBP 15.6.0 Darwin Kernel Version 15.6.0: Thu Jun 23 18:25:34 PDT 2016; root:xnu-3248.60.10~1/RELEASE_X86_64 x86_64

### Signing

Sign some sample text:

    $ echo "$(uname -a)" | gpg --armor --clearsign --default-key 0xFF3E7D88647EBCDB
    -----BEGIN PGP SIGNED MESSAGE-----
    Hash: SHA256

    Darwin Your-MBP 15.6.0 Darwin Kernel Version 15.6.0: Thu Jun 23 18:25:34 PDT 2016; root:xnu-3248.60.10~1/RELEASE_X86_64 x86_64
    -----BEGIN PGP SIGNATURE-----
    Version: GnuPG/MacGPG2 v2

    iQEcBAEBCAAGBQJXqCSPAAoJEB5+leoiEIrnZP4IAKeLKo+rhrdSN8q50xY4LHYA
    F94k/emaSWdgBKMkbEbuSnBwtYkZl7YzFjmYWqQ6rbvwZI9JQzBp03cAaJ6Os3GC
    fQUNznVS0HXYl/7i0kFvuLfkSxd2zuJzaLZifL3pvZVmpDtZH1K+XZkuQDPmiFRi
    FnZAxYlTXiTlgRzEU4F9ZAd3ssBjp3KQmVuLoVV4rgMmMTFPKsDYurZTyalE63nk
    oCbdZtLu4INnbOOGB+F5yk0BcoYu5WVQJevAFirSWqEqYmxD+QDxYPj1LC55TIYe
    3b/zMA1YaWikwmC64G3SHu219nDB1fJcLnKhBjuLLpx+bY0qZJrGvN/Cmyz3qio=
    =fMOU
    -----END PGP SIGNATURE-----

### Verifying signature

Verify the previous signature by running the following command:

    $ gpg
    gpg: Go ahead and type your message ...

Paste in the signed message and signature created above.

    -----BEGIN PGP SIGNATURE-----
    Version: GnuPG/MacGPG2 v2

    iQEcBAEBCAAGBQJXqCSPAAoJEB5+leoiEIrnZP4IAKeLKo+rhrdSN8q50xY4LHYA
    F94k/emaSWdgBKMkbEbuSnBwtYkZl7YzFjmYWqQ6rbvwZI9JQzBp03cAaJ6Os3GC
    fQUNznVS0HXYl/7i0kFvuLfkSxd2zuJzaLZifL3pvZVmpDtZH1K+XZkuQDPmiFRi
    FnZAxYlTXiTlgRzEU4F9ZAd3ssBjp3KQmVuLoVV4rgMmMTFPKsDYurZTyalE63nk
    oCbdZtLu4INnbOOGB+F5yk0BcoYu5WVQJevAFirSWqEqYmxD+QDxYPj1LC55TIYe
    3b/zMA1YaWikwmC64G3SHu219nDB1fJcLnKhBjuLLpx+bY0qZJrGvN/Cmyz3qio=
    =fMOU
    -----END PGP SIGNATURE-----Darwin Your-MBP 15.6.0 Darwin Kernel Version 15.6.0: Thu Jun 23 18:25:34 PDT 2016; root:xnu-3248.60.10~1/RELEASE_X86_64 x86_64

Press Control-D twice. Output:

    gpg: Signature made Sun Aug  7 23:19:59 2016 PDT
    gpg:                using RSA key 0x1E7E95EA22108AE7
    gpg: Good signature from "Firstname Lastname <firstname.lastname@wheniwork.com>" [ultimate]
    gpg:                 aka "[jpeg image of size 2750]" [ultimate]

Putting it all together:

    $ echo "$(uname -a)" | gpg --encrypt --sign --armor --default-key 0xF932D46EFBBF395C --recipient 0x1E7E95EA22108AE7 | gpg --decrypt --armor
    gpg: encrypted with 2048-bit RSA key, ID 0xAF6A01035CFD7DF0, created 2016-08-03
          "Firstname Lastname <firstname.lastname@wheniwork.com>"
    Darwin Your-MBP 15.6.0 Darwin Kernel Version 15.6.0: Thu Jun 23 18:25:34 PDT 2016; root:xnu-3248.60.10~1/RELEASE_X86_64 x86_64
    gpg: Signature made Sun Aug  7 23:32:40 2016 PDT
    gpg:                using RSA key 0x1E7E95EA22108AE7
    gpg: Good signature from "Firstname Lastname <firstname.lastname@wheniwork.com>" [ultimate]
    gpg:                 aka "[jpeg image of size 2750]" [ultimate]

## SSH

### Update configuration

Paste the following text into a terminal window to create a [recommended](https://github.com/drduh/config/blob/master/gpg-agent.conf) GPG agent configuration:

    $ cat << EOF > ~/.gnupg/gpg-agent.conf
    pinentry-program /usr/local/MacGPG2/libexec/pinentry-mac.app/Contents/MacOS/pinentry-mac
    enable-ssh-support
    write-env-file
    use-standard-socket
    default-cache-ttl 600
    max-cache-ttl 7200
    debug-level advanced
    log-file /var/log/gpg-agent.log
    EOF

### Replace ssh-agent with gpg-agent

    $ pkill ssh-agent
    $ pkill gpg-agent
    $ eval $(gpg-agent --daemon --enable-ssh-support --use-standard-socket --log-file ~/.gnupg/gpg-agent.log --write-env-file)

### Copy public key to server

Copy and paste the following output to the server authorized keys file:

    $ ssh-add -L
    ssh-rsa AAAAB4NzaC1yc2EAAAADAQABAAACAz[...]zreOKM+HwpkHzcy9DQcVG2Nw== cardno:000605553211

### Connect with public key authentication

Run the following command. 

    $ ssh -T git@github.com -vvv

You will be prompted to enter your PIN. Output:

    [...]
    debug2: key: cardno:000605553211 (0x1234567890),
    debug1: Authentications that can continue: publickey
    debug3: start over, passed a different list publickey
    debug3: preferred gssapi-keyex,gssapi-with-mic,publickey,keyboard-interactive,password
    debug3: authmethod_lookup publickey
    debug3: remaining preferred: keyboard-interactive,password
    debug3: authmethod_is_enabled publickey
    debug1: Next authentication method: publickey
    debug1: Offering RSA public key: cardno:000605553211
    debug3: send_pubkey_test
    debug2: we sent a publickey packet, wait for reply
    debug1: Server accepts key: pkalg ssh-rsa blen 535
    debug2: input_userauth_pk_ok: fp e5:de:a5:74:b1:3e:96:9b:85:46:e7:28:53:b4:82:c3
    debug3: sign_and_send_pubkey: RSA e5:de:a5:74:b1:3e:96:9b:85:46:e7:28:53:b4:82:c3
    debug1: Authentication succeeded (publickey).
    [...]



# Share your public keys

Run the following clone command to get the gpg keys public repo.

    git clone https://github.com/wheniwork/public_gpg_keys

Add your key:

    git 

# Setup Git for signing

You will want to use the following command to make git use the signing key:

    git config --global user.signingkey <key id>

To sign a commit, use the following command:

    git commit -a -S -m "<message>"

# Yubikey Touch Password Disable

To disable the single password touch feature. On MacOS you will need to download yubiswitch from Download here You will need to find the product id using the ioreg tool. Using brew you will need to install the following packages:

    brew update && brew tap jlhonora/lsusb && brew install lsusb

Once those are installed run the following command and find to find idProduct.

    ioreg -p IOUSB -l -w 0 -x | grep Yubikey -A10 | grep Product

which will output something like this:

    "idProduct" = 0x405
    "iProduct" = 0x2
    "USB Product Name" = "Yubikey 4 OTP+CCID"

Once you have yubiswitch installed and launched. Go to the preferences and set Yubikey ProductID to what the ioreg command returned for a value. Once you do that you can now enable and disable the yubikey using this. This disables the touch password feature. This shouldn't effect the gpg keys that you use for ssh and git signing.

# Troubleshooting

- If you don't understand some option, read `man gpg`.

- If you encounter problems connecting to YubiKey with GPG, simply try unplugging and re-inserting your YubiKey, and restarting the `gpg-agent` process.

- If you receive the error, `gpg: decryption failed: secret key not available` - you likely need to install GnuPG version 2.x.

- If you receive the error, `Yubikey core error: no yubikey present` - you likely need to install newer versions of yubikey-personalize as outlined in [Install required software](#install-required-software).

- If you receive the error, `Yubikey core error: write error` - YubiKey is likely locked. Install and run yubikey-personalization-gui to unlock it.

- If you receive the error, `Key does not match the card's capability` - you likely need to use 2048 bit RSA key sizes.

- If you totally screw up, you can [reset the card](https://developers.yubico.com/ykneo-openpgp/ResetApplet.html).

# References

<https://developers.yubico.com/yubikey-personalization/>

<https://developers.yubico.com/PGP/Card_edit.html>

<https://blog.josefsson.org/2014/06/23/offline-gnupg-master-key-and-subkeys-on-yubikey-neo-smartcard/>

<https://www.esev.com/blog/post/2015-01-pgp-ssh-key-on-yubikey-neo/>

<https://blog.habets.se/2013/02/GPG-and-SSH-with-Yubikey-NEO>

<https://trmm.net/Yubikey>

<https://rnorth.org/8/gpg-and-ssh-with-yubikey-for-mac>

<https://jclement.ca/articles/2015/gpg-smartcard/>

<https://github.com/herlo/ssh-gpg-smartcard-config>

<http://www.bootc.net/archives/2013/06/09/my-perfect-gnupg-ssh-agent-setup/>

<https://help.riseup.net/en/security/message-security/openpgp/best-practices>

<https://alexcabal.com/creating-the-perfect-gpg-keypair/>

<https://www.void.gr/kargig/blog/2013/12/02/creating-a-new-gpg-key-with-subkeys/>


