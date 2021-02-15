---
layout: post
title: "Exploring HTTP 2: HPACK"
---

The first installment in Exploring HTTP/2 series introduces HTTP/2 protocol, and this post will describe protocol HPACK for compressing HTTP headers.

### 1. Fundaments
The basic principles of HPACK are: indexing tables and Huffman coding. During compressing (encoding) and decompressing (decoding), it can store specified headers fields (including field names and values) to indexing tables. Every item in theses tables consists of an index (an integer), name and value. If a header field is already in the tables, the encoding just uses the index instead of the field itself, and of course the decoding searches the field from the table by the given index. However, Huffman coding could be used for compressing the headers not in the indexing tables.

#### 1.1 Indexing table
The indexing table is combined by two sub tables: the static table and the dynamic table. The static table is pre-defined by HPACK protocol. It contains 61 the most popular headers, though the values of the most of the headers are empty. The static table is read-only that the items and their positions (indices) cannot be changed. The appendix A in HPACK protocol lists all items in the static table. The structure of the dynamic table is similar to that in static table, but the items are not fixed. In practice, it allows to insert and remove items from the dynamic table.

HPACK protocol requires the both of the indexing tables share the same address space. And the first 61 addresses are allocated to the static table, so the index of the first item in the dynamic table is 62 in the whole indexing table. 

Although the protocol requires the static table and the dynamic table should be combined, each protocol implementation can use different data structures to store the items. For example, the static table could be in a immutable array, however the dynamic table just be managed by a mutable list.

It's need to be highlighted that the items in the dynamic table can be duplicated.

#### 1.2 Huffman coding
Huffman coding is a lossless data compression based on weight path length. It's known that Huffman algorithm requires a table containing all symbols with given Huffman codes. With the statistics on a large sample of HTTP headers, the appendix B in the HPACK protocol provides this code table.

It has to be aware of that HPACK doesn't require to compress the headers by neither indexing table nor Huffman coding. In practices, the encoding can simply covert the header texts to bytes without any compressing.

### 2. Primitive type representations
HPACK protocol uses only two primitive types: unsigned variable-length integers and strings of octets. This protocol uses integer to represent name indexes, header field indexes, or string lengths. The numbers in header names or values just be treated as string.

#### 2.1 Integer representation
HPACK doesn't simply represent an integer by using its binary format. This protocol requires that the representation should end at the last bit in a octet. For instance, representing 127. If the representation starts from the first bit of a octet, it must end at the last bit of the octet, like the below,

