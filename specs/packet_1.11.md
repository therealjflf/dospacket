```
Revision:      1.11
Release date:  Thursday, June 02, 1994
Print date:    Thursday, June 02, 1994
Developed by:  FTP Software
               2 High Street
               North Andover, MA 01845
               (508) 685-4000
```

Copyright (c) 1986-1994 FTP Software, Inc. Permission is granted to reproduce and distribute this document without restriction or fee. This document may be re-formatted or translated, but the functional specification of the programming interface described herein may not be changed without FTP Software's permission. FTP Software's name and this notice must appear on any reproduction of this document. This specification was originally developed at FTP Software by John Romkey.

Support of a hardware interface or mention of an interface manufacturer by the Packet Driver Specification does not necessarily indicate that the manufacturer endorses this specification.


# Document Conventions

All numbers in this document are given in C-style representation. Decimal is expressed as 11, hexadecimal is expressed as 0x0B, octal is expressed as 013. All reference to network hardware addresses (source, destination and multicast) and demultiplexing information for the packet headers assumes they are represented as they would be in a MAC-level packet header being passed to the send_pkt() function.


# Introduction and Motivation

This document describes the programming interface to FTP Software Packet Drivers. Packet drivers provide a simple, common programming interface that allows multiple applications to share a network interface at the data link level. The packet drivers demultiplex incoming packets among the applications by using the network media's standard packet type or service access point field(s).

The intent of this specification is to allow protocol stack implementations to be independent of the actual brand or model of the network interface in use on a particular machine. Different versions of various protocol stacks still must exist for different network media (Ethernet, 802.5 ring, serial lines), because of differences in protocol-to-physical address mapping, header formats, maximum transmission units (MTUs) and so forth.

The packet driver provides calls to initiate access to a specific packet type, to end access to it, to send a packet, to get statistics on the network interface and to get information about the interface.

Protocol implementations that use the packet driver can completely coexist on a PC and can make use of one another's services, whereas multiple applications which do not use the driver do not coexist on one machine properly. Through use of the packet driver, a user could run TCP/IP, XNS, and a proprietary protocol implementation such as DECnet, Banyan's, LifeNet's, Novell's or 3Com's without the difficulties associated with preempting the network interface.

Applications which use the packet driver can also run on new network hardware of the same class without being modified; only a new packet driver need be supplied.

Several levels of packet drivers are described in this specification. The first is the basic packet driver, which provides minimal functionality but should be simple to implement and which uses very few host resources. The basic driver provides operations to broadcast and receive packets. The second driver is the extended packet driver, which is a superset of the basic driver. The extended driver supports less commonly used functions of the network interface such as multicast, and also gathers statistics on use of the interface and makes these available to the application. The third level, the high- performance functions, support performance improvements and tuning.

Functions which are available in only the extended packet driver are noted as such in their descriptions. All basic packet driver functions are available in the extended driver. The high-performance functions may be available with either basic or extended drivers.


# Identifying network interfaces

Network interfaces are named by a triplet of integers, `<class, type, number>`. The first number is the class of interface. The class tells what kind of media the interface supports: DEC/Intel/Xerox (DIX or Bluebook) Ethernet, IEEE 802.3 Ethernet, IEEE 802.5 Token Ring, ProNET-10, Appletalk, serial line, etc.

The second number is the type of interface: this specifies a particular instance of an interface supporting a class of network medium. Interface types for Ethernet might name these interfaces: 3Com 3C503 or 3C505, InterLan NI5210, Univation, BICC Data Networks ISOLAN, Ungermann-Bass NIC, etc. Interface types for IEEE 802.5 might name these interfaces: IBM Token Ring adapter, Proteon p1340, etc.

The last number is the interface number. This was orginally intended to allow support of multiple interfaces by a single Packet Driver. However, given that the send_pkt() function doesn't get passed either an interface number or a handle, this can't be implemented in the spec as it presently exists.

An appendix details constants for classes and types. The class of an interface is an 8-bit integer, and its type is a 16 bit integer. Class and type constants are managed by FTP Software. Contact FTP to register a new class or type number. See the section on Specific Characteristics of Classes for details of packet header format, MTU and demultiplexing information for the various classes.

The type 0xFFFF is a wildcard type which matches any interface in the specified class. It is unnecessary to wildcard interface numbers, as 0 will always correspond to the first interface of the specified class and type.

This specification has no provision for the support of multiple network interfaces (with similar or different characteristics) via a single Packet Driver and associated interrupt. We feel that this issue is best addressed by loading several Packet Drivers, one per interface, with different interrupts (although all may be included in a single TSR software module). Applications software must check the class and type returned from a driver_info() call in any case, to make sure that the Packet Driver is for the correct media and packet format. This can easily be generalized by searching for another Packet Driver if the first is not of the right kind.


# Initiating driver operations

The packet driver is invoked via a software interrupt in the range 0x20 through 0xFF (in versions prior to 1.10, the low and high limits were 0x60 and 0x80, respectively). This document does not specify a particular interrupt, but describes a mechanism for locating which interrupt the driver uses. The interrupt must be configurable to avoid conflicts with other pieces of software that also use software interrupts. The program which installs the packet driver should provide some mechanism for the user to specify the interrupt.

The handler for the interrupt is assumed to start with 3 bytes of executable code; this can either be a 3-byte jump instruction, or a 2-byte jump followed by a NOP (do not specify "jmp short" unless you also specify an explicit NOP). This must be followed by the null-terminated ASCII text string

```
PKT DRVR  (ASCII: 0x50 0x4A 0x54 0x20 0x44 0x52 0x56 0x52 0x00)
```

To find the interrupt being used by the driver, an application should scan through the handlers for vectors 0x20 through 0xFF until it finds one with the text string "PKT DRVR" in the proper location.


# Link-layer demultiplexing

If a network media standard is to support simultaneous use by different transport protocols (e.g. TCP/IP, XNS, OSI), it must define some link-level mechanism which allows a host to decide which protocol a packet is intended for. In DIX Ethernet, this is the 16-bit Ethertype field immediately following the 6-byte destination and source addresses. In IEEE 802.3 where 802.2 headers are used, this information is in the variable-length 802.2 header. In Proteon's ProNET-10, this is done via the 8-bit "type" field. Other media standards may demultiplex via a method of their own, or 802.2 headers as in 802.3.

Our access_type() function provides access to this link-layer demultiplexing. Each call establishes a destination for a particular type of link-layer packet, which remains in effect until release_type() is called with the handle returned by that particular access_type(). The link-layer demultiplexing information is passed via the type and typelen fields, but values and interpretation depend on the class of packet driver (and thus on the media in use).

A class 1 driver (DIX Ethernet) should expect type to point at an Ethertype value (in network byte order, and greater than 0x05EE), and might reasonably require typelen to equal either 2 (a 16-bit Ethertype) or 0 (if it supports match all types mode). A class 2 driver could require 1 byte. However, a class 3 (802.5) or 11 (Ethernet with 802.2 headers) driver should be prepared for typelen values between 1 (for the DSAP field only) and 8 (3 byte 802.2 header plus 3-byte Sub-Network Access Protocol extension header plus 2-byte Ethertype as defined in RFC 1042).


# Programming interface

All functions are accessed via the software interrupt determined to be the driver's via the mechanism described earlier. On entry, register AH contains the code of the function desired.

Driver implementors are warned that a number of protocol stacks exist that support multiple active network interfaces simultaneously, notably Phil Karn's KA9Q, which can function as an IP Router. For this reason it is necessary to use coding techniques which will allow multiple drivers (possibly for the same type of board) to be active at once. At a minimum, this requires that the driver's vector be configurable, and that it avoid issuing non-specific EOIs to the interrupt controller.


## Handles

The handle is an arbitrary 16-bit integer value associated with each MAC-level demultiplexing type that has been established via the access_type() call. Internally to the packet driver, it will probably be a pointer, or a table offset.

As of version 1.10 of this specification, the handle values 0x0000 and 0xFFFF (-1) has been reserved for use by this specification (and applications) as an "invalid handle" indication. There is reason to believe that no older Packet Drivers will ever return these handles, because existing protocol stacks already use one or the other as an "invalid handle" indication.

Other than the reserved values named above, the application calling the packet driver cannot depend on a handle assuming any particular range, or any other characteristics. In particular, if an application uses two or more packet drivers, handles returned by different drivers for the same or different types may have the same value.


## Levels of Functionality

Note that some of the functions defined below are labelled as extended driver functions and high- performance functions. Because these are not required for basic network operations, their implementation may be considered optional. Programs wishing to use these functions should use the driver_info() function to determine if they are available in a given packet driver.


## Entry conditions

FTP Software applications which call the packet driver are coded in Microsoft C and assembly language. All necessary registers are saved by FTP's routines before the INT instruction to call the packet driver is executed. Our current receiver() functions behave as follows: DS, BP, SS, SP and the flags are saved and restored. All other registers may be modified, and should be saved by the packet driver, if necessary. Processor interrupts may be enabled while in the upcall, but the upcall doesn't assume interrupts are disabled on entry. On entry, receiver() switches to a local stack, and switches back before returning.

Note that some older versions of PC/TCP may enable interrupts during the upcall, and leave them enabled on return to the Packet Driver. Note also that while some drivers (notably the Clarkson/Crynwr collection) save all registers on entry, others don't. Application developers should take care save their own registers to ensure interoperability.


## Number of Handles Opened

When using a class 1 driver, PC/TCP will normally make 5 access_type() calls for IP, ARP and 3 kinds of Berkeley Trailer encapsulation packets (LANWatch will only make one, with typelen equal to 0). On other media, the number of handles we open will vary, but it is usually at least two (IP and ARP). Implementors should make their tables large enough to allow two protocol stacks to co-exist. We recommend support for at least 10 open handles simultaneously.


## Byte and Bit ordering

Developers should note that, on many networks and protocol families, the byte-ordering of 16-bit and 32-bit quantities on the network is different from the native byte-order of integers on the PC. This means, among other things, that DEC-Intel-Xerox Ethertype values passed to access_type() must be byte-swapped (passed in network order). The IEEE 802.3 length field needs similar handling, and care should be taken with packets passed to send_pkt(), so all fields are in the proper order. Developers working with 802.5 should be aware that it does not use the same bit-order as Ethernet - where the 'group' bit is the most significant bit of a MAC address on 802.5, it is the least significant bit of the first byte on Ethernet. IEEE Organizationally Unique Identifiers (Ethernet manufacturer codes) also change bit-order between the two media. FDDI uses the same bit- order as 802.5 on the physical medium, but current practice requires that it be converted to match Ethernet by the MAC-layer driver before presentation to protocol stacks.


## Byte Alignment

Because some packet drivers can provide greater performance if packet buffers are double word aligned, protocol stack developers should double word align the packet buffers. However this is not an absolute requirement.


## driver_info()

```
driver_info(handle)
    AH      1
    AL      255
    BX      handle      /* Optional */

error return:
    carry   set
    DH      error code

possible errors:
    BAD_HANDLE          /* older drivers only */

non-error return:
   carry    clear
   BX       version
   CH       class
   DX       type
   CL       number
   DS:SI    name
   AL       functionality
                1 == basic functions present.
                2 == basic and extended present.
                5 == basic and high-performance.
                6 == basic, high-performance, extended.
                255 == not installed.
```

This function returns information about the interface. The version is assumed to be an internal hardware driver identifier. In earlier versions of this spec, the handle argument (which must have been obtained via access_type()) was required. It is now optional, but drivers developed according to versions of this spec previous to 1.07 may require it, so implementers should take care.


## access_type()

```
int access_type(if_class, if_type, if_number, type, typelen, receiver)
    AH      2
    AL      if_class
    BX      if_type
    DL      if_number
    DS:SI   type
    CX      typelen
    ES:DI   receiver

error return:
    carry   set
    DH      error code

possible errors:
    NO_CLASS
    NO_TYPE
    NO_NUMBER
    BAD_TYPE
    NO_SPACE
    TYPE_INUSE

non-error return:
    carry   clear
    AX      handle

receiver call:
    (*receiver)(handle, flag, len [[, lah_len], buffer])
        BX  handle
        AX  flag

        if AX == 0,
            DX      lah_len
            DS:SI   lah_buffer      /*  if DX != 0  */

Returns:
    ES:DI   buffer          /*  If 0, no buffer  */
    CX      buffer_len      /*  May not equal entry CX  */

    if AX == 1,
        DS:SI   buffer
        CX      buffer_len      /* Bytes actually copied */

    error return:
        ES:DI   0:0

    non-error return:
        ES:DI   buffer pointer
        CX      buffer length
```

Initiates access to packets of the specified type. The argument type is a pointer to a packet type specification. The argument typelen is the length in bytes of the type field. The argument receiver is a pointer to a subroutine which is called when a packet is received. If the typelen argument is 0, this indicates that the caller wants to match all packets (match all requests may be refused by packet drivers developed to conform to versions of this spec previous to 1.07).

When a packet is received, receiver is called twice by the packet driver. The first time it is called to request a buffer from the application to copy the packet into. AX == 0 on this call. The application may return a pointer to a buffer to which the packet should be copied by the driver in ES:DI. If the application has no buffers, it may return 0:0 in ES:DI, and the driver should throw away the packet and not perform the second call.

It is important that the packet length (CX) be valid on the AX == 0 call, so that the receiver can allocate a buffer of the proper size. This length (as well as the copy performed prior to the AX == 1 call) must include the MAC header and all received data, but not the trailing Frame Check Sequence (if any), except in the case of asynchronous PPP, where space for the Frame Check Sequence may be included, even though the protocol stack will make no use of the information. The driver must never copy more bytes than CX specifies; if the hardware requires that the copy length be even, the initial CX value must be rounded up to reflect this. On the second call, AX == 1. This call indicates that the copy has been completed, and the application may do as it wishes with the buffer. The buffer that the packet was copied into is pointed to by DS:SI. As discussed in the "Entry Conditions" section, the driver is expected to save the registers it cares about before calling receiver(). Protocol stacks should not expect more than about 20 bytes of stack to be available, and must switch to a stack of its own before using more, or enabling interrupts.

Version 1.10 of this spec (use the get_parameters() function to determine the version of a given driver), specifies three enhancements to the receiver mechanism: First, on the AX == 0 call, a non-zero value in DX is the length of the data in a "look-ahead buffer" pointed to by DS:SI. The application may inspect this data to determine where to put the packet. Second, on the return from the AX == 0 call, the value of CX is the actual length of the application's buffer. The driver must truncate the packet to fit as it copies it. Third, on the AX == 1 upcall, CX contains the number of bytes actually copied by the driver.

Note that the functionality defined in the paragraph above must be present in drivers which indicate version 1.10 or greater in the values returned by get_parameters(), and will not be found in drivers indicating 1.09 or previous. It is advisable not to be completely dependent on the presence of a 1.10 driver, given the number of older drivers in the field.

Note that many drivers are not re-entrant, and so are unable to process functions (particularly send_pkt()) while either receiver() upcall is in progress. Queue any transmits or other packet driver calls for execution later, possibly via hooking the int_num vector returned by get_parameters().


## release_type()

```
int release_type(handle)
    AH      3
    BX      handle

error return:
    carry   set
    DH      error code

possible errors:
    BAD_HANDLE

non-error return:
    carry   clear
```

This function ends access to packets associated with a handle returned by access_type(). The handle is no longer valid.


## send_pkt()

```
int send_pkt(buffer, length)
    AH      4
    DS:SI   buffer
    CX      length

error return:
    carry   set
    DH      error code

possible errors:
    CANT_SEND

non-error return:
    carry   clear
```

Transmits length bytes of data, starting at buffer. The application must supply the entire packet, including local network headers. Any MAC or LLC information in use for packet demultiplexing (e.g. the DEC-Intel-Xerox Ethertype) must be filled in by the application as well. This cannot be performed by the driver, as no handle is specified in a call to the send_packet() function.

Note that when transmitting to a group destination MAC address (broadcast or multicast), the hardware may receive the packet itself. Protocol stack implementors are advised to check for this in situations where the stack would otherwise be confused by seeing a copy of its own broadcast (e.g. RARP or broadcast name-defense schemes).


## terminate()

```
terminate(handle)
    AH      5
    BX      handle

error return:
    carry   set
    DH      error code

possible errors:
    BAD_HANDLE          /* older drivers only */
    CANT_TERMINATE

non-error return:
    carry   clear
```

Terminates the driver associated with handle. If possible, the driver will exit and allow MS-DOS to reclaim the memory it was using. In earlier versions of this spec, the handle argument (which must have been obtained via access_type()) was required. It is now optional, but drivers developed according to versions of this spec previous to 1.10 may require it, so implementers should take care.


## get_address()

```
get_address(handle, buf, len)
    AH      6
    BX      handle      /*  Optional  */
    ES:DI   buf
    CX      len

error return:
    carry   set
    DH      error code

possible errors:
    BAD_HANDLE          /*  older drivers only  */
    NO_SPACE

non-error return:
    carry   clear
    CX      length
```

Copies the current local net address of the interface into buf. The buffer buf is len bytes long. The actual number of bytes copied is returned in CX. If the NO_SPACE error is returned, this indicates that len was insufficient to hold the local net address. If the address has been changed by set_address(), the station address presently in effect should be returned. In earlier versions of this spec, the handle argument (which must have been obtained via access_type()) was required. It is now optional, but drivers developed according to versions of this spec previous to 1.10 may require it, so implementers should take care.


## reset_interface()

```
reset_interface(handle)
    AH      7
    BX      handle      /*  Optional  */

error return:
    carry   set
    DH      error code

possible errors:
    BAD_HANDLE          /* older drivers only */
    CANT_RESET

non-error return:
    carry   clear
```

Resets the interface associated with handle to a known state, aborting any transmits in process and reinitializing the receiver. The local net address is reset to the default (from ROM), the multicast list is cleared, and the receive mode is set to 3 (own address & broadcasts). If multiple handles are open, these actions might seriously disrupt other applications using the interface, so CANT_RESET should be returned. In earlier versions of this spec, the handle argument (which must have been obtained via access_type()) was required. It is now optional, but drivers developed according to versions of this spec previous to 1.10 may require it, so implementers should take care.


## get_parameters()      high-performance driver function

```
get_parameters()
    AH      10

error return:
    carry   set
    DH      error code

possible errors:
    BAD_COMMAND         /* No high-performance support */

non error return:
    carry   clear
    ES:DI   param
```
```c
struct param {
    unsigned char major_rev;        /* Revision of Packet Driver spec */
    unsigned char minor_rev;        /* this driver conforms to. */
    unsigned char length;           /* Length of structure in bytes */
    unsigned char addr_len;         /* Length of a MAC-layer address */
    unsigned short mtu;             /* MTU, including MAC headers */
    unsigned short multicast_aval;  /* Buffer size for multicast addr */
    unsigned short rcv_bufs;        /* (# of back-to-back MTU rcvs) - 1 */
    unsigned short xmt_bufs;        /* (# of successive xmits) - 1 */
    unsigned short int_num;         /* Interrupt # to hook for post-EOI
                                       processing, 0 == none */
};
```

The performance of an application may benefit from using get_parameters() to obtain a number of driver parameters. This function was added to v1.09 of this specification, and may not be implemented by all drivers (see driver_info()).

The major_rev and minor_rev fields are the major and minor revision numbers of the version of this specification the driver conforms to. For this document, major_rev is 1 and minor_rev is 10. The length field may be used to determine which values are valid, should a later revision of this specification add more values at the end of this structure. For this document, length is 14. The addr_len field is the length of a MAC address, in bytes. Note the param structure is assumed to be packed, such that these fields occupy four consecutive bytes of storage.

In the param structure, the mtu is the maximum MAC-level packet the driver can handle (on Ethernet this number is fixed at 1514 bytes, but it may vary on other media, e.g. 802.5 or FDDI). This value should include all MAC-layer data that the application must supply to a send_pkt() call (for class 1, source and destination addresses, and Ethertype), but not framing normally generated by the hardware (preamble, FCS, etc.). The multicast_aval field is the number of bytes required to store all the multicast addresses supported by any "perfect filter" mechanism in the hardware. Calling set_multicast_list() with its len argument equal to this value should not fail with a NO_SPACE error. A value of zero implies no multicast support.

The rcv_bufs and xmt_bufs indicate the number of back-to- back receives or transmits the card/driver combination can accomodate, minus 1. The application can use this information to adjust flow-control or transmit strategies. A single-buffered card (for example, an InterLan NI5010) would normally return 0 in both fields. A value of 0 in rcv_bufs could also be used by a driver author to indicate that the hardware has limitations which prevent it from receiving as fast as other systems can send, and to recommend that the upper-layer protocols invoke lock-step flow control to avoid packet loss.

The int_num field should be set to an interrupt that the application can hook in order to perform interrupt-time protocol processing after the EOI has been sent to the 8259 interrupt controller and the card is ready for more interrupts. Because software adapter modules may not have a hardware interrupt available, this value is specified as corresponding to the argument to an INT instruction, rather than the specific hardware interrupt line. A value of zero indicates that there is no such interrupt. Any application hooking this interrupt and finding a non-zero value in the vector must pass the interrupt down the chain and wait for its predecessors to return before performing any processing or stack switches. Any driver which implements this function via a separate INT instruction and vector, instead of just using the hardware interrupt, must prevent any saved context from being overwritten by a later interrupt. In other words, if the driver switches to its own stack, it must either support re-entrancy, or detect and prevent it.


## old_as_send_pkt()     high-performance driver function

```
old_as_send_pkt()
    AH      11

error return:
    carry   set
    DH      error code

possible errors:
    BAD_COMMAND
```

This function was added in v1.09 of this specification in order to provide an asynchronous version of send_pkt(). The specification of this function was found to be deficient, and no implementations of it were created. This function was removed in v1.10 of this specification in favor of the new improved as_send_pkt() (AH = 12).


## as_send_pkt()         high-performance driver function

```
int as_send_pkt(iocb)
    AH      12
    ES:DI   struct iocb far* iocb
```
```c
struct iocb {
    char far *buffer;           /* Pointer to xmit buffer */
    unsigned length;            /* Length of buffer */
    unsigned char flagbits;     /* Flag bits */
    unsigned char code;         /* Error code */
    void (far *transmitter)();  /* Transmitter upcall */
    char reserved[4];           /* Future gather-write */
    char private[8];            /* Driver's private data */
};
```
```
flag bits (little endian encoding):

    Name    Bit     Value   Meaning
    DONE    0       0x01    the packet driver is done with this iocb.
    UPCALL  1       0x02    the application requests an upcall when the buffer is re-usable.

error return:
    carry   set
    DH      error code

possible errors:
    CANT_SEND           /* transmit  error,  re-entered, etc. */
    BAD_COMMAND         /*  Level 0 or level 1 driver  */

non-error return:
    carry   clear

transmitter upcall:
    void (*transmitter)(iocb)
        ES:DI   struct iocb far *iocb
```

as_send_pkt() provides an asynchronous mechanism for the transmission of packets. It differs from the the synchronous send_pkt() in two major ways. First, all parameters including the normal length and buffer parameters are passed within an iocb (I/O Control Block) structure rather than being passed directly in registers. Second, as_send_pkt() never blocks awaiting buffer re- usability. For example, send_pkt() returns to the calling application only after the application's data has been copied out of the buffer and the application can safely modify or re-use the buffer. as_send_pkt() returns to the calling application immediately and signals buffer availability by setting a bit in the flagbits field of the iocb and optionally by making an application specified upcall.

as_send_pkt() takes a single argument, a far pointer to an iocb structure. The buffer and length fields of the iocb structure have exactly the same meaning as those passed to send_pkt(). The flagbits field currently has two flag bits defined: DONE and UPCALL. The DONE bit is used by the packet driver to signal re-usability of an iocb and associated buffer. It MUST be cleared to zero by the calling application before passing the iocb to the packet driver. Once the packet driver is done with an iocb -- either because the buffer has been transmitted or because an error was asynchronously detected -- it will set the DONE bit to one. The UPCALL bit is used by the calling application to request an upcall from the packet driver after the DONE bit is set to one. If the UPCALL bit is set to zero, no upcall will be made and the application must poll the DONE bit to determine iocb re- usability. If the UPCALL bit is set to one, then the packet driver will call the routine specified in the transmitter field of the iocb after setting the DONE bit to one. The application may change the state of the UPCALL bit at any time, but care should be taken to avoid introducing race conditions. Specifically, the application should be careful when examining and modifying application level queues of iocbs, if such queues exist. Depending on the strategy implemented, it may be necessary to disable interrupts while testing queue lengths and testing or modifying iocb flag bits. All other flag bits should be set to zero by the calling application.

If the driver detects an error on the initial call to as_send_pkt(), it will return the error using the standard carry flag/DH register mechanism, the iocb is left in an indeterminant state (in particular the code field and DONE bit should not be relied on), and any requested upcall will not be made. However, if as_send_pkt() detects an error later asynchronously, it will return the error in the iocb code field, set the DONE bit, and if requested, make an upcall. Successful packet transmission is indicated by a code of zero.

The structure field iocb->reserved is for the purposes of implementing a future scatter/gather capability. This field must be set to zero.

The final field in the structure, iocb->private, is for the private use of the packet driver. It will typically be used to maintain a queue of iocbs. The application must not reference or modify this field in any way. This function was added in v1.10 of this specification, and may not be implemented by all drivers (see driver_info()).


## drop_pkt()            high-performance driver function

```
int drop_pkt(iocb)
    AH      13
    ES:DI   struct iocb far* iocb

error return:
    carry   set
    DH      error code

possible errors:
    BAD_COMMAND         /*  Level 0 or level 1 driver  */

non-error return:
    carry   clear
```

Searches the as_send_pkt() asynchronous transmission queue for the specified iocb, and if found, drops it from the queue. To avoid race conditions, if the iocb is not found no error is signaled. After return from drop_pkt(), the calling application may consider the specified iocb and buffer to be safely re-usable. This function provides an application a way to implement any desired congestion control algorithms (such as random drop).


## set_rcv_mode()        extended driver function

```
set_rcv_mode(handle, mode)
    AH      20
    BX      handle
    CX      mode

error return:
    carry   set
    DH      error code

possible errors:
    BAD_HANDLE
    BAD_MODE

non-error return:
    carry   clear
```

Sets the receive mode on the interface associated with handle. The following values are accepted for mode:

| mode  | meaning                                                           |
| ----- | ----------------------------------------------------------------- |
| 1     | turn off receiver (no packets)                                    |
| 2     | receive only packets sent to this interface's station address     |
| 3     | mode 2 plus broadcast destination address                         |
| 4     | mode 3 plus multicast addresses as set via set_multicast_list()   |
| 5     | mode 3 plus all multicast packets                                 |
| 6     | all packets                                                       |
| 7     | raw mode (new in 1.10 for serial line only)                       |

Note that not all interfaces support all modes. The receive mode affects all packets received by this interface, not just packets associated with the handle argument. See the extended driver functions get_multicast_list() and set_multicast_list() for programming hardware hash filters to receive specific multicast addresses.

Most drivers do not allow set_rcv_mode() if more than one handle is open, to avoid confusing a stack with packets it doesn't expect. If a driver does not return BAD_MODE when more than one handle is open, it must maintain separate receive modes on a per-handle basis, and filter incoming traffic as appropriate to each handle's mode.

Note that mode 3 is the default, and if the set_rcv_mode() function is not implemented, mode 3 is assumed to be in force as long as any handles are open (from the first access_type() to the last release_type()).

The handle argument (which must have been obtained via access_type()) is required on this function in order to allow drivers to limit the effect of this function to the handle specified. Accordingly, protocol stacks must not depend on a change effecting all handles.


## get_rcv_mode()        extended driver function

```
get_rcv_mode(handle, mode)
    AH      21
    BX      handle

error return:
    carry   set
    DH      error code

possible errors:
    BAD_HANDLE

non-error return:
    carry   clear
    AX      mode
```

Returns the current receive mode of the interface associated with handle.


## set_multicast_list()  extended driver function

```
set_multicast_list(addrlst, len)
    AH      22
    ES:DI   char far* addrlst
    CX      len

error return:
    carry   set
    DH      error code

possible errors:
    NO_MULTICAST
    NO_SPACE
    BAD_ADDRESS

non-error return:
    carry   clear
```

The addrlst argument is assumed to point to an len-byte buffer containing a number of multicast addresses. BAD_ADDRESS is returned if len modulo the size of an address is not equal to 0, or the data is unacceptable for some reason. NO_SPACE is returned (and no addresses from the current list are set while any previously-set addresses remain in effect) if there are more addresses than the hardware or driver supports.

The recommended procedure for setting multicast addresses is to issue a get_multicast_list(), copy the information to a local buffer, add any addresses desired, and issue a set_multicast_list(). This should be reversed when the application exits. If the set_multicast_list() fails due to NO_SPACE, use set_rcv_mode() to set mode 5 instead.

Note that most hardware multicast filters use hashing techniques, and may pass addresses not specified in the addrlst. For this reason, stacks using multicast must check the destination MAC address if there is any chance they'll become confused by a packet they weren't expecting.


## get_multicast_list()  extended driver function

```
get_multicast_list()
    AH      23

error return:
    carry   set
    DH      error code

possible errors:
    NO_MULTICAST
    NO_SPACE

non-error return:
    carry   clear
    ES:DI   char far* addrlst;
    CX      len
```

On a successful return, addrlst points to len bytes of multicast addresses currently in use. The application program must not modify this information in-place. A NO_SPACE error indicates that there wasn't enough room for all active multicast addresses.


## get_statistics()       extended driver function

```
get_statistics(handle)
    AH      24
    BX      handle      /* Optional */

error return:
    carry   set
    DH      error code

possible errors:
    BAD_HANDLE          /* older drivers only */

non-error return:
    carry   clear
    DS:SI   char far* stats
```
```c
struct statistics {
    unsigned long packets_in;   /* Totals across all handles */
    unsigned long packets_out;
    unsigned long bytes_in;     /* Including MAC headers */
    unsigned long bytes_out;
    unsigned long errors_in;    /* Totals across all error types */
    unsigned long errors_out;
    unsigned long packets_lost; /* No buffer from receiver(), card */
                                /* out of resources, etc. */
};
```

Returns a pointer to a statistics structure for the interface. The values are stored so as to be normal 80x86 32-bit integers. Stacks should assume that this structure is dynamic, and copy it to local storage with interrupts disabled. In earlier versions of this spec, the handle argument (which must have been obtained via access_type()) was required. It is now optional, but drivers developed according to versions of this spec previous to 1.10 may require it, so implementers should take care.


## set_address()          extended driver function

```
set_address(addr, len)
    AH      25
    ES:DI   char far* addr
    CX      len

error return:
    carry   set
    DH      error code

possible errors:
    CANT_SET
    BAD_ADDRESS

non-error return:
    carry   clear
    CX      length
```

This call is used when the application or protocol stack needs to use a specific LAN address. For instance, DECnet protocols on Ethernet encode the protocol address in the Ethernet address, requiring that it be set when the protocol stack is loaded. A BAD_ADDRESS error indicates that the Packet Driver doesn't like the len (too short or too long), or the data itself (a driver may refuse a station address with the 'group' bit set, for instance). Note that packet drivers should refuse to change the address (with a CANT_SET error) if more than one handle is open (lest it be changed out from under another protocol stack).


## send_raw_bytes()       extended driver function

```
send_raw_bytes(buffer, length)
    AH      26
    DS:SI   char far* buffer
    CX      count

error return:
    carry   set
    DH      error code

possible errors:
    BAD_MODE        /* Not in raw mode */
    BAD_COMMAND     /* Not a serial line driver */

non-error return:
    carry flag clear
```

Transmits the string of bytes in buffer for the given count out the serial port. This routine does not return until the last transmit interrupt for the last byte of the string has been successfully processed. It is assumed that the packet driver has been placed in raw mode before the call is made. If not, a BAD_MODE error is returned.


## flush_raw_bytes()      extended driver function

```
flush_raw_bytes()
    AH      27

error return:
    carry   set
    DH      error code

possible errors:
    BAD_MODE        /* Not in raw mode */
    BAD_COMMAND     /* Not a serial line driver */

non-error return:
    carry   clear
```

Flushes the receive ring buffer of any characters it may have received. It is assumed that the packet driver has been placed in raw mode before the call is made. If not, a BAD_MODE error is returned.


## fetch_raw_bytes()          extended driver function

```
fetch_raw_bytes(buffer, buf_len, timeout)
    AH      28
    DS:SI   char far* buffer
    CX      buf_len
    DX      timeout

error return:
    carry   set
    DH      error code

possible errors:
    BAD_MODE
    BAD_COMMAND

non-error return:
    carry   clear
    CX      bytes copied
```

Fetch bytes from the serial port until either the timeout expires, or the specifed count of bytes has been received. The timeout is measured in clock ticks (~18/sec). It is assumed that the packet driver has been placed in raw mode before the call is made. If not, a BAD_MODE error is returned.


## signal

```
int signal (signal_number, data)
    AH      29
    AL      signal_number
            = 1  REGISTER_UPCALL
            = 2  UNREGISTER_UPCALL
            = 3  THIS_LAYER_START
            = 4  THIS_LAYER_FINISH
            = 5  PHYSICAL_UP_EVENT
            = 6  PHYSICAL_DOWN_EVENT

    if AL = 1 or AL = 2, data is:
        ES:DI   int (far *signal_handler)();

error return:
    carry   set
    DH      error code

possible errors:
    BAD_COMMAND         /* The command is not implemented */
    BAD_SIGNAL          /* The signal is not implemented */
    if AL = 2
        BAD_ARGUMENT   /* The upcall was not registered */

non-error return:
    carry   clear

signal handler upcall:
    (*signal_handler)(event)
        AX  event
            = 1  THIS_LAYER_DOWN
            = 2  THIS_LAYER_UP
    No return value
```

The signal() call allows the client application to register an upcall which can be used for signalling events. This mechanism was designed to accomodate the requirements of a PPP packet driver, however the mechanism was also designed to be as generic as possible so that it hopefully may be used for other as yet unknown purposes in the future.

Before a client application can expect event signals from the packet driver, the client application must register its signal_handler() upcall with the packet driver. This is accomplished by calling signal() with a signal_number of REGISTER_UPCALL and also supplying the function pointer of the signal_handler() upcall as the data. The packet driver then records this upcall pointer in a table and uses it to signal an event to the client application.

Before the client application exits, the client application must unregister its signal_handler() upcall. This is accomplished by calling signal() with a signal_number of UNREGISTER_UPCALL and also supplying the function pointer of the upcall as the data. If the upcall is not unregistered, the packet driver may call the unloaded routine with unpredictable results, (ie. the machine will likely hang).

For the purposes of a PPP packet driver, this call allows the NCP layers and the LCP layer to send signals to one another. An NCP layer sends signals to the LCP layer by calling signal(). The LCP layer sends signals to the NCP layers by calling the signal_handler() upcall.

For the purposes of PPP, the NCP layers use the signal() call to send the THIS_LAYER_START and THIS_LAYER_FINISH signals to the LCP layer. The LCPlayer uses the registered signal_handler upcall to send the THIS_LAYER_UP and THIS_LAYER_DOWN signals to the NCP layers.

In the case where the LCP layer is a standalone packet driver seperate from the physical layer packet driver, the LCP layer uses the signal() call to send the THIS_LAYER_START and THIS_LAYER_FINISH signals to the physical layer. The physical layer uses the registered signal_handler upcall to send the THIS_LAYER_UP and THIS_LAYER_DOWN signals to the LCP layer.

See the description of PPP (Class 16/18) under "Specific Characteristics of Classes" for more detail.


## get_structure

```
get_structure(structure_type)
    AH      30
    BX      structure_type

error return:
    carry   set
    DH      error code

possible errors:
    BAD_ARGUMENT        /* structure_type value unknown */

non-error return:
    carry   clear
    DS:SI   char far* structure_ptr
```

Returns a pointer to the structure indicated in the structure_type argument. The format of the structure varies with the value of structure_type. Packet drivers should implement the structures which are most relevant, and must return BAD_ARGUMENT for any structure not implemented.

The client application should assume that the structures are dynamic, and copy them to local storage with interrupts disabled.


# Specific Characteristics of Classes

Packet Drivers have been developed for the following media classes, and interpretations of demultiplexing criteria, MTU etc. have stabilized. If you wish to implement a Packet Driver for a class which isn't listed below, you should contact FTP Software to make sure that your interpretation agrees with any other work we may be aware of.


## DIX Ethernet (Class 1)

Class 1 is demultiplexed by the 'ethertype', a 16-bit value (MSB first) transmitted as the thirteenth and fourteenth bytes of a frame. typelen is normally 2. The maximum length allowed by send_pkt() is 1514, 1500 bytes of data preceded by the 14-byte MAC header (source address, destination address, ethertype). The minimum "look-ahead buffer" length allowed is 60 bytes. If the typelen is 0, requesting promiscuous mode, the driver should pass both Class 1 and Class 11 packets to the protocol stack.


## 802.5 Token Ring (Class 3)

Class 3 is demultiplexed according to the contents of the IEEE 802.2 header which follows the 802.5 header in the packet. typelen values may vary between 1 (for demultiplexing on the Destination SAP only) and 8 (for demultiplexing on a complete RFC 1042 header: SNAP, OUI and 'ethertype'). In any comparison which, like RFC 1042, includes the 802.2 'control' field, the driver must mask the 'poll/final' bit in the 'control' byte prior to the comparison.

On the wire, 802.5 headers are variable-length, consisting of the 14-byte 802.5 MAC header, an optional Routing Information Field (used by source-routing bridges), and finally the variable-length 802.2 header. Across the Packet Driver interface (in receiver() upcalls and in calls to send_pkt()), the RIF remains unmodified (unlike Class 17). The 802.2 header remains variable- length.

If an application needs to determine if a RIF is actually present in a received packet, it must examine the RIF_present bit in the source MAC address (this occupies the same location as the global bit in the destination MAC address). If an application needs to send a RIF, the 'RIF length' and other components of the RIF must be inserted between the MAC addresses and the 802.2 header. The user is responsible for setting the RIF_present bit in the source MAC address if necessary.

The MTU on 802.5 is variable, negotiated during the initial exchange of RIFs. Several existing implementations of IP over 802.5 require that the value encoded in the RIF be 2052 bytes. The minimum "look-ahead buffer" length allowed is 100 bytes.


## Appletalk (Class 5)

Class 5 provides access to an ATALK.SYS driver, for forwarding IP packets to and from an Internet Router on an Appletalk net. As such, there is no demultiplexing taking place within the Packet Driver, and typelen is 0. No header is prefixed to the IP datagram on transmission. The MTU is 568 bytes, derived from the underlying Appletalk DDP service.


## SLIP (Class 6)

SLIP as defined in RFC 1055 does not include any packet demultiplexing information. Thus, type and typelen are irrelevant, and all open handles receive the same packets. Many implementations assume an MTU (not including any byte-stuffing required to escape in-band control characters found in the data stream) of 1006 bytes, but there are exceptions.

Note that the get_address() function is not applicable to Class 6.


## AX.25 Amateur Radio (Class 9)

The AX.25 frame structure looks like this (taken from version 2.0 of the spec):

```
U and S frame:
    First Bit Sent
    Flag        Address         Control     FCS         Flag
    01111110    112/560 Bits    8 Bits      16 Bits     01111110

Information frame:
    Flag        Address         Control     PID     Info.       FCS         Flag
    01111110    112/560 Bits    8 Bits      8 Bits  N*8 Bits    16 Bits     01111110
```

The Address field has a variable length because it is used to hold the source and destination callsigns, and 0 to seven callsigns of radio stations which are used to repeat the packet. The control field contains N(r) and N(s), a poll/final bit, and some other stuff not relevant to this discussion. The information field is an integral number of octets long. AX.25 doesn't specify the MTU; my driver will handle 2k. The FCS is CRC-16. The PID field specifies the layer 3 protocol in use, such as AX.25 layer 3, IP, ARP, NET/ROM, etc.

Phil Karn's KA9Q implementation calls the packet driver with typelen set to zero. It is proposed that the driver should accept a pointer to a PID field with typelen set to 1 byte, in the future.

Note that the get_address() function is not applicable to Class 9.


## 802.3 with 802.2 headers (Class 11)

Class 11 is demultiplexed according to the contents of the IEEE 802.2 header which follows the 14-byte 802.3 header in the packet. typelen values may vary between 1 (for demultiplexing on the Destination SAP only) and 8 (for demultiplexing on a complete RFC 1042 header: SNAP, OUI and 'ethertype'). The MTU is fixed at 1514 bytes, the same as DIX ethernet, but because of the presence of the 802.2 header, the usable data length is at least 3 bytes shorter. The minimum "look-ahead buffer" length allowed is 60 bytes.


## FDDI with 802.2 headers (Class 12)

Class 12 is demultiplexed according to the contents of the IEEE 802.2 header which follows the FDDI header in the packet. typelen values may vary between 1 (for demultiplexing on the Destination SAP only) and 8 (for demultiplexing on a complete RFC 1042 header: SNAP, OUI and 'ethertype'). In any comparison which, like RFC 1042, includes the 802.2 'control' field, the driver must mask the 'poll/final' bit in the 'control' byte prior to the comparison. The MTU is variable, The Packet Driver must perform the bit-swap on all MAC addresses, because while the bit order on the physical FDDI media matches 802.5, the bit order handled by the protocol stacks is specified as matching 802.3.


## Internet X.25 (Class 13)

Class 13 is a special case: It is designed to allow X.25 to be used as a transport for IP and other datagrams. Each packet of data sent by the protocol stack is preceded by a protocol specific header; the first byte is protocol type and the meaning of the rest of the header varies with protocol type. IP's protocol type is 1, and its header is defined as follows:

```c
struct x25_pd_hdr {
    unsigned char type;     /* Type: 1 indicates IP */
    unsigned char tos;      /* IP Type Of Service value to use */
    unsigned long dest;     /* IP next hop address */
    unsigned long src;      /* IP Source address */
};
```

Header layouts specific to other protocol families will be defined as the need arises. The typelen value passed to access_type() is always 1, and the type pointer points at a byte defining the protocol to be received; 1 for IP as above. On receiver() upcalls, the header specified for transmission is present, but only the type value is set. The MTU is variable, depending on the configuration of the attached network. The minimum "look-ahead buffer" length is 40 bytes.


## Northern Telecom LANSTAR encapsulating DIX (Class 14)

Class 14 is a "simulated ethernet" class, and the MTU, demultiplexing and other characteristics are similar to Class 1.


## Point to Point Protocol MAC layer driver (Class 16)

Note: Class 16 drivers are obselete and should not be used, with the exception for very special cases (such as LANWatch). PC/TCP uses a class 18 driver to do PPP.

The MTU of a PPP link is determined at link startup through the negotiation of the Maximum-Receive-Unit (MRU). The MRU of the link is the size of the info field of a PPP frame, and does not include the two byte protocol field in the header. Thus, just as the MTU of an ethernet frame is 1500 for the data area, plus 14 bytes for the header, the MTU of a PPP link is equal to the negotiated MRU plus 2 for its header.

Class 16 is demultiplexed based on the two byte protocol field in the PPP frame header. The packet driver passes to the protocol stack the PPP protocol field, followed by the info field. The stack can examine this field to determine the protocol type of the incoming packet.

In packets passed to send_packet, the protocol stack must set an appropriate value in the PPP protocol field ahead of the PPP info field. The packet driver will add the flag, address, control, and FCS bytes as it sends the packet.

During PPP frame reception, the flag, address, control, and FCS bytes of a PPP frame are all consumed by the packet driver before the packet is passed up to the protocol stack.

During PPP frame reception, the total incoming length cannot be determined at the time that only the PPP header has been successfully parsed. The entire frame must be received before the actual length can be determined. Since for performance reasons it is preferable to receive the incoming frame into a protocol stack buffer rather than into a local buffer, the only recourse is to request a full size buffer from the protocol stack just after the frame header has been fully received, but before the info field has been received.

The buffer size requested from the protocol stack would be at least MRU+2 where the two extra bytes hold the two byte protocol field. However, in order to simplify the implementation of an async PPP packet driver, the size requested must be MRU+4, which allows not only two bytes for the protocol field, but also two more bytes to hold the FCS field which occurs at the end of the frame. Once the closing flag has been received, the length must be decreased by 2 to adjust for the FCS field.

Thus a class 16 protocol stack must be capable of supplying buffers which total MRU+4 bytes, so as to accomodate the needs of an asynchronous PPP packet driver.


## 802.5 Token Ring w/expanded RIFs (Class 17)

Class 17 is demultiplexed in the same way as Class 3, and has the same MTU and minimum "look-ahead buffer" length. The significant difference is that across the Packet Driver interface (in receiver() upcalls and in calls to send_pkt()), the RIF is padded to its maximum size, 18 bytes, while the 802.2 header remains variable-length. This was defined this way to streamline RIF handling in protocol implementations where fixed buffer offsets are used to increase performance.

If an application needs to determine if a RIF is actually present in a received packet, it must examine the RIF_present bit in the source MAC address (this occupies the same location as the global bit in the destination MAC address). If an application needs to send a RIF, the 'RIF length' and other components of the RIF must be left-justified in the RIF area. Because it must perform RIF expansion, send_pkt() is responsible for setting the RIF_present bit in the source MAC address if necessary.

Note that the RIF expansion on received packets is specified to take place during the data copy, so the "look-ahead buffer" will contain a compressed RIF on the AX == 0 receiver() upcall.


## Point to Point Protocol with LLC layer (Class 18)

WARNING: The information presented on the class 18 packet driver is a documentation of how FTP Software wrote it's PPP16550 packet driver that shipped with PC/TCP version 3.0 and OnNet version 1.1. It is presented purely for historical and/or compatibility reasons, and should be implemented only when compatibility is desired with these versions of software. Future drivers from FTP Software will likely not utilize the class 18 packet driver for PPP communications.

Note: Much of the text below assumes familiarity with the PPP RFC's, notably RFCs 1548 and 1549.

### Components

All class 18 packet drivers contain an LCP option negotiation layer. This layer allows them to perform common link level option negotiation on behalf of the client applications. All NCP option negotiation layers are provided by the client applications.

All class 18 packet drivers except type 1 contain a physical link driver in addition to the LCP layer. The class 18 type 1 packet driver contains an LCP layer but no physical link level driver. The class 18 type 1 packet driver instead relies on a class 16 packet driver to supply the physical link driver functionality. The purpose behind this arrangement is to avoid the reimplementation of the LCP layer in each PPP packet driver written.

Because there must be only one LCP entity but perhaps multiple NCP entities, client applications which provide network layer service such as IP/IPCP or IPX/IPXCP should avoid including an LCP layer. Client applications should rely on and share the services of this LCP equipped class 18 packet driver.

### Packet Processing

PPP frames received are demultiplexed based on the two byte protocol field found in the PPP header. Valid protocol field values are defined in RFC1548. If the sender used protocol field compression to reduce the protocol field ot one byte, the field must be expanded again to two bytes before demultiplexing.

The info field of the PPP frame contains the higher layer protocol data. The length of the info field varies from frame to frame, and is not indicated in the PPP frame header. However the maximum length (called the MRU) is negotiated at link startup using LCP packets and is guaranteed not to change for the duration of the link, or until LCP negotiation reoccurs. Thus, the MTU of the link is determined by the LCP layer during option negotiation.

In packets passed to send_pkt(), the client application must set an appropriate value in the PPP protocol field ahead of the PPP info field. The packet driver will add the flag, address, control and FCS bytes as it sends the packet.

The packet driver passes to the client application the PPP protocol field followed by the info field. The flag, address, control and FCS bytes of a PPP frame are all consumed by the packet driver before the packet is passed up to the client application.

The FCS is verified by the physical link driver in the packet driver. If the FCS is valid, the packet driver passes the frame to the client application for processing. The client application can examine the protocol field to determine the protocol type of the packet. If the FCS is invalid, the frame is discarded and the packet driver returns the buffer to the client application with a zero length.

During receiver() upcall processing, a client application should be prepared to honor a request from the packet driver for a packet size of up to MRU+4. The extra 4 bytes include 2 for the protocol field, and 2 for the FCS field. Although the contents of the 2 byte FCS field will not be interesting to the client application, the extra 2 bytes for the FCS are useful in implementing async byte- oriented packet drivers. Because there is no length field, a byte-oriented frame parser implemented in software can only determine the position of the FCS field once it has received the end-of-frame flag. Using the default MRU size of 1500 bytes, the application should allocate 1504-byte buffers. If the MTU is negotiated to some size smaller or larger than 1500 bytes, then the client application should take this alternate MTU value, add 4 and use this number as the buffer size to offer the packet driver.

### Layer Signalling and Packet Type Registration

The RFC1548 PPP option negotiation automaton specifies the following events for use in driving the negotiation process: START, FINISH, UP, DOWN. Notification of these events must be passed between the NCP and LCP layers (and between the LCP and physical link driver layers, if they are different drivers). The signal() call defined in this spec is used to pass these indications between the TSR's which implement the NCP, LCP, and physical link driver layers.

At startup, the NCP layer and the network protocol layer should not call access_type() in the packet driver to hook their protocol types, but should make instead a call to the LCP layer using signal() with a REGISTER_UPCALL signal in order to register a signal_handler() upcall with the LCP layer. It should then call signal() again with a THIS_LAYER_START signal. Then it must wait for the LCP layer to signal a THIS_LAYER_UP event via the signal_handler() upcall. Once the THIS_LAYER_UP signal is received, the NCP layer can hook its protocol type value (ie. 0x8021 for IPCP) and begin negotiation with the peer. Once the NCP has reached an OPENED state (see RFC1331) the network layer can hook its protocol type (ie. 0x0021 for IP) and begin passing packets.

If at any point the NCP layer receives a THIS_LAYER_DOWN event from the LCP layer, the NCP layer must notify the network layer, and both the NCP layer and the network layer must unhook their protocol types using release_type() until the LCP layer again signals a THIS_LAYER_UP event.

When the NCP/network layer exits, it must unhook its protocol types, signal a THIS_LAYER_FINISH event to the LCP layer using the signal() call, and unregister the signal_handler() using the signal() call with an UNREGISTER_HANDLER signal.

When a class 18 packet driver starts up, it must be prepared to begin negotiating LCP options regardless of whether any client applications have or have not registered a signal_handler(). As client applications register signal_handler() upcalls and send THIS_LAYER_START signals, the packet driver must save the upcall routine pointers in a table. After registering a signal upcall for a new client application in the table, the packet driver should check the LCP layer current state. If the LCP layer is currently in an OPENED state, the packet driver should send a THIS_LAYER_UP signal to the client application immediately before returning.

Each time the LCP layer transitions into an OPENED state, it must traverse the table of registered signal_handler() upcalls and send a THIS_LAYER_UP signal to each in succession. Each time the LCP layer transitions out of an OPENED state, it must traverse the table of registered signal_handler() upcalls and send a THIS_LAYER_DOWN signal to each in succession. This corresponds with the PPP automaton table's tlu and tld events.

