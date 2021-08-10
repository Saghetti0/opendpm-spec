# OpenDPM Specification 02: Vault Format

## Datatypes

The following datatypes are used in this documentation:

| Name    | Size (bytes)       | Description                                                  |
| ------- | ------------------ | ------------------------------------------------------------ |
| byte    | 1                  | A single byte. Equivalent to a uint8.                        |
| uint8   | 1                  | An unsigned 8-bit integer.                                   |
| uint16  | 2                  | An unsigned 16-bit integer.                                  |
| uint32  | 4                  | An unsigned 32-bit integer.                                  |
| string8 | 1 + size of string | A UTF8 string prefixed with a uint8 containing the length of the string. |

## Chunks, Blocks, and HSMs

One of the main goals for OpenDPM is to allow the use of Hardware Security Modules (HSMs). Most HSMs that will be used with OpenDPM are microcontrollers with small amounts of memory and slow processors. OpenDPM is designed to work on microcontrollers that have at least 2KB of RAM, preferably more.

The initial, naïve approach for vault decryption was for the host to send the entire vault (up to 1MB of data) to the HSM. The HSM was then responsible for storing the vault in memory, decrypting it, and responding to queries from the host. This required at least 1MB of RAM to be available on the HSM, which would greatly increase the cost and complexity of HSMs, as high-end microcontrollers or external RAM chips would be required to buffer all the data. This also ruled out using devices like smart cards as HSMs.

The new approach involves breaking the data in a vault into 1KB chunks, which are then further sub-divided into 64 byte blocks. Each chunk is individually encrypted with xchacha20, and authenticated using poly1305. xchacha20-poly1305 doesn't support authenticated random-access decryption, which is why chunks need to be individually encrypted. Chinks exist in order to reduce the overhead of individually encrypting each block. If 64-byte blocks were individually encrypted, the MAC and nonce (20 bytes combined) would mean a ~31% overhead for individual encryption. When using 1KB chunks, this number is now ~1.9%, greatly increasing storage efficiency. 

Using the new approach, HSMs only need to buffer 1 chunk at a time. If the host requests a record from the HSM, the HSM only needs to load, decrypt, and parse the data from 1 chunk at once. For records that span multiple chunks, the HSM can decrypt and send the data from the first chunk, then reuse the memory that the first chunk was using in order to load the second chunk, and so on and so forth. Because the host already received the data, the HSM doesn't need to store it anymore. As the full data for a record can be split across multiple out-of-order chunks, the host is responsible for re-ordering and re-constructing the full record from the fragments that the HSM sends.

### Chunk Format

| Type       | Name        | Description                                                  |
| ---------- | ----------- | ------------------------------------------------------------ |
| byte[16]   | mac         | The poly1305 MAC of this chunk.                              |
| byte[4]    | nonce_lower | The last 4 bytes of the nonce. The first 20 bytes of the nonce are stored at the start of the file. |
| byte[1024] | data        | The 16 encrypted blocks that make up this chunk.             |

The format for blocks is described in the next section.

## Records

A record is an object in an OpenDPM vault, similar to a file or folder in a filesystem. Each record is comprised of a unique identifier, a name, and optional data and metadata that is specific to each type of record. Records are used to store passwords, private keys, small files, etc. Records can also be used for other purposes, such as categories and  A record is stored in one or more blocks, similar to how one or more sectors make a file in a filesystem. More details on records are described in the section.

### Block Format

| Type     | Name       | Description                                                  |
| -------- | ---------- | ------------------------------------------------------------ |
| uint8    | flags      | Flags for this                                               |
| uint8    | block_type | The type of this block. A complete table is available under the **Block Types** section. |
| byte[62] | body       | The body of this block. The contents of the body depends on what type of block this is. If the block's body doesn't fill the full 62 bytes, the extra space should be filled with `0x00` |

## Block Types

In order to reduce the amount of overhead per record, there are multiple different block types. The block type specifies multiple attributes about the block, including the sizes of datatypes in the block, and the data the block contains. 

### 0x00: Null

The null block is a block that has no data, and is used to mark free blocks.

### 0x01: Meta 

The meta block is used to store data about the vault. Only one meta block should ever appear in a vault, and it should be the first block in the first chunk. The meta block should be loaded into memory once when the vault is loaded, and should stay resident in memory for the entire duration the vault is unlocked. The format is as follows:

**Size:** 48 bytes

| Type     | Name            | Description                                                  |
| -------- | --------------- | ------------------------------------------------------------ |
| uint16   | opendpm_version | The version of OpenDPM that this vault is on. The high byte is the major version number, and the low byte is the minor version number. Major versions are versions that introduce breaking changes, whereas minor versions are just additions.<br />The latest version (and the version that this document covers) is `0x0100` |
| uint16   | record_count    | The number of records that are in this vault.                |
| uint32   | next_record_id  | The next free record ID. Code that needs to generate a record ID should use this value and post-increment it. |
| uint16   | chunk_count     | The number of chunks that this vault contains. This number should be at least 1 and at most 1024 |
| uint16   | block_count     | The number of blocks that this vault contains. This number should be at least 1 (for the meta block) and at most 16384. |
| byte[16] | vault_id        | The unique ID that identifies this vault across multiple servers. |
| byte[16] | root_patch_key  | The master key that is used to derive per-server patch keys. The algorithm to get a per-server patch key is as follows:<br />`patch_key = hmac(root_patch_key, server_ip + vault_revision)`<br />Note that both `server_ip` and `vault_revision` are binary. |
| uint32   | vault_revision  | The revision number of this vault. This number is increased by one before the vault is uploaded to servers. |

### 0x10: Linked Block

A linked block describes a block in a doubly linked list, used to store data larger than can fit into a single block. In order to get the complete contents of a chain of linked blocks, the entire chain must be traversed from start to end, and the data for each block must be concatenated in order.

**Size:** 9 bytes + content

| Type       | Name           | Description                                                  |
| ---------- | -------------- | ------------------------------------------------------------ |
| uint32     | parent_record  | The record ID that this block belongs to.                    |
| uint16     | previous_block | The index of the previous block in the list. `0` if this is the first block. |
| uint16     | next_block     | The index of the next block in the list. `0` if this is the last block. |
| uint8      | size           | The size of the contents of this block. Must be greater than `0` and less than `53` |
| byte[size] | contents       | The contents of this block                                   |

### 0x20: Login Record

A login record contains a username, password, and 

**Size:** 

| Type   | Name                   | Description                                                  |
| ------ | ---------------------- | ------------------------------------------------------------ |
| uint32 | record_id              | The ID of this record.                                       |
| uint64 | creation_timestamp     | The creation date of this record, in milliseconds since the Unix Epoch. |
| uint64 | modification_timestamp | The last modification date of this record, in milliseconds since the Unix Epoch. |
| uint16 |                        |                                                              |
| uint32 | record_name[32]        | The name of this record, as a null-terminated string with a max length of 32 characters. |
|        |                        |                                                              |



## Enums

### Record Types



### 0x20 - 0x27: Record Descriptor

Every record descriptor has the following b

| Type | Name | Description |
| ---- | ---- | ----------- |
|      |      |             |

#### Bit 0: 







| Type | Name | Description |
| ---- | ---- | ----------- |
|      |      |             |

