# Task 1: Public-key based authentication 
**Question 1**: 
Implement public-key based authentication step-by-step with openssl according the following scheme.


**Answer 1**:
##Step 1: Creating different docker compose:
First, we set up 2 different computers, the Server-Client to be exact

![Server](image.png)
And Client:<br>
![Client].(Client.png)

Next, we will create the private key for encrytion, in the client side of course
##Step 2: Creating public and private key 
First, we have to install some nessesary packets: Openssl, nmap, SSH, so on

![Installing](InstallSSL.PNG)
Then, we create a private and public key in the client by using

` openssl genrsa -out client_private_key.pem 2048`

And 

`openssl rsa -in client_private_key.pem -pubout -out client_public_key.pem`<br>

Then, we create a "Challenge" file<br>

And 'CHALLENGE$(openssl rand -base64 32) echo -n "$CHALLENGE" | openssl rsautl -encrypt -pubin -inkey client_public_key.pem -out encrypted_challenge.bin'

After that, we send the file from the server to the client, returning them with Challenge message encrypted by public key

We are here: `Creating Client and Server`-> `Set public key and private key` -> `Create message file`-> *Send encrypted file*-> `Decrypted message by private key`->`Sign with the private key`->'Send back to Server'->`Server received and decrypt by private key`

Next, we will Decrypt the challenge message:

I'm current storing them in boot/client_private_key.pem:

`openssl pkeyutl -decrypt -inkey boot/client_private_key.pem -in /tmp/encrypted_challenge.bin -out decrypted_challenge.txt`

This is a bit conventional, but I just use the cmd to send file:

`docker cp encrypted_challenge.bin Client:/ecrypted_challenge.bin`

Now, I can decrypt using the private key:

`# openssl pkeyutl -decrypt -inkey /boot/client_private_key.pem -in /dev/encrypted_challenge.bin -out decrypted_challenge.txt`

A decrypted_challenge.txt file is made

![image](https://github.com/user-attachments/assets/b9baf5e7-39f0-4471-87f7-2f0112e48c1d)

From the binary file, we can observe the result in decrypted_challenge.txt file in dockerfile, as create :

![image](https://github.com/user-attachments/assets/6e032612-e89f-41d0-9bc9-d32a80601b74)

Message said *This is encrypted message* 

Let's sign it with the private key to send back to server:
`openssl pkeyutl -decrypt -inkey boot/client_private_key.pem -in /boot/encrypted_challenge.bin -out decrypted_challenge.txt`

And sign them:

`openssl dgst -sha256 -sign /path/to/client_private_key.pem -out signed_challenge.bin decrypted_challenge.txt`

I use SHA-256 to sign it, expect them to solve the same 

Next, we will send back to server, this time I have installed file scp:

`scp signed_challenge.bin Client@172.17.0.4/16.:/boot/`

Now, it should be in boot/challenge.bin

Then, on the server's side, I will read and verified if the sender was the client or not:

`openssl dgst -sha256 -verify /boot/client_public_key.pem -signature /dev/signed_challenge.bin dev/decrypted_challenge.txt`

It shows the following:
![image](https://github.com/user-attachments/assets/beac8d6e-ee4d-4368-b4c4-91058ee01f34)

`Verify OK`

As for my researching, it means that the OpenSSL has completely verify the signature for it to decrypt sucessfully:
![image](https://github.com/user-attachments/assets/c280739c-44c6-4fd1-88f8-272c921df395)

I have sucessfully verify the signature, yeah!

# Task 2: Encrypting large message 
Create a text file at least 56 bytes.
**Question 1**:
Encrypt the file with aes-256 cipher in CFB and OFB modes. How do you evaluate both cipher as far as error propagation and adjacent plaintext blocks are concerned. 
**Answer 1**:
 First, we create the text file (I'm only using ChatGPT so if the file is not 56 bytes valid, don't mark me down please:<)

We will have the following file:
![image](https://github.com/user-attachments/assets/7fe127d8-e05e-459b-b825-e7dc649dd52e)

**Note: I have change the file to random.txt for integrity**

Next, I will try to check for 56 bits:

![image](https://github.com/user-attachments/assets/4fc682d3-9579-4bc8-83b7-25d503eafa83)

Then, I will create a Key and IV ( Initialization vector )

I will put a key with the size of 256 (looks like SHA-256 to me though)

`openssl rand -hex 32 > key.txt`

Then, the Initialization vector (IV)

`openssl rand -hex 16 > iv.txt`

Why 16 but not 32 of 8? Because AES Block Size: AES has a fixed block size of 128 bits (16 bytes). This is a design characteristic of AES, regardless of whether the key size is 128, 192, or 256 bits.

And for the IV Size Requirement: The IV size must match the block size of the cipher, which is 16 bytes for AES. This is because the IV is used to XOR with the first plaintext block (in CFB, OFB, CBC, etc.), or to initialize a feedback mechanism.

Now, using OFB mode:

First, we use OpenSSL with OFB mode to encrypt file:

`openssl enc -aes-256-ofb -in random.txt -out encrypted_ofb.bin -K $(cat key.txt) -iv $(cat iv.txt)`

Same thing for CFB, change syntax to aes-256-ofb to aes-256-cfb

`openssl enc -aes-256-cfb -in random.txt -out encrypted_ofb.bin -K $(cat key.txt) -iv $(cat iv.txt)`

Next, we will decrypt (so difficult to find syntax T.T)

Using `openssl enc -aes-256-cfb -d -in encrypted_cfb.bin -out decrypted_cfb.txt -K $(cat key.txt) -iv $(cat iv.txt)`

And `openssl enc -aes-256-ofb -d -in encrypted_ofb.bin -out decrypted_ofb.txt -K $(cat key.txt) -iv $(cat iv.txt)`

![image](https://github.com/user-attachments/assets/a84029d3-6bb3-48c8-91dd-4c21e1600656)

Next, we compare what changes, how many bytes it took,etc

![image](https://github.com/user-attachments/assets/d32ca514-3d9c-4e80-ba1c-c3754da29ec2)

Firstly, both done a great job encrypting and decrypting, took less then 1 sec to do both task (OFB and CFB)

![image](https://github.com/user-attachments/assets/2f094533-a1dd-415a-93cd-ca1a69c68123)

Encryption method seems to be different (of course)

And it looks like cfb is taking more memory

Adjacent Plaintext Blocks:

CFB:

Each plaintext block depends on the encryption of the previous ciphertext block, ensuring that adjacent blocks are linked.

OFB:

The keystream is independent of the ciphertext, meaning adjacent plaintext blocks are encrypted using a predictable keystream, but they remain independent of each other.


**Question 2**:
Modify the 8th byte of encrypted file in both modes (this emulates corrupted ciphertext).
Decrypt corrupted file, watch the result and give your comment on Chaining dependencies and Error propagation criteria.

**Answer 2**:

First, let's change them by using hex converter:

![image](https://github.com/user-attachments/assets/3993978c-4e3c-41a6-b02b-ae766c4dfe45)

Next, I use Powershell to format Hex:

`$ Format-Hex '.\random.txt'`

![image](https://github.com/user-attachments/assets/d3a548f9-d8ed-44a3-bdd1-915a6e07e376)

It shows the same thing:

Now, let's modify the 8th byte:




