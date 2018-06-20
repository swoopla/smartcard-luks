# opensc_luks
## How to use a PKCS#15-compliant smartcard to unlock a LUKS encrypted /root
This has been tested on Ubuntu 18.04 LTS with a smartcard and a generic USB smartcard reader. The Ubuntu instance has an unencrypted /boot partition and an encrypted /root partition. Essential info for unlocking the partition was found here: https://blog.g3rt.nl/luks-smartcard-or-token.html. After many VM restarts I was able to put together the patches for the initramfs hook and script and am successfully using this on several laptops.

For smartcard, i'm inspired of the great works of https://github.com/ramann/smartcard-luks/.
For USBKey, i'm inspired of the great works of https://www.oxygenimpaired.com/debian-lenny-luks-encrypted-root-hidden-usb-keyfile.1


### The general steps:
* Erase and initialize card
* Create public/private key pair on smartcard
* Create key file and add it to a LUKS key slot
* Encrypt key file using public key from smartcard
* Modify initramfs to use smartcard to decrypt the encrypted keyfile
* Modify decrypt_opensc script to swicth between usbkey, smartcard or luks password

### The details:
1. Install smartcard middleware

    ```sudo apt-get install pcscd opensc```

2. Erase smartcard

    ```pkcs15-init -E```
    
3. Initialize smartcard

    ```pkcs15-init --create-pkcs15 -p pkcs15+onepin --pin 1234 --puk 4321```
    
4. Create public/private key pair on smartcard

    ```pkcs15-init -G rsa/2048 -i 01 -a 01 -u decrypt --pin 1234```
    
5. Create a random key file and add it to a LUKS key slot

    ```sudo touch /root/rootkey```
    
    ```sudo chmod 600 /root/rootkey```

    ```sudo dd if=/dev/random of=/root/rootkey bs=1 count=245 #change to urandom if you can't wait```
    
    ```sudo cryptsetup luksAddKey /dev/sda2 /root/rootkey```
    
6. Export the public key from smartcard

    ```pkcs15-tool --read-public-key 01 -o public_key_rsa2048.pem```

7. Encrypt key file using public key

    ```sudo openssl rsautl -encrypt -pubin -inkey public_key_rsa2048.pem  -in /root/rootkey -out /root/rootkey.enc```
    
    ```sudo rm /root/rootkey```

8. Edit crypttab. This change sends the encrypted key file as a param to the keyscript

    This should be of the form: 
    
    ```mapped_device_name source_block_device key_file luks,keyscript=decrypt_opensc```
    
    For example:
    
    ```sda2_crypt UUID=d332ecc5-ce8b-4900-a04a-a79abd029d6d /root/rootkey.enc luks,keyscript=decrypt_opensc```

9. Apply patch to cryptopensc hook and regenerate initramfs

    ```sudo patch /lib/cryptsetup/script/decryp_cryptopensc < decrypy_opensc.patch ```
    
    ```sudo update-initramfs -u```
