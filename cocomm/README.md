Client socket interface to CANopenNode ASCII command interface
==============================================================

`cocomm` is a small command line program, which establishes socket connection with `canopend` (CANopen Linux commander device). It sends standardized CANopen commands (CiA309-3) to gateway and prints the responses to stdout and stderr. It is similar to command `nc -U /tmp/CO_command_socket`, but adjusted to CANopen.


Compile and install
-------------------

    cd cocomm
    make
    sudo make install

This will compile the `cocomm` utility and copy it to the `/usr/bin/` directory.


Example usage
-------------

    cocomm --help
    cocomm "help"
    cocomm "help datatype"
    cocomm "help lss"
    cocomm "1 read 0x1017 0 u16"
    cocomm "1 write 0x1017 0 u16 1000"
    cocomm "1 w 0x1010 1 vs save"
    cocomm "1 reset node"

Example will display usage help, read Heartbeat producer time from CANopen device with NodeId = 1, write 1000 ms to the same variable, store all non-volatile data on the device and reset the device. (Suppose CANopen device with NodeId = 1 is our CANopen Linux commander device. After 'reset node' command, our device will be stopped and cocomm won't work any more. Of course, cocomm can access any CANopen device, just by specifying it's node ID.)

Parameters to program can be set by program arguments, as described in `cocomm --help`, and can also be changed by environmental variables. For example, to change default socket path for all next `cocomm` commands in current terminal, type:

    export cocomm_socket="some other path than /tmp/CO_command_socket"

Commands can be also written into a file, for example create a `commands.txt` file, and for its content enter the commands:

    [1] r 0x1017 0 u16
    [2] 1 start

Then make `cocomm` use that file:

    $ cocomm -f commands.txt
    [1] 1000
    [2] OK

Program writes data to stdout and messages in green or red color to stderr.

For more examples see [CANopenDemo](https://github.com/CANopenNode/CANopenDemo).


Background about communication paths, when using cocomm
-------------------------------------------------------

1. `canopend` serves a socket connection on `/tmp/CO_command_socket` address. This is local Unix socket, TCP socket can be used also. `canopend` is pure CANopen device with commander functionalities and gateway. It listens for socket connections.
2. When run, `cocomm` tries to connect to `/tmp/CO_command_socket` address (this is default setting). `canopend` accepts the connection.
3. `cocomm` writes the specified ascii command to established socket connection, for example `[1] 4 read 0x1017 0 u16`.
4. Gateway in `canopend` receives the command and decodes it. `read` commands goes internally into `CO_SDOclientUploadInitiate()` and then command is processed with multiple `CO_SDOclientUpload()` function calls.
5. `CO_SDOclientUpload()` now sends a CAN message to targeted CANopen device. (CAN interface in Linux is implemented with CAN sockets. This is the third type of sockets mentioned here.) However, in our example targeted CANopen device receives SDO request, asking the value of the variable, located in Object Dictionary at index 0x1017, subindex 0.
6. Targeted CANopen device receives CAN message with CAN ID=0x604. It determines SDO request, so `CO_SDOserver_process()` function gets the message. Function gets the value from internal Object Dictionary and sends the CAN response with CAN ID=0x584. Those messages can be seen in candump terminal. And it is not necessary to understand the details of SDO communication, it may be quite complex.
7. `canopend` receives the CAN message, `CO_SDOclientUpload()` decodes it and sends binary value to the gateway.
8. Gateway in `canopend` translates binary value to asciiValue, unsigned16 in our example. It prepares the response, in our case `[1] ` + asciiValue + `\r\n`. Then writes the response text back to `/tmp/CO_command_socket`.
9. `cocomm` reads the response text from local socket and prints it partly to stderr (`[1] \r\n`) and partly to stdout (asciiValue).
10. If there are more commands, step 3 is repeated. Otherwise `cocomm` closes the socket connection and exits.