In order to simplify the implementation of the protocol- reject mechanism defined in RFC1548, the LCP layer of the class 18 to class 16 converter will call access_type() with typelen of zero to receive packets with unregistered protocol types. When the class 16 packet driver receives a access_type() call with a typelen argument value of zero, the receiver() upcall function must be passed every subsequently received packet for which there is not a registered protocol type receiver. The LCP layer in the converter can then send protocol-reject frames in response to packets with unrecognized protocols.

### LCP Option Configuration

Before LCP layer option negotiation begins, the LCP layer must be configured with a set of initial option parameters. There may be a requirement that the values of these parameters vary from one connection to the next. The get_structure() call is used to retrieve a pointer to a structure within the LCP layer in the class 18 packet driver. Using this pointer, the client application can copy the desired values into the LCP layer data space. See Appendix E for structure layout.

After the LCP layer option negotiation completes, the LCP layer must reconfigure the physical link driver with the final negotiated values. The get_structure() call is used to retrieve a pointer to the structure within the physical link driver in the class 16 (or 18) packet driver which holds the current operating values. The LCP layer uses this pointer to copy in the new operating values. See Appendix E for structure layout.

When the LCP layer transitions out of the OPENED state, the LCP layer must reset the current operating values in the physical link driver to the default values as specified in RFC1548. The get_structure() call is used to retrieve a pointer to the structure within the physical link driver in the class 16 (or 18) packet driver which holds the current operating values. The LCP layer uses this pointer to copy in the default operating values. See Appendix E for structure layout.


# Appendix A. Interface classes and types

The following are defined as network interface classes, with their individual types listed immediately following the class.


## Class 1     DEC/Intel/Xerox "Bluebook" Ethernet

```
3COM 3C500/3C501 ........................................1
3COM 3C505 ..............................................2
InterLan NI50103 ........................................3
BICC Data Networks 4110 .................................4
BICC Data Networks 4117 .................................5
InterLan NP600 ..........................................6
Ungermann-Bass PC-NIC ...................................8
Univation NC-516 ........................................9
TRW PC-2000 ............................................10
InterLan NI5210 ........................................11
3COM 3C503 .............................................12
3COM 3C523 .............................................13
Western Digital WD8003 .................................14
Spider Systems S4 ......................................15
Torus Frame Level ......................................16
10NET Communications ...................................17
Gateway PC-bus .........................................18
Gateway AT-bus .........................................19
Gateway MCA-bus ........................................20
IMC Pcnic ..............................................21
IMC PCnic II ...........................................22
IMC PCnic 8bit .........................................23
Tigan Communications ...................................24
Micromatic Research ....................................25
Clarkson "Multiplexor" .................................26
D-Link 8-bit ...........................................27
D-Link 16-bit ..........................................28
D-Link PS/2 ............................................29
Research Machines parallel .............................30
Research Machines 16 ...................................31
Research Machines MCA ..................................32
Radix Microsys. EXM1 16-bit ............................33
InterLan NI9210 ........................................34
InterLan NI6510 ........................................35
Vestra LANMASTER 16-bit ................................36
Vestra LANMASTER 8-bit .................................37
Allied Telesis PC/XT/AT ................................38
Allied Telesis NEC PC-98 ...............................39
Allied Telesis Fujitsu FMR .............................40
Ungermann-Bass NIC/PS2 .................................41
Tiara LANCard/E AT .....................................42
Tiara LANCard/E MC .....................................43
Tiara LANCard/E TP .....................................44
Spider Comm. SpiderComm 8 ..............................45
Spider Comm. SpiderComm 16 .............................46
AT&T Starlan NAU .......................................47
AT&T Starlan-10 NAU ....................................48
AT&T Ethernet NAU ......................................49
Intel smart card .......................................50
Xircom Pocket Adapter / Credit Card Adapter ............51
Aquila Ethernet ........................................52
Novell NE-1000 .........................................53
Novell NE-2000 .........................................54
SMC PC-510 .............................................55
AT&T Fiber NAU .........................................56
NDIS to Packet Driver adapter ..........................57
InterLan ES3210 ........................................58
General Systems ISDN simulated Ether ...................59
Hewlett-Packard ........................................60
IMC EtherNic-8 .........................................61
IMC EtherNic-16 ........................................62
IMC EtherNic-MCA .......................................63
NetWorth EtherNext .....................................64
Dataco Scanet ..........................................65
DEC DEPCA ..............................................66
C-net ..................................................67
Gandalf LANLine ........................................68
Apricot built-in .......................................69
David Systems Ether-T ..................................70
ODI to Packet Driver adapter ...........................71
AMD Am2110 16-bit ......................................72
Intel ICD Network controller family ....................73
Intel ICD PCL2 .........................................74
Intel ICD PCL2A ........................................75
AT&T LANPacer ..........................................76
AT&T LANPacer+ .........................................77
AT&T EVB ...............................................78
AT&T StarStation .......................................79
SLIP simulated ether (skl) .............................80
InterLan NIA310 ........................................81
InterLan NISE ..........................................82
InterLan NISE30 ........................................83
InterLan NI6610 ........................................84
Ether over IP/UDP ......................................85
ICL EtherTeam 16 .......................................86
David Systems Ether-T AT Plus ..........................87
NCR WaveLAN ............................................88
Thomas Conrad TC5045 ...................................89
Parallel Port driver ...................................90
Intel EtherExpress 16 ..................................91
IBMTOKEN simulated Ether on 802.5 ......................92
Zenith Data Systems Z-Note .............................93
3Com 3C509 .............................................94
Mylex LNE390 ...........................................95
Madge Smart Ringnode ...................................96
Novell NE2100 ..........................................97
Allied Telesis 1500 ....................................98
Allied Telesis 1700 ....................................99
Fujitsu EtherCoupler ..................................100
```


## Class 2     ProNET-10

```
Proteon p1300 ...........................................1
Proteon p1800 ...........................................2
```


## Class 3     IEEE 802.5 without expanded RIFs

```
IBM Token ring adapter ..................................1
Proteon p1340 ...........................................2
Proteon p1344 ...........................................3
Gateway PC-bus ..........................................4
Gateway AT-bus ..........................................5
Gateway MCA-bus .........................................6
Madge ??? ...............................................7
NDIS to Packet Driver adapter ..........................57
ODI to Packet Driver adapter ...........................71
```


## Class 4     Omninet


## Class 5     Appletalk

```
ATALK.SYS adapter .......................................1
```


## Class 6     Serial line (SLIP)

```
Clarkson 8250-SLIP / PC/TCP's SLP16550.COM ..............1
Clarkson "Multiplexor" ..................................2
Eicon Technologies ......................................3
```


## Class 7     Starlan

Note: This class was consumed by the Ethernet class (class 1).


## Class 8     ArcNet

```
Datapoint RIM ...........................................1
```


## Class 9     AX.25

```
Ottawa PI card ..........................................1
Eicon Technologies ......................................2
```


## Class 10    KISS


## Class 11    IEEE 802.3 w/802.2 hdrs

Types as given under Class 1 (DIX Ethernet)


## Class 12    FDDI w/802.2 hdrs


## Class 13    Internet X.25

```
Western Digital .........................................1
Frontier Technology .....................................2
Emerging Technologies ...................................3
The Software Forge ......................................4
Link Data Intelligent X.25 ..............................5
Eicon Technologies ......................................6
```


## Class 14    N.T. LANSTAR (encapsulating DIX)

```
NT LANSTAR/8 ............................................1
NT LANSTAR/MC ...........................................2
```


## Class 15    SLFP (MIT serial spec)

```
MERIT ...................................................1
```


## Class 16    Point to Point Protocol (no LCP)

```
8250/16550 UART .........................................1
Niwot Networks synch ....................................2
Eicon Technologies ......................................3
```


## Class 17    IEEE 802.5 with expanded RIFs

Types as given under Class 3 (802.5 with verbatim RIFs)


## Class 18    Point to Point Protocol (with LCP)

```
Class 16 to class 18 converter ..........................1
PC/TCP's PPP16550.COM ...................................2
```


# Appendix B. Function call numbers

The following decimal numbers are used to specify which operation the packet driver should perform. The number is stored in register AH on call to the packet driver.

```
driver_info .............................................1
access_type .............................................2
release_type ............................................3
send_pkt ................................................4
terminate ...............................................5
get_address .............................................6
reset_interface .........................................7
+get_parameters ........................................10
+old_as_send_pkt .......................................11
+as_send_pkt ...........................................12
+drop_pkt ..............................................13
*set_rcv_mode ..........................................20
*get_rcv_mode ..........................................21
*set_multicast_list ....................................22
*get_multicast_list ....................................23
*get_statistics ........................................24
*set_address ...........................................25
*send_raw_bytes ........................................26
*flush_raw_bytes .......................................27
*fetch_raw_bytes .......................................28
*signal ................................................29
*get_structure .........................................30
```

