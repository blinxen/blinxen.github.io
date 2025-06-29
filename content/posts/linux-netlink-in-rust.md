---
title: Calling the Netlink subsystem in Linux using Rust
date: 2024-03-10
description: Introduction into how Linux Netlink could be used to retrieve network information using Rust.
---

*DISCLAIMER: This post is longer than intended.
Since I am still in the process of learning Netlink,
explaining it clearly presents some challenges.
Nevertheless, I hope the notes here prove useful to others beginning to work with Netlink.*

While building my own status line generator for i3 ([i3rustus](https://github.com/blinxen/i3rustus)),
I wanted to display network information for two specific interfaces: WLAN and LAN.
The first thing that came to my mind was [NetworkManager](https://www.networkmanager.dev/).
[It is the default service in Fedora for configuring anything network related in the Linux Kernel](https://fedoraproject.org/wiki/Tools/NetworkManager).
Luckily it provides a [DBus interface](https://networkmanager.dev/docs/api/latest/nm-settings-dbus.html)
to communicate with it and retrieve network information. This worked great at the
beginning but this solution relied on two external services, DBUS and NetworkManager.
This is not really a porblem per se but I wanted to explore if there are other
solutions out there that require 0 external services. While looking more into this,
I stumbled upon [Netlink](https://docs.kernel.org/userspace-api/netlink/intro.html)
which looked like it is exactly what I am looking for.
To clarify, in this context I don't consider the Linux Kernel an external service
because it is required to do basically anything.

## What is Netlink?

This question is not very easy to answer and during development I did not find
that many good explanations on this topic. All my knowledge about this subsystem
comes from:

* https://docs.kernel.org/userspace-api/netlink/intro.html
* https://www.yaroslavps.com/weblog/genl-intro/
* https://github.com/vishvananda/netlink

To summarize it, it is basically a replacement for `ioctl`. *What is `ioctl`?*
Well it is used to communicate with the Linux Kernel. *But wait, isn't that
what syscalls are used for?* Yes, and `ioctl` is a system call but sometimes
you want to talk to the kernel in a way that does not fit in regular file semantics.
*So Netlink is some kind of IPC protocol that is used to communicate with the Linux Kernel
from userland?* Yes again, that is a very simplified version of what Netlink is.

Before we go into our code, I should mention Generic Netlink here. You don't need
to know much about it except that it is a Netlink family and we will use it.

## How can I now use Netlink with Rust?

The Netlink subsystem uses UNIX sockets for communication. So we just need to create
a socket, write something into it and interpret the output.

### Creating a Netlink socket

To create a socket in Rust we will use the `libc` crate.
The Netlink subsystem has its own domain called `AF_NETLINK`.
The type of the socket will be `SOCK_RAW` since we are not using TCP or UDP.
The last argument that we need to specify is which Netlink family we want to talk to.
Obviously this depends on what we want to achieve. For now lets stick with
`NETLINK_GENERIC` and see what we can get from this family.

First we will create the socket:

```rust
use libc::{socket, AF_NETLINK, SOCK_RAW, NETLINK_GENERIC};

let generic_netlink_socket = unsafe { socket(AF_NETLINK, SOCK_RAW, NETLINK_GENERIC) };

if generic_netlink_socket < 0 {
    return Err(std::io::Error::last_os_error());
}
```

After creating the socket we can connect to it with:

```rust
use libc::{bind, connect, sockaddr, socklen_t};

if unsafe {
    bind(
        generic_netlink_socket,
        &socket_address as *const sockaddr,
        std::mem::size_of::<sockaddr>() as socklen_t,
    )
} < 0
{
    return Err(std::io::Error::last_os_error());
}

if unsafe {
    connect(
        generic_netlink_socket,
        &socket_address as *const sockaddr,
        std::mem::size_of::<sockaddr>() as socklen_t,
    )
} < 0
{
    return Err(std::io::Error::last_os_error());
}
```

### Talking to the Netlink socket

We have established a connection to the Netlink subsystem and can now talk to it.
But what do we want to talk about?
For now lets just get the name of the connected wireless device (SSID).
This information can be retrieved using the Netlink interface `nl80211`.
The `nl80211` interface provides a lot of commands which can be found
[here](https://github.com/torvalds/linux/blob/master/include/uapi/linux/nl80211.h#L347).
The one we are interessted in is the `NL80211_CMD_GET_SCAN` command.
This command returns all wireless devices that were found during the last scan.
Ok, now we know what we want to talk about and which interface to
use. Lets get the family ID of this interface and actually talk to Netlink!

*NOTE*: The family ID here is the Netlink family ID, not to be confused with the socket famil ID.

We will start by first introducing a couple of structures that help us build the
Netlink messages.

*NOTE*: If you did not read [the introduction into Generic Netlink](https://www.yaroslavps.com/weblog/genl-intro/)
then you should do that now to at least understand why the packets looks like this.

Each message starts with the message header.

```rust
// https://github.com/torvalds/linux/blob/master/include/uapi/linux/netlink.h#L52
pub struct NetlinkMessageHeader {
    pub length: u32,
    pub message_type: u16,
    pub flags: u16,
    pub sequence_number: u32,
    pub pid: u32,
    pub payload: Vec<u8>,
}
```

The fields should be pretty self-explanatory. The payload is not part of the header
but to make our live easier, we will just include it here. As we said before,
we will be using the Generic Netlink family.
The payload for Generic Netlink consists of a header and some attributes.

```rust
// https://github.com/torvalds/linux/blob/master/include/uapi/linux/genetlink.h#L13
pub struct GenericNetlinkMessageHeader {
    pub cmd: u8,
    pub version: u8,
    pub reserverd: u16,
    pub attributes: Vec<NetlinkAttribute>,
}
```

Again, the fields should be pretty self-explanatory and attributes are not part of the header
but we include them here to make our live easier.

```rust
// https://github.com/torvalds/linux/blob/master/tools/include/uapi/linux/netlink.h#L211
pub struct NetlinkAttribute {
    pub length: u16,
    pub attribute_type: u16,
    pub data: Vec<u8>,
}
```

To create a Netlink request we need to build a Netlink message and then write
it into the socket. Since the logic will always be the same for all requests,
we can create a helper method that handles this for us.

```rust
const MAX_NETLINK_MESSAGE_SIZE: usize = 32768;

fn request(
    socket: RawFd,
    netlink_message_type: i32,
    flags: i32,
    payload: Vec<u8>,
) -> Result<Vec<NetlinkMessageHeader>, IOError> {
    let mut result_buffer = Vec::new();
    // Create netlink header
    let input_buffer: Vec<u8> = NetlinkMessageHeader::build(netlink_message_type, flags, payload).serialize();
    // Send and receive answer from socket
    unsafe {
        send(
            socket,
            input_buffer.as_ptr() as *const c_void,
            input_buffer.len(),
            0,
        );
        loop {
            // Temporary buffer that will hold the current response
            let mut buffer = vec![0; MAX_NETLINK_MESSAGE_SIZE];
            let mut bytes_read: u32 = 0;
            let response_size = recv(
                socket,
                buffer.as_mut_ptr() as *mut c_void,
                MAX_NETLINK_MESSAGE_SIZE,
                0,
            );

            // Our input buffer is initialized with the maximum netlink message size
            // So we truncate the result to only contain the actual response bytes
            buffer.truncate(response_size as usize);
            // Create a vector which we can walk through
            // The idea here is that we don't want to manipulate the original response
            // vector, since this would require to allocate Vecs that are not needed
            let mut walkable_buffer = WalkingVec {
                buffer,
                position: 0,
            };
            // In a single response, there could be multiple netlink message headers
            loop {
                // Break out of loop if we have finished reading all bytes
                if bytes_read == response_size as u32 {
                    break;
                }
                let header: NetlinkMessageHeader = NetlinkMessageHeader::deserialize(&mut walkable_buffer);
                walkable_buffer.position += (header.length as usize) - NETLINK_HEADER_SIZE;
                bytes_read += header.length;
                result_buffer.push(header);
            }
            // Check if we are done with reading the response
            if let Some(last_header) = result_buffer.last() {
                // Error + error code 0 is the ACK message
                if last_header.message_type as i32 == NLMSG_DONE
                    || (last_header.message_type as i32 == NLMSG_ERROR)
                        && last_header.payload == Payload::Error(0)
                {
                    break;
                } else if last_header.message_type as i32 == NLMSG_ERROR
                    && last_header.payload != Payload::Error(0)
                {
                    log::error!(
                        "Error occured with netlink request of type {}",
                        netlink_message_type
                    );
                    result_buffer.clear();
                    break;
                }
            }
        }
    }

    if result_buffer.is_empty() {
        Err(IOError::new(
            ErrorKind::Other,
            "No netlink response could be found",
        ))
    } else {
        Ok(result_buffer)
    }
}
```

While it may look like a lot, the method simply writes some bytes into a socket,
waits for a response and then reads the response. A response can contain
multiple messages, so we return a `Vec` of `NetlinkMessageHeader`.
By using what I call a `WalkingVec`, we can parse the response efficiently without
having to modify or clone the original response. For anyone interessted, you
can find the implementation
[here](https://github.com/blinxen/i3rustus/blob/46056bb7125b4291545e86b6c2f09464381b65c0/src/utils/walking_vec.rs).

Soooo we created all this for what again? Ah right, we wanted to get the family ID
of the `nl80211` interface. With everything that we have setup now, the request
is now as simple as doing:

```rust
fn get_80211_family_id(socket: RawFd) -> Result<i32, IOError> {
    let mut family_id = i32::MIN;

    let genl_header = GenericNetlinkMessageHeader::build(
        CTRL_CMD_GETFAMILY,
        vec![NetlinkAttribute::build(
            CTRL_ATTR_FAMILY_NAME,
            WIRELESS_SUBSYSTEM_NAME.as_bytes().to_vec(),
        )],
    );
    let response = Self::request(
        socket,
        GENL_ID_CTRL,
        NLM_F_REQUEST | NLM_F_ACK,
        genl_header.serialize(),
    )?;

    let family_id_attribute = netlink_header::get_attribute(&message.attributes, CTRL_ATTR_FAMILY_ID);
    if let Some(family_id_attribute) = family_id_attribute {
        family_id =
            u16::from_le_bytes(family_id_attribute.data.clone().try_into().unwrap()) as i32;
    }

    if family_id == i32::MIN {
        Err(IOError::new(
            ErrorKind::Other,
            "Could not retrieve nl80211 family ID",
        ))
    } else {
        Ok(family_id)
    }
}
```

Now that we got the family ID of the Netlink interface, we can use it to ask
Netlink for information about the WLAN devices it knows about and extract the
information we need.

```rust
// Build generic netlink header
let genl_header = GenericNetlinkMessageHeader::build(
    NL80211_CMD_GET_SCAN,
    vec![NetlinkAttribute::build(
        NL80211_ATTR_IFINDEX,
        // this can be retrieved using the if_nametoindex syscall
        // interface refers to the network interface, in this case the WLAN
        // if you still don't know what I mean then just run "ip a" and you will
        // see the interface names of all connected networks
        interface_index.to_le_bytes().to_vec(),
    )],
);

// Ask netlink to give us the result of the last scan for the specified
// network interface
let response = Self::request(
    self.generic_netlink_socket,
    *nl_80211_family_id,
    NLM_F_REQUEST | NLM_F_DUMP | NLM_F_ACK,
    Payload::GenericNetlink(genl_header),
)?;

// Iterate over result
for message in response.iter() {
    if let Payload::GenericNetlink(message) = &message.payload {
        // Search for a BSS attribute since that will contain the SSID
        let bss_attribute =
            netlink_header::get_attribute(&message.attributes, NL80211_ATTR_BSS);
        if bss_attribute.is_none() {
            // We did not find a BSS attribute--> ignore this message
            continue;
        }
        // Parse the nested attributes in the BSS attribute
        let bss_attributes = netlink_header::parse_attributes(&mut WalkingVec {
            buffer: bss_attribute.unwrap().data.to_owned(),
            position: 0,
        });
        if bss_attributes.is_empty() {
            // No attributes found --> ignore this message
            continue;
        }

        // Check if we are currently connected to this BSS
        let bss_status =
            netlink_header::get_attribute(&bss_attributes, NL80211_BSS_STATUS);
        if let Some(bss_status) = bss_status {
            let status =
                u32::from_le_bytes(bss_status.data.clone().try_into().unwrap());
            if status != NL80211_BSS_STATUS_ASSOCIATED
                && status != NL80211_BSS_STATUS_IBSS_JOINED
            {
                // We are not connected to this BSS --> ignore this message
                continue;
            }
        } else {
            // No status could be found --> ignore this message
            continue;
        }

        // We are connected to this BSS, now begin extracting the SSID
        let bss_information_elements = netlink_header::get_attribute(
            &bss_attributes,
            NL80211_BSS_INFORMATION_ELEMENTS,
        );
        if let Some(bss_information_elements) = bss_information_elements {
            // Based on https://github.com/i3/i3status/blob/main/src/print_wireless_info.c#L141
            let mut ies = bss_information_elements.data.to_owned();
            while ies.len() > 2 && ies[0] != 0 {
                ies = ies[(ies[1] as usize + 2)..].to_owned();
            }

            if ies.len() < 2 || ies.len() < ies[1] as usize + 2 {
                break;
            };

            let ssid_len = ies[1] as usize;
            let ssid_bytes = &ies[2..][..ssid_len];

            bss.ssid = String::from_utf8_lossy(ssid_bytes).into_owned();
        }

        // The frequency can also be retrieved here
        let bss_freq =
            netlink_header::get_attribute(&bss_attributes, NL80211_BSS_FREQUENCY);
        if let Some(bss_freq) = bss_freq {
            // Frequency is in megahertz, but we want it in gigahertz
            bss.frequency =
                u32::from_le_bytes(bss_freq.data.clone().try_into().unwrap()) as f32
                    / 1000.0;
        }

        // We found the Access Point that we are connected to if ssid is not empty
        if !bss.ssid.is_empty() {
            break;
        }
    }
}
```

For those that made it this far, congrats! You now know how to actually talk to
Netlink in Rust and retrieve the SSID of the currently connected WLAN.
