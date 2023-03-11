# p2shdata
p2shdata protocol: A protocol to store data contained in pay-to-script-hash (p2sh) scriptsigs

Coins in a p2sh address are spent by revealing a bitcoin scriptsig that evaluates to true. The script usually includes a redeemscript with one or more public keys, along with a signature for each pubkey in the scriptsig. However other scripts, including those that just reveal arbitrary data and then drop it, can be included in the redeemscript. This protocol uses this script functionality to push data and then drop it from the operations stack in the spending transaction. The final operation pushes the value 1 (true, op_1) so that the script evaluates true and coins at the p2sh address are spendable while saving data to blockchain.
 
The p2shdata redeemscript has the following script structure	                             
OP_PUSH(varint) [data] OP_PUSH(8 bytes) [salt] OP_2DROP OP_1	
 
The redeemscript is base58(ripemd160(sha256))) to create the p2sh address. Coins are sent to the address and then spent to reveal the scriptsig. When the script is evaluated, up to 500 bytes data is pushed to the stack, then 8 bytes of salt, then those top 2 elements are dropped with OP_2DROP. OP_1 is the final element pushed to the stack to evaluate to true and spend coins from the p2sh address. Larger files can be saved by chunking in 500 byte strings in a separate address for each chunk. Addresses should be spent as transaction inputs in the order of reassembly. Each redeemscript also includes 8 bytes of random salt to prevent anyone observing the transactions from spending the coins by guessing the data that's being stored. The OP_RETURN output in the spending transaction contains metadata (site/protocol, version, filename, size, assembly script, and a hash160 of the data file), that's used to reconstruct and verify the reassembled file.

The OP_RETURN output contains metadata to reconstruct and verify the file
[ site ]	[ protocol ]	[ version ]	[ filename ]	[ filetype ]	[ filesize ]	[ assembly_script ]	[ datahash160 ]
(12 byte)	(10 byte)	(2 byte)	(16 byte)	(4 byte)	(4 byte)	(12 byte)	(20 byte)
opreturn.net	/p2shdata	hex decimal	hex	hex	hex	hex	ripemd160(sha256(data))
0 pad right	0 pad right	0 pad left	0 pad left	0 pad left	0 pad left	0 pad right	no padding


TYPICAL ORDER OF OPERATIONS TO SAVE DATA WITH P2SHDATA
Obtain data file, encode to hex, and calculate metadata needed for the op_return output (filename, type, bytesize, hash160 digest)
Divide data in 500 byte chunks. The last data chunk may be some size other than 500 bytes
Add preceeding OP_PUSHDATA2 code (4df401 to push 500 bytes to stack, or some other OP_PUSHDATA value if chunk size is under 500 bytes). For bitcoin script details, see https://en.bitcoin.it/wiki/Script
Add trailing 8 bytes random salt and OP_2DROP OP_1 codes (11 bytes total), (e.g. 085e212ab6c54e84bc6d51) to generate the redeemscript
With the completed redeemscripts (OP_PUSHDATA [DATA] OP_PUSH8 [salt] OP_2DROP OP_1), perform base58check-encode(ripemd160(sha256(redeemscript))) to generate the p2sh addresses
Fund all the p2sh addresses in 1st transaction with sufficient funds to cover fees and not be considered dust
Generate a second transaction to spend addresses from step 6. Addresses are spent by placing the entire redeemscript, along with the appropriate OP_PUSHDATA OP codes, into the scriptsig location of the transaction vin(s); no signatures are needed for these p2sh pushscript transactions. Include op_return output with p2shdata metadata from step 1 and assembly-script
Once transaction 2 is confirmed, the scriptsigs for the vins contain data from step 1 and are assembled & decoded according to the assembly script, and verifed by comparing the hash160 in the op_return output with your own calculated hash160 of the assembled data payload

REASSEMBLING DATA FROM SCRIPTSIG INPUTS

As discussed above, the redeemscripts have the structure:

OP_PUSH(varint) [data] OP_PUSH(8 bytes) [salt] OP_2DROP OP_1

