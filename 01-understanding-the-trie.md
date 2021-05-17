Ethereum is a big virtual machine whose state all clients agree on. The state is big and a lot to handle for a node, so efforts to build stateless clients (clients who don't have to hold the whole state, or any of it) are underway.

To get a full understanding of the problem that needs to be solved, we should know what the state is and how it is stored. Basically, it is the set of all accounts (and their balances, storage and contract code) and it is stored in a data structure called a hexary trie.

This will be a top-down, line-by-line tentative of understanding the trie starting from [py-trie](https://github.com/ethereum/py-trie), a trie implementation written in Python specifically for Ethereum. It's not meant to be a "tutorial", but maybe it can be a way to answer questions someone else might have when doing the same thing.

### What you should know

- Concepts of [tries](https://en.wikipedia.org/wiki/Trie), [hashing](https://en.wikipedia.org/wiki/Hash_function), [hex encoding](https://en.wikipedia.org/wiki/Hexadecimal) and maybe [binary trees](https://en.wikipedia.org/wiki/Binary_tree)
- How to read basic Python code

### What you will learn
- How Ethereum stores and encodes data in a trie

### Optional reading

- [Ethereum State Trie Architecture Explained](https://medium.com/@eiki1212/ethereum-state-trie-architecture-explained-a30237009d4e), by **@eiki1212**

## Setup

Install all the required software:

```bash
pip install eth-utils rlp trie
```

Then, launch a Python interpreter.

## An empty hexary trie

Let's initialize an empty hexary trie:

```python
>>> from trie import HexaryTrie
>>> t = HexaryTrie(db={})
```

- **What's a hexary trie?** — Think of what the name suggests: while in a _binary_ trie every node has _at most two children_, in a **hexary** trie every node can have **up to 16 children**.

Let's see how an empty hexary trie looks like.

```python
>>> t
HexaryTrie({}, root_hash=b'V\xe8\x1f\x17\x1b\xccU\xa6\xff\x83E\xe6\x92\xc0\xf8n[H\xe0\x1b\x99l\xad\xc0\x01b/\xb5\xe3c\xb4!', prune=False)
```

Well it doesn't look _completely_ empty, it has a **root hash**.

- **What's the root hash?** — The hash of all the hashes of all the leaf nodes (which basically hold the data we are trying to store: in the case of Ethereum, that would be accounts, transactions, etc.)
Yet, we have no data stored in the trie. So the root hash, at this point, is just **the hash of nothing, i.e. an empty byte string**.
- **How do I hash the empty byte string, in practice?** — Ethereum uses `keccak256` as a hash function. You can try hashing an empty byte string:

```python
>>> from eth_utils import keccak
>>> keccak(b'')
b"\xc5\xd2F\x01\x86\xf7#<\x92~}\xb2\xdc\xc7\x03\xc0\xe5\x00\xb6S\xca\x82';{\xfa\xd8\x04]\x85\xa4p
```

- **Ugh, what is that ugly thing and how do I read it?** — Welcome to Python's representation of binary data. Binary data is usually displayed as `\x` followed by a hex-encoded byte. Examples:

```python
>>> bytes([0])
b'\x00' # hex: 00
>>> bytes([15])
b'\x0f' # hex: 0f
>>> bytes([9])
b'\t' # WTF? should be hex: 09, or b'\x09'
```

Here's the thing: if a byte can be displayed as an ASCII character, it will be, and ASCII character #9 is a horizontal tab delimiter (check out an ASCII table [here](https://www.cs.cmu.edu/~pattis/15-1XX/common/handouts/ascii.html)):

```python
>>> chr(0)
'\x00' # not a printable ASCII character
>>> chr(9)
'\t'
```

To get a more readable hex representation of your data, just call `.hex()` on your byte string:

```python
>>> bytes([0]).hex()
'00'
>>> bytes([15]).hex()
'0f'
>>> bytes([9]).hex()
'09'
```

So now that we know what we're looking at, let's go back to where we were: a `keccak`-hashed empty byte string, which should be the root hash of our trie. Let's see if that is the case:

```python
>>> keccak(b'').hex()
'c5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470'
>>> t.root_hash.hex()
'56e81f171bcc55a6ff8345e692c0f86e5b48e01b996cadc001622fb5e363b421'
>>> keccak(b'').hex() == t.root_hash.hex()
False
```

What's going on here? It looks like **the trie root hash is not a `keccak`-hashed empty byte string**.

- **Have you been lying to me?** — no, not really. Welcome to **RLP (Recursive Length Prefix) encoding**.

RLP encoding is an Ethereum-specific way of encoding binary data. This is all you need to know for now, together with the fact that it's used only to encode the values stored in the trie (and not the "keys", i.e. the path that is taken to get to the values).

Optional reading on RLP encoding if you want to know more:

- [Data structure in Ethereum | Episode 1: Recursive Length Prefix (RLP) Encoding/Decoding.](https://medium.com/coinmonks/data-structure-in-ethereum-episode-1-recursive-length-prefix-rlp), by Phan Sơn Tự
- [RLP entry on the official Ethereum wiki](https://eth.wiki/en/fundamentals/rlp)

What we care about is that there's an easy way of encoding data into RLP.

```python
>>> import rlp
>>> rlp.encode(b'')
b'\x80'
>>> keccak(rlp.encode(b'')).hex()
'56e81f171bcc55a6ff8345e692c0f86e5b48e01b996cadc001622fb5e363b421'
>>> keccak(rlp.encode(b'')).hex() == t.root_hash.hex()
True
```

Yes! So the root hash of an empty `HexaryTrie` is an **RLP-encoded, keccak-hashed empty byte string**.

### Adding data to the trie

Let's examine the root node of our blank tree:

```python
>>> t.root_node
HexaryTrieNode(
    sub_segments=(),
    value=b'',
    suffix=(),
    raw=b'',
    node_type=<NodeType.BLANK: 0>
)
```

Nothing to see here, just an empty node. Note that the type is `BLANK`, quite self-explanatory.

Let's try adding some data:

```python
>>> t.set(b'blue', rlp.encode(b"I'm blue, da buh dee ba buh daa"))
>>> t.root_node
HexaryTrieNode(
    sub_segments=(),
    value=b"\x9fI'm blue, da buh dee ba buh daa",
    suffix=(0x6, 0x2, 0x6, 0xc, 0x7, 0x5, 0x6, 0x5),
    raw=[b' blue', b"\x9fI'm blue, da buh dee ba buh daa"], node_type=<NodeType.LEAF: 1>
)
```

The node is now a leaf node. A leaf node is a node that holds a single value and sits at the bottom of our trie, that is now looking like this:

```
    (root)
      |
      |
--LEAF-(blue)--
|   I'm blue  |
|   da buh..  |
---------------
```

The suffix is the "remaining" part of the path that is used to reach the leaf value (i.e. it does not belong to any intermediary nodes). Since we only have a leaf node, this is the full (and only) path.

Notice how:

```python
>>> b'blue'.hex()
'626c7565'
>>> t.root_node.suffix
(0x6, 0x2, 0x6, 0xc, 0x7, 0x5, 0x6, 0x5)
```

The path of our leaf node is the hex-encoded key. But if you look closely, the "raw" value is `b' blue'`.

```python
>>> t.root_node.raw[0]
b' blue'
```

What's going on?

```python
>>> t.root_node.raw[0].hex()
'20626c7565'
```

Looks like there is one extra byte (`20`) in front of our hex-encoded `b'blue'`. In Ethereum, trie paths are encoded using Hex Prefix encoding. Basically, hex values from 0 to 3 are used to indicate whether the path length is even or odd, and whether the node type is a leaf or not.

I could try to explain HP encoding, but there's already a good article about it. Please read it before continuing!

- [Data structure in Ethereum | Episode 1+: Compact (Hex-prefix) encoding.](https://medium.com/coinmonks/data-structure-in-ethereum-episode-1-compact-hex-prefix-encoding-12558ae02791), by Phan Sơn Tự

So, now we know that the `2` was added in front of our path to indicate that ours is a **leaf node** of **even length** (`len(b'blue'.hex()) % 2 == 0`). A `0` was then added as padding after the `2`, to maintain the even length of the path.

Let's try adding some more data:

```python
>>> t.set(b'black', b'I\'m the old orange')
>>> t.root_node
HexaryTrieNode(
    sub_segments=((0x6, 0x2, 0x6, 0xc),)
    value=b''
    suffix=()
    raw=[b'\x00bl', b'\x90ZT\xfdx\x06?\xb1b\xdd\xc38\xe7\x99\x00\xf3\xd6xh\x0fR\xf4c\xce\x84\x82\xeb\xbaC\xde\xaa\xbb']
    node_type=<NodeType.EXTENSION: 2>
)
```

Our root node is now an extension node. Think of an extension node as a compact way to hold path information that is shared by different leaf nodes.

```python
>>> t.root_node.sub_segments
((0x6, 0x2, 0x6, 0xc),)
>>> b'bl'.hex()
'626c'
```

See? Our extension node is holding `bl`, the common path between `blue` and `black`. Let's go one layer deeper:

```python
>>> t.traverse((0x6, 0x2, 0x6, 0xc))
HexaryTrieNode(
    sub_segments=((0x6,), (0x7,)),
    value=b'',
    suffix=(),
    raw=[b'', b'', b'', b'', b'', b'', [b'1ck', b"I'm the old orange"], b'\xb4\xaf\x86f\xee\xf1\x84006i\xc9e[\xc0\xa5_`!w\x9e"\x15\x81B\xb9\x83\xfcs\xfa\x9f\xb4', b'', b'', b'', b'', b'', b'', b'', b'', b''],
    node_type=<NodeType.BRANCH: 3>
)
```

Next, there's a branch node. A branch node contains up to 16 "pointers" to other nodes that share its same prefix (`626c` in our case). By traversing one step further, we can find our leaf nodes:

```python
>>> t.traverse((0x6, 0x2, 0x6, 0xc, 0x7))
HexaryTrieNode(
    sub_segments=()
    value=b"\x9fI'm blue, da buh dee ba buh daa"
    suffix=(0x5, 0x6, 0x5)
    raw=[b'5e', b"\x9fI'm blue, da buh dee ba buh daa"]
    node_type=<NodeType.LEAF: 1>
)
```

and

```python
>>> t.traverse((0x6, 0x2, 0x6, 0xc, 0x6))
HexaryTrieNode(
    sub_segments=()
    value=b"I'm the old orange"
    suffix=(0x1, 0x6, 0x3, 0x6, 0xb)
    raw=[b'1ck', b"I'm the old orange"]
    node_type=<NodeType.LEAF: 1>
)
```

This is how our trie looks now:

```
                  (root)
                    |
                    |
              -----EXT------
              |    "bl"    |
              |  (0x626c)  |
              --------------
                    |
    --------------BRANCH----------------------
    0|1|2|3|4|5| 6 | 7 |8|9|a|b|c|d|e|f|value|
                 /   \
                /     \
               /       \
   -LEAF-(0x1636b)-  -LEAF-(0x565)--
   |   I'm blue  |   |   I'm the   |
   |   da buh..  |   |   old or..  |
   ---------------   ---------------
   (0x626c61636b)      (0x626c7565)
```

And again in HP encoding:

```
                  (root)
                    |
                    |
                 0x00626c
                    |
    ------------------------------------------
    0|1|2|3|4|5| 6 | 7 |8|9|a|b|c|d|e|f|value|
                 /   \
                /     \
               /       \
   ---(0x31636b)--   --(0x3565)-----
   |   I'm blue  |   |   I'm the   |
   |   da buh..  |   |   old or..  |
   ---------------   ---------------
   (0x626c61636b)      (0x626c7565)
```