```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| 0 | 1 | 1 | 1 | 1 | 1 | 1 | 1 |
+---+---+---+---+---+---+---+---+
```
If the first bit is occupied by another value ("?" for it), the representation on 127 has to start from the second bit of this octet, but one octet is still enough, like the below,
```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| ? | 1 | 1 | 1 | 1 | 1 | 1 | 1 |
+---+---+---+---+---+---+---+---+
```
But how about if it starts from the third bit? Obviously, one more octet is needed. Could it be represented as the following format?
```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| ? | ? | 1 | 1 | 1 | 1 | 1 | 1 |
+---+---+---+-------------------+
| 1 | ? | ? | ? | ? | ? | ? | ? |
+---+---+---+---+---+---+---+---+
```
It absolutely isn't compliant to the protocol due to the above representation doesn't end at the last bit of the second octet. So, HPACK designs a delicate representation, as shown as the below,
```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| ? | ? | 1 | 1   1   1   1   1 |
+---+---+---+-------------------+
| 1 |    Value-(2^N-1) LSB      |
+---+---------------------------+
               ...
+---+---------------------------+
| 0 |    Value-(2^N-1) MSB      |
+---+---------------------------+
```
The count of bits in the first octet used for integer representation is called prefix (it's 6 in the above case). The below is the implementation in Java.
```
public void encodeInteger(int value, int prefix) {
    if (value >> prefix <= 0) {
        printBinary(value);
    } else {
        int number = (1 << prefix) - 1;
        printBinary(number);
        for (value -= number; value >= 128; value /= 128) {
            printBinary(value % 128 + 128);
        }
        printBinary(value);
    }
}

private void printBinary(int value) {
    System.out.println(String.format("%8s", Integer.toBinaryString(value)).replace(" ", "0"));
}
```
According to the above algorithm, if prefix is 6, the representation of 127 is the below,
```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| ? | ? | 1 | 1 | 1 | 1 | 1 | 1 |
+---+---+---+-------------------+
| 0 | 1 | 0 | 0 | 0 | 0 | 0 | 0 |
+---+---+---+-------------------+
```

#### 2.2 String representation
```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| H |    String Length (7+)     |
+---+---------------------------+
|  String Data (Length octets)  |
+-------------------------------+
```
The above diagram illustrates the representation on string. It consists of three parts:
* Huffman flag (H): It indicates if the string is Huffman codes. The length is 1-bit.
* Length: An integer indicating the length of the string. It applies the representation for integer stated in section 2.1, and the prefix value is 7.
* String data: If the Huffman flag is 0, the value is original string bytes; otherwise, it's the result encoded by Huffman coding. Since the Huffman-encoded data may not end at the last bit of a octet, it has to add some EOS (end-of-string) symbols as padding.

### 3. Header field representations
Now it can represent header fields based on the representations described in section 2. HPACK protocol categorizes the representations to 4 different types. The leading variable-length bits in each representation are used to indicate the types.

#### 3.1 Indexed header field representation
The type flag needs 1 bit, and the value is 1. The index applies the integer representation with prefix 7. The decoding doesn't update the dynamic table.
```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| 1 |        Index (7+)         |
+---+---------------------------+
```

#### 3.2 Literal header field with incremental Indexing
The type flag needs 2 bits, and the value is 01. The decoding will insert new items into the dynamic table. However there're two cases:
* The header field name is already in the indexing table. The field name applies integer representation with prefix 6, and field value applies string literal representation.
```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| 0 | 1 |      Index (6+)       |
+---+---+-----------------------+
| H |     Value Length (7+)     |
+---+---------------------------+
| Value String (Length octets)  |
+-------------------------------+
```
* The header field name isn't in the indexing table. The both of field name and value are expressed as string literals. And the trailing 6 bits in the first octet are filled with 0. 
```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| 0 | 1 |           0           |
+---+---+-----------------------+
| H |     Name Length (7+)      |
+---+---------------------------+
|  Name String (Length octets)  |
+---+---------------------------+
| H |     Value Length (7+)     |
+---+---------------------------+
| Value String (Length octets)  |
+-------------------------------+
```
#### 3.3 Literal header field without indexing
The type flag needs 4 bits, and the value is 0000. The decoding doesn't insert new items into the dynamic table. And there're also two cases:
* The header field is already in the indexing table. The field name applies integer representation with prefix 4, and field value applies string literal representation.
```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| 0 | 0 | 0 | 0 |  Index (4+)   |
+---+---+-----------------------+
| H |     Value Length (7+)     |
+---+---------------------------+
| Value String (Length octets)  |
+-------------------------------+
```
* The header field name isn't in the indexing table. The both of field name and value are expressed as string literals. And the trailing 4 bits in the first octet are filled with 0. 
```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| 0 | 0 | 0 | 0 |       0       |
+---+---+-----------------------+
| H |     Name Length (7+)      |
+---+---------------------------+
|  Name String (Length octets)  |
+---+---------------------------+
| H |     Value Length (7+)     |
+---+---------------------------+
| Value String (Length octets)  |
+---+---------------------------+
```
#### 3.4 Literal header field never indexed
The type flag needs 4 bits, and the value is 0001. The decoding doesn't insert new items into the dynamic table. And there're also two cases:
* The header field is already in the indexing table. The field name applies integer representation with prefix 4, and field value applies string literal representation.
```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| 0 | 0 | 0 | 1 |  Index (4+)   |
+---+---+-----------------------+
| H |     Value Length (7+)     |
+---+---------------------------+
| Value String (Length octets)  |
+-------------------------------+
```
* The header field name isn't in the indexing table. The both of field name and value are expressed as string literals. And the trailing 4 bits in the first octet are filled with 0. 
``` 
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| 0 | 0 | 0 | 1 |       0       |
+---+---+-----------------------+
| H |     Name Length (7+)      |
+---+---------------------------+
|  Name String (Length octets)  |
+---+---------------------------+
| H |     Value Length (7+)     |
+---+---------------------------+
| Value String (Length octets)  |
+-------------------------------+
```
It can find that the representations in section 3.2 and 3.3 are the same, except for the type flags. What's the key difference between them? The header fields represented by type 0000 would be inserted into the indexing table at an intermediate encoding or decoding. However, the representation type 0001 ensures that the fields always cannot be inserted into the indexing table.

### 4. The dynamic table management
It can calculate the size of an item in the dynamic table, the expression is: `the length of field name octets + the length of field value octets + 32`. The lengths are the lengths of original field name and value octets, but not the lengths of Huffman-encoded string octets.

The size of the dynamic table is the sum of the size of its items. The size of the dynamic table is limited. The below integer representation can be used to inform the protocol implementations to change the size of the dynamic table.
```
  0   1   2   3   4   5   6   7
+---+---+---+---+---+---+---+---+
| 0 | 0 | 1 |   Max size (5+)   |
+---+---------------------------+
```
Inserting a new item or resizing the dynamic table may result in one or more items are removed, and even clearing the whole dynamic table. It allows to resize the dynamic table to 0. In fact, this is a general way to reset this table.