+ indicates a high-performance packet driver function
* indicates an extended packet driver function

AH values from 128 through 255 (0x80 through 0xFF) are reserved for user-developed extensions to this specification. While FTP Software cannot support user extensions, we are willing to act as a clearing house for information about them. For more information, contact us.

The following values have been allocated to the listed organizations:
   233  Crynwr Software          autoselect transceiver


# Appendix C. Error codes

Packet driver calls indicate error by setting the carry flag on return. The error code is returned in register DH (a register not used to pass values to functions must be used to return the error code). The following error codes are defined:

| Code  | Error             | Description                                                   |
| ----- | ----------------- | ------------------------------------------------------------- |
| 1     | BAD_HANDLE        | Invalid handle number                                         |
| 2     | NO_CLASS          | No interfaces of specified class found                        |
| 3     | NO_TYPE           | No interfaces of specified type found                         |
| 4     | NO_NUMBER         | No interfaces of specified number found                       |
| 5     | BAD_TYPE          | Bad packet type specified                                     |
| 6     | NO_MULTICAST      | This interface does not support multicast                     |
| 7     | CANT_TERMINATE    | This packet driver cannot terminate                           |
| 8     | BAD_MODE          | An invalid receiver mode was specified                        |
| 9     | NO_SPACE          | Operation failed because of insufficient space                |
| 10    | TYPE_INUSE        | The type had previously been accessed, and not released       |
| 11    | BAD_COMMAND       | The command was out of range, or not implemented              |
| 12    | CANT_SEND         | The packet couldn't be sent (usually hardware error)          |
| 13    | CANT_SET          | Hardware address couldn't be changed (more than 1handle open) |
| 14    | BAD_ADDRESS       | Hardware address has bad length or format                     |
| 15    | CANT_RESET        | Couldn't reset interface (more than 1 handle open)            |


# Appendix D. 802.3 vs. Blue Book Ethernet

An issue concerning the present specification is that there is no provision for simultaneous support of 802.3 and Blue Book (the old DEC-Intel-Xerox standard) Ethernet headers via a single Packet Driver (as defined by its interrupt). The problem is that the Ethertype of Blue Book packets is in bytes 12 and 13 of the header, and in 802.3 the corresponding bytes are interpreted as a length. In 802.3, the field which would appear to be most useful to begin the type check in is the 802.2 header, starting at byte 14. This is only a problem on Ethernet and variants (e.g. Starlan), where 802.3 headers and Blue Book headers are likely to need co-exist for many years to come.

One solution is to redefine class 1 as Blue Book Ethernet, and define a parallel class for 802.3 with 802.2 packet headers. This requires that a 2nd Packet Driver (as defined by its interrupt) be implemented where it is necessary to handle both kinds of packets, although they could both be part of the same TSR module.

As of v1.07 of this specification, class 11 was assigned to 802.3 using 802.2 headers, to implement the above.

Note: According to this scheme, an application wishing to receive IP encapsulated with an 802.2 SNAP header and Ethertype of 0x800, per RFC 1042, would specify an typelen argument of 8, and type would point to:

```c
char ieee_ip[] = {0xAA, 0xAA, 3, 0, 0, 0, 0x08, 0x00};
```


# Appendix E. Structure layout for get_structure call

The packet driver structures detailed below have been created to allow a client application to access data held within the packet driver in a generalized manner. Each structure has a structure_type which serves as an identifying tag. This tag is used in the get_structure() call to specify the structure requested.

The following structure_types have been defined. The corresponding structure layouts are also detailed.

```c
STRUCT_IO_STATS  1
    struct io_statistics {
        unsigned long packets_in;
        unsigned long packets_out;
        unsigned long bytes_in;
        unsigned long bytes_out;
        unsigned long errors_in;
        unsigned long errors_out;
        unsigned long packets_lost;
    };

STRUCT_LCP_INITIAL_CONFIG  2
    struct lcp_options {
        unsigned mru;
        unsigned long int rcvaccm;
        unsigned long int sndaccm;
        unsigned long int magic;
        unsigned char acfc;
        unsigned char pfc;
        unsigned authprot;
    };

STRUCT_LCP_CURRENT_CONFIG  3
    struct lcp_options {
        unsigned mru;
        unsigned mtu;
        unsigned long int rcvaccm;
        unsigned long int sndaccm;
        unsigned long int magic;
        unsigned char acfc;
        unsigned char pfc;
        unsigned authprot;
    };

STRUCT_LCP_PARAMS  4
    struct lcp_params {
        unsigned restart_ctr;
        unsigned restart_timer;
        unsigned timeout_period;
        unsigned max_configure;
        unsigned max_terminate;
    };

STRUCT_LCP_PKT_STATS  5
    struct lcp_pkt_stats {
        unsigned pkts_rcvd;
        unsigned pkts_sent;
        unsigned pkts_discarded;
        unsigned code_unknown;
        unsigned cnfg_req_rcvd;
        unsigned cnfg_ack_rcvd;
        unsigned cnfg_nak_rcvd;
        unsigned cnfg_rej_rcvd;
        unsigned term_req_rcvd;
        unsigned term_ack_rcvd;
        unsigned code_rej_rcvd;
        unsigned prot_rej_rcvd;
        unsigned echo_req_rcvd;
        unsigned echo_rep_rcvd;
        unsigned disc_req_rcvd;
        unsigned prot_rej_sent;
        unsigned rcvq_oflow;
        unsigned sendq_oflow;
    };

STRUCT_LCP_EVENT_STATS  6
    struct lcp_event_stats {
        unsigned event_up;
        unsigned event_down;
        unsigned event_open;
        unsigned event_close;
        unsigned event_timeout_plus;
        unsigned event_timeout_minus;
        unsigned event_rcr_plus;
        unsigned event_rcr_minus;
        unsigned event_ruc;
        unsigned event_rxj_plus;
        unsigned event_rxj_minus;
        unsigned event_rxr;
        unsigned event_illegal;
        unsigned event_ignored;
    };

STRUCT_LCP_STATE  7
    struct lcp_state {
        unsigned char status;
        unsigned char lcp_state;
        unsigned char state_history[10];
        unsigned char state_history_num[11];
        unsigned char padding;
    };

STRUCT_PAP_CONFIG  8
    struct pap_config {
        char *identity;
        char *password;
        char *message;
    };

STRUCT_PAP_STATS  9
    struct pap_stats {
        unsigned pkts_sent;
        unsigned pkts_rcvd;
        unsigned auth_req_sent;
        unsigned auth_req_rcvd;
        unsigned auth_ack_sent;
        unsigned auth_ack_rcvd;
        unsigned auth_nak_sent;
        unsigned auth_nak_rcvd;
        unsigned status;
    };

STRUCT_UART_CONFIG  10
    struct uart_conf {
        unsigned uart;
        unsigned port;
        unsigned io8250;
        unsigned char irq;
        unsigned char pattr;
        unsigned char clrbda;
        unsigned char rtscts;
        unsigned long baud;
        unsigned baudiv;
    };

STRUCT_UART_STATS  11
    struct uart_stats {
        unsigned long int rcv_ints;
        unsigned long int xmt_ints;
        unsigned long int modem_ints;
        unsigned long int line_ints;
        unsigned long int fifo_ints;
        unsigned xmt_tmo;
        unsigned breaks;
        unsigned framing;
        unsigned parity;
        unsigned overruns;
        unsigned rb_overflow;
        unsigned char modem_status;
    };

STRUCT_HDLC_STATS  12
    struct hdlc_stats {
        unsigned long int mode1_bytes_rcvd;
        unsigned long int mode6_bytes_rcvd;
        unsigned long int mode7_bytes_rcvd;
        unsigned over_mtu;
        unsigned pkts_disc_no_bufs;
        unsigned over_buffer;
        unsigned max_len_pkt;
        unsigned pkts_disc_no_lcp_bufs;
        unsigned fcs_failures;
        unsigned runt;
        unsigned protocol_unknown;
        unsigned char state;
        unsigned char padding[3];
    };

STRUCT_TYPE_TABLE  13
    struct type_table {
        short maxentries;
        short currentmode;
        struct {
            unsigned short handle;
            unsigned short type;
            void far* upcall;
        } [maxentries];
    };

STRUCT_SIGNAL_TABLE  14
    struct signal_table {
        short maxentries;
        struct {
            unsigned short type;
            void far* signal_upcall;
        } [maxentries];
    };
```

