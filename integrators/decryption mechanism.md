
# Decryption mechanism

- The editor will deliver a protected vault instance to all integrators. This instance will contain the encryption key of the bundles delivered to the integrator. This instance will include inside it an RSA key pair. The licence file will contain the encryption key (AES symmetric key) encrypted with the public key of this pair.
- When the runtime starts, it will extract the AES key encrypted with the public key of the pair and ask vault to decrypt it with the private key and send back the decrypted key
- When the decrypted key is received, it is cut into two halves. Two random numbers are generated. Each random number is xored with each half of the key. The xored halves and the two random numbers are stored in non contiguous areas of the memory to prevent encryption key discovering by memory dump.
- whenever the runtime needs to decrypt a file of the bundle, it rebuilds the encryption key, decrypts the file, and zeroes the memory zone of the encryption key