ECOXFR
======

_Econet Transfer_ station-to-station transfer utility for the BBC
Micro, without needing a dedicated fileserver.

It uses full 32-bit addresses for file sizes and offsets so can
transfer files >64KiB.

The speed is fairly good and seems to be about the same as a Level 3
File Server: copying a file from my Master running ANFS 4.25 to my
Model B running either ANFS 4.18 or L3FS 1.25, both running
[PiTubeDirect](https://github.com/hoglet67/PiTubeDirect) JIT 6502 (CPU
#24), it took about 36s to transfer 200KiB on a quiet Econet (to/from a
[Pi1MHz](https://github.com/dp111/Pi1MHz) ADFS hard disc at each end).

Usage
-----

The parameters are all specified on the command line and the same
program is used at both the sending and receiving end.

For example, to send the file `PROJECT1` to station 81, do:

    *ECOXFR 81 PROJECT1

On the receiving station, you can just specify the station number as
`*` to receive from any station and miss out the filename to indicate
receive mode and use the filename from the sender, e.g.:

    *ECOXFR *

You can limit the station from which the file is received by specifying
the number instead of `*` and/or override the name for the file to be
written out as with `=filename` - for example, to receive from station
82 and save out the file as `PROJ1`, regardless of the original name of
the file:

    *ECOXFR 82 =PROJ1

The name send will be truncated to just the leaf part - for example, if
you do:

    *ECOXFR 81 $.PROGRAMS.CODE.THINGY

... the name sent to the receiver will just be `THINGY` and written
into the current directory (unless overridden at the receiving end with
`=filename`).

When transferring, information about the file (name, load, exection,
length) and my/peer station IDs will be displayed, as well as a counter
showing the offset into the file currently being transferred.

The program should cope with transfers to stations on other nets (e.g.
5.100) but this hasn't been verified.

If run without any parameters, it will display some help on usage.

Memory usage
------------

The code runs at &FFFF2200 (which is where PAGE is on my Model B with
DFS, ADFS and ANFS), finishing below &3000, including buffer space.
This means it should be able to run in any screen mode without
corrupting any system or sideways ROM workspace, in any 'reasonable'
configuration.

When a Tube processor is present, the program will run on the host
(I/O) processor so should not corrupt anything in Tube memory (such as
a BASIC program) and will also run regardless of Tube processor type
(so it will run with a Z80 or ARM CPU, for example).

Code
----

The program is written using the BBC BASIC built-in 6502 assembler and
is fairly large, so you'll likely need HIBASIC to assemble it.

The code stored in GitHub is an ASCII dump of the BASIC program
generated with *SPOOL on the BBC and passed through a simple script to
strip out the line numbers and insert `*BASIC`, `NEW` and `AUTO` at
the start - this allows you to simply `*EXEC` the file to create the
BASIC program but also allows git to track the changes between
revisions in the normal way.

Protocol
--------

The transfer protocol is deliberately simple and consists of 3 packets:

1. File information
2. Block request
3. Block send

All sizes and offsets in the packets are sent little endian, matching
the format used on the 6502 and BBC MOS.

### File information

The ***file information*** packet is the first packet transmitted by the
sender - it contains the source port number and file information (with
given offsets from the start of the packet):

1. Operation = &01 (+0, 1 byte)
2. Sender port number = n (+1, 1 byte)
3. Load address (+2, 4 bytes)
4. Execution address (+6, 4 bytes)
5. File length (+10, 4 bytes)
6. Attributes as per OSFILE (+14, 4 bytes)
7. Filename + CR/&0D (+18, 1-11 bytes)

The arrangement of the load/execution/length/attribute data is
deliberately the same as OSFILE operation 5 so that information can be
sent as is.

The maximum size of this packet is 29 bytes, with a maximum filename
length of 10 characters + CR.

This packet is sent to port 77 on the receiver.

### Block request

The ***block request*** packet is transmitted by the receiver back to
the sender, upon receipt of the file information packet (typically
after the file is created on the receiving system, but that is up to
the implementation):

1. Operation = &81 (+0, 1 byte)
2. Offset (+1, 4 bytes)
3. Maximum transfer (+5, 4 bytes)

The _offset_ is the byte position in the file being requested and will
start at &00000000.  The sender expects to transfer each portion of the
file once, in contiguous and sequential order - if the offset sent in
the block request does not match the offset the sender is expecting,
the sender will typically abort with an error.

_Maximum transfer_ is the maximum size of the data portion of the
packet to be returned by the sender in the following packet and
excludes the size of that header.  This can be for more data than
remains in the file, but the sender must not send more than what
remains.

This packet is sent to the _sender port number_ sent in the file
information packet.  ECOXFR uses 78 for this but it could be any valid
port.  This number cannot change during the transfer of any single
file.

### Block send

The ***block send*** packet contains a block of the file and is
transmitted by the sender to the receiver:

1. Operation = &02 (+0, 1 byte)
2. Offset (+1, 4 bytes)
3. Data (+2, <= maximum transfer bytes)

The _offset_ must equal the offset sent in the preceeding block request
packet: if this is not the case, the receiver will typically abort with
an error.

The _data_ will contain a block of data from the file, starting at the
offset position.  This can be of any size up to the maximum transfer
sent in the block request and excludes the size of the header - for
example, if the request asks for a maximum of 100 bytes, the sending
packet can be up to 105 bytes (1 byte for operation, 4 bytes for offset
and 100 bytes for the data).

The cummulative amount of data sent in total cannot exceed the length
of the file sent in the file information block at the start, else the
receiver will abort with an error.  For example, if the file is 2000
bytes long and the offset requested is 1980 with a maximum transfer
size of 100, the data portion of the returned block send packet can
only contain a maximum of 20 bytes.

Like the file information packet, this packet is sent to port number
77.

The protocol then loops back and forth between block request and block
send packets until the file is transferred.  There is no final
'transfer complete' packet sent by the receiver: the final packet will
be the last block of the file in a block send packet.

### Errors

There are no error packets or ways of reporting problems to the peer
station - if an error occurs, the end detecting the problem will
typically abort with a suitable message and leave the peer to time out,
if it is expecting a reply.

The implemented timeout is 5s but this is not specified by the
protocol: this number has been chosen as it seems sensible but the
author has no experience of busy or problematic Econet installations
and if this is appropriate.