Since only the [data] portion of the revealed scriptsig needs to be reassembled, the leading OP codes used to push the data as well as the trailing [salt] and OP codes should be removed before assembly. A typical data chunk of 500 bytes will have this following structure:

4df401 . [500 bytes data] . 08 . [8 bytes salt] . 6d51

where the 3 leading bytes (6 characters: 4df401) should be dropped since this is the OP code to push 500 bytes onto the stack. After the data is pushed, the 11 trailing bytes of OP code and salt (08.salt.6d51) should be dropped.

Note that when data does not fit perfectly into 500 byte chunks, the last transaction scriptsig will be less than 500 bytes. Since pushing smaller chunks of data may require fewer OP codes, you may need to calculate the size of the last scriptsig to determine leading byte count to drop. However if you're using a bitcoin script parser to find the data elements this won't be necesary.

Scriptsigs < 76 bytes use 1 byte of OP code to push the data (e.g. 35)
Scriptsigs 76-255 bytes use 2 bytes of OP codes to push the data (e.g. 4c96)
scriptsigs 256-500 bytes use 3 bytes of OP codes to push the data (e.g. 4df401)

A simple way to determine the leading bytes to drop would be to check the first scriptsig byte:

if == '4d' drop 3 bytes
elseif == '4c' drop 2 bytes
else drop 1 byte

Drop trailing 11 bytes of salt and OP codes, then concatenate all the chunks in the transaction vin order. Perform the hash160 function to check against the hash value in the OP_RETURN output to verify data integrity. Encode per the assemblyscript if OP ec exists, otherwise encode as ascii


ADVANTAGES OF P2SHDATA OVER OTHER METHODS

Scriptsigs provide a much larger datasize than op_return or unspendable outputs to store data. Since the protocol doesn't use signatures, the vast majority of the transaction size is due to the data within the scriptsig. This means transaction fees per datasize are lower when compared to other methods. Unspendable outputs, which some methods use to store data by sending dust to an unspendable address, increase the utxo database, which can never be pruned. Utxo database bloat is a cost incurred by all nodes since they must keep this in the fastest accessible location, such as system RAM, whereas p2shdata transactions are fully prunable. And even if nodes choose not to prune, the data is stored on disk as old block files rather than utxo data. Since p2shdata uses transaction inputs to save the data, no dust is created and coins minus transaction fees can be returned to your wallet. Data sizes are currently limited by the maximum transaction size, which is 100 kb. Since OP codes, salt, and other non-data components of a transaction (i.e. the vin(s) & vout(s)) take up approximately 12% of that size, the actual largest data file that can be saved is around 88 kb. Future versions of the protocol may employ multiple transactions which could be stacked together for even larger data files much like the multiple scriptsigs are concatenated.

ASSEMBLY SCRIPT NOTES

The assembly script can be up to 12 bytes (bytes 48 to 60 in the 80 byte op_return output). This script gives the range (first to last) in hex of indexed outputs that contain the data, and provide any specific encoding requirements.

Assembly script example: 05dc00ffec64000000000000
1st byte (05) is the byte size of the script (dc00ffec64), so the right padded zeros can be ignored
Assembly operation example: dc00ffec64
First byte of the assembly (dc) is data location, followed by 2 bytes for first (00) and last (ff) tx input with data
dc00ff will parse every tx input (input 0 to 255) for script data
A data range other than 0 to 255 can be specified if some tx inputs don't contain data
The next byte after data range (ec) indicates data should be encoded in a format other than ascii text
64 indicates the data should be base64 encoded
16 indicates the data should is base16 hex and doesn't need any decoding at all
10 indicates the data should be base10 decimal encoded
f8 indicates the data should be UTF-8 encoded
If there's no ec in the assembly, the default output is to encode the data to ascii text

DATAHASH160 NOTES

The last 20 bytes in the op_return output of the p2shdata transaction contain a hash of the reconstructed data. Hash160 is the bitcoin160 hash type, which is a double hash, first with SHA256, then followed by RipeMD-160 (often called HASH160 or represented as ripemd-160(sha256(data)).) The hash digest contained in the op_return output (op_datahash160) can be compared to the hash160 value calculated at time of data re-assembly (payloadhash160) to ensure data integrity.
